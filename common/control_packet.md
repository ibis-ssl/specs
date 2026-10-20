# 制御パケット

AI（crane）から CM4 を経由して Main へ送る指令の形式と、各フィールドの意味をまとめる。コマンドの定義は Orion（4 輪オムニホイール）と Atlas（4 輪ステアリング）で共通である。

経路は `crane →(Wi-Fi UDP 65 B)→ CM4 →(機種別の物理通信)→ Main`。中身は 64 バイトの `RobotCommandSerializedV2` である。CM4 から送るパケットの配置は共通とし、物理通信方式と採用状況は [システム構成](system_overview.md#cm4-と-main-の通信)を参照。

以下の G474 におけるフィールドの利用、受信処理、接続判定、フィードバックの説明は、`G474_Orion_main` で確認した Orion の実装を指す。UART の物理設定と `FWVR` / `FWUP` の受信処理も Orion の記録で、Atlas の STM 側の実装詳細は未記載である。

## 正本と一致検査

`RobotCommandSerializedV2` のレイアウトの正本は crane の `crane_sender/include/crane_sender/robot_packet.h`。ほかのリポジトリは次のように対応している。レイアウトは影響範囲が広いので、互換を保ち、できるだけ変えない。

| リポジトリ | ファイル | 状態 |
|---|---|---|
| crane | `crane_sender/include/crane_sender/robot_packet.h` | **正本**（byte 0..37 を定義） |
| Orion_CM4 | `cm4/bridge/robot_packet.h` | byte 0..37 が一致 |
| framework | `src/simulator/ibis_protocol.h` | 一致 |
| G474_Orion_main | `Core/Inc/robot_packet.h` | byte 0..27 だけを定義（mode 3 のみ）。28..63 は使わない |

`cm4/bridge/robot_packet_layout_test.cpp` は、byte 0..37 のすべてのオフセット、`ControlMode`、`FlagAddress` を `static_assert` で固定する。さらにゴールデンベクタで量子化の挙動（丸めずに切り捨てる）まで検査する。`cm4/build.sh` と CI から実行されるので、`robot_packet.h` を編集したら必ず通すこと。

## 機体別の制御パケット（65 バイト）

送信側は機体ごとに 1 データグラム 65 バイトを、その機体の CM4 の `192.168.20.(100 + N):12345` へユニキャストで送る（`N` は [機体番号](system_overview.md#機体番号とネットワーク)）。

```text
byte 0    : robot_id（0..10）
byte 1..64: RobotCommandSerializedV2（64 バイト）
```

- 複数の機体へ送るときは、各機体の IP へ個別にデータグラムを送る。
- 受信側（`ai_cmd_v2.out`）は次のパケットを採用しない：長さが 65 バイトでないもの、`robot_id` が自機と違うもの、64 バイトのコマンドがすべてゼロのもの。
- 使用中のコマンドでも byte 28..31 と 38..63 はゼロとは限らない。受信側は予約領域がゼロであることを前提にしない。

GUI_Qt の送信実装（Qt Communication Tester）もこの 65 バイト形式で mode 3 を送る。mode 4 の位置指令を送る側は、同じ外枠で下記の mode 4 フィールドを設定する。

> [!WARNING]
> crane の送信器はまだこの形式に対応していない。11 台分 715 B を 1 つのデータグラムにまとめてブロードキャストしている。[crane との連携](../pc/crane_integration.md#既知の不整合) を参照。

## RobotCommandSerializedV2（64 バイト）

64 バイト固定長。使っているのは byte 0..37 で、38..63 は未使用。

| offset | 名前 | 符号化 | 意味 |
|---|---|---|---|
| `0` | `HEADER` | 生値 | crane は `0x00` を入れる。CM4 が Main へ送る直前に `0xFE` で上書きする |
| `1` | `CHECK_COUNTER` | 生値 | [後述](#check_counter) |
| `2..3` | `VISION_GLOBAL_X` | float, range `32.767` | 外部グローバルカメラ座標系での機体の X [m] |
| `4..5` | `VISION_GLOBAL_Y` | float, range `32.767` | 同 Y [m] |
| `6..7` | `VISION_GLOBAL_THETA` | float, range `M_PI` | 同 現在角度 [rad] |
| `8..9` | `TARGET_GLOBAL_THETA` | float, range `M_PI` | グローバル座標系での目標角度 [rad] |
| `10` | `KICK_POWER` | `uint8 = value * 20` | キッカー出力 |
| `11` | `DRIBBLE_POWER` | `uint8 = value * 20` | ドリブラー出力 |
| `12..13` | `ACCELERATION_LIMIT` | float, range `32.767` | 並進加速度の上限 [m/s²] |
| `14..15` | `LINEAR_VELOCITY_LIMIT` | float, range `32.767` | 並進速度の上限 [m/s] |
| `16..17` | `ANGULAR_VELOCITY_LIMIT` | float, range `32.767` | 角速度の上限 [rad/s] |
| `18..19` | `LATENCY_TIME_MS` | uint16 生値（上位バイトが先） | 外部カメラ情報の遅延 [ms] |
| `20..21` | `ELAPSED_TIME_MS_SINCE_LAST_VISION` | uint16 生値（上位バイトが先） | この機体を最後にビジョンで捉えてからの経過時間 [ms] |
| `22` | `FLAGS` | ビット | [後述](#flags) |
| `23` | `CONTROL_MODE` | 生値 | `3` または `4`。[後述](#制御モード) |
| `24..31` | `CONTROL_MODE_ARGS` | union | `CONTROL_MODE` によって意味が変わる |
| `32..33` | `TARGET_GLOBAL_POS_X` | float, range `32.767` | mode 4 の目標位置 X [m] |
| `34..35` | `TARGET_GLOBAL_POS_Y` | float, range `32.767` | mode 4 の目標位置 Y [m] |
| `36..37` | `TERMINAL_VELOCITY` | float, range `32.767` | mode 4 の到達時の速度上限（スカラー）[m/s] |

crane は mode 3 で送るときも `TARGET_GLOBAL_POS` に現在位置を、`TERMINAL_VELOCITY` に 0 を入れる。

### FLAGS

| bit | 名前 | 意味 |
|---|---|---|
| `0` | `IS_VISION_AVAILABLE` | 外部カメラでこの機体が見えているか |
| `1` | `ENABLE_CHIP` | 1 ならチップキック、0 ならストレートキック |
| `2` | — | 未使用（常に 0） |
| `3` | `STOP_EMERGENCY` | 停止指令。パケット内のほかのどの指令よりも優先して走行出力を止める |
| `4..7` | — | 未使用（常に 0） |

### スケーリング

- 「float, range `R`」のフィールドは `convertFloatToTwoByte(value, R)` で 2 バイトにする。計算は `(uint16_t)(32767.f * (value / R) + 32767.f)` で、上位バイトが先。
  - range `32.767` の量子化幅は約 1 mm（1 mm/s）、range `M_PI` は約 0.0001 rad。
  - **丸めずに切り捨てる。** `roundf()` を足すと crane と 1 LSB ずれ、レイアウト検査テストで検出される。
  - 範囲外の値は ±range にクランプする。crane は `std::cout` に警告を出す。CM4 は制御ループで毎周期呼ぶので警告は出さず、クランプ回数を `robotPacketClampCount()` で数えてデバッグ表示にまとめて出す。
- `kick_power` と `dribble_power` は `value * 20` を 1 バイトに入れる。
- `latency_time_ms` と `elapsed_time_ms_since_last_vision` は素の `uint16_t`。crane は 65535 で飽和させてから入れる。

### ゼロ埋めはゼロ値にならない

range 付きの 2 バイトのフィールドをゼロ埋めすると、`0.0` ではなく **`-32.767`**（range `M_PI` なら `-π`）として復号される。`0.0` を符号化すると `0x7FFF` になるためである。

- `bytearray(64)` のままのテストパケットは、すべてのフィールドが `-32.767` になる。使わないフィールドも `0.0` を明示的に符号化して埋めること。
- byte 18..21 は素の `uint16` なので、ゼロ埋めすると正しく `0` になる。同じパケットに 2 種類の符号化が混在している点に注意する。
- テストでは FLAGS の bit0 を立てること。立て忘れるとロボットは動かない。
- CM4 の `position_controller` は `|v| >= 32` と NaN を「未設定」として扱う（[位置制御](../cm4/position_control.md#未設定値の扱い)）。

## 各フィールドの使われ方

ここでは共通のフィールドに対する CM4 の処理と、Orion の G474 での利用方法を示す。Atlas 側の利用方法は機種別仕様で扱う。

- **`VISION_GLOBAL_*`**：遅延が大きく不安定なので、参照するときは注意する。旋回中は特に角度が当てにならない。G474 はこの値をオドメトリと融合して位置を推定する。CM4 は実機ではこの値を書き換えずに素通しする（CM4 の推定値を書き戻すと自己帰還になるため）。
- **`IS_VISION_AVAILABLE`**：見えていない状態が続いたら、暴走を防ぐためにロボット側でも走行出力を止める。一瞬見失うだけなら、ロボット側のオドメトリを使って安全を保ちつつ動ける。停止条件は [停止判定](safety_stop.md#g474-の走行停止条件) を参照。
- **`ACCELERATION_LIMIT`**：機体に期待する並進加速度の上限。機体側がこれより高い加速度を使うことは許される。G474 は 3.0 未満か 20 を超える値を受けると 3.0 を使う。
- **`LINEAR_VELOCITY_LIMIT`**：G474 は使っていない。CM4 の位置制御器は出力速度の上限として使う（[次節](#linear_velocity_limit--0-の扱い)）。
- **`ANGULAR_VELOCITY_LIMIT`**：G474 は角度制御が作る目標角速度をこの値で制限する。ボールを保持していてゆっくり回りたいときに下げたり、不要に旋回させたくないときに 0 にしたりする。
- **`LATENCY_TIME_MS` / `ELAPSED_TIME_MS_SINCE_LAST_VISION`**：G474 は、受け取った座標から現在位置を推定する補償時間を `latency_time_ms + elapsed_time_ms_since_last_vision + ai_cmd_delta_time` として計算する（`odom.c` の `inntegOdomUpdate()`）。最後の項は、`check_counter` の変化を G474 が最後に検出してからの経過時間。前の 2 項は足し合わされるので、同じ遅延を二重に数えないよう、AI 側で基準時刻を揃える必要がある。

### `LINEAR_VELOCITY_LIMIT = 0` の扱い

このフィールドの `0` は、上流と下流で意味が違う。

- crane の位置制御則と CM4 の `position_controller` は、`clampNorm` の仕様上 `0` を「停止」として扱う。
- framework の `ibis_protocol.h` は `0 means "no limit"` と定義している。simulator-cli は値が `> 0` のときだけ制限を適用する。

CM4 では上流（位置制御器）の解釈が先に効く。`linear_velocity_limit = 0` なら位置制御器が `r = 0` を出すので、このバイトを下流へ素通ししても結果は変わらない。これは意図した決定で、`position_controller` の単体テストで固定している。

## 制御モード

| 値 | 名前 | ARGS（24..31）の意味 | 送信元 → 受信先 |
|---|---|---|---|
| `3` | `POLAR_VELOCITY_TARGET_MODE` | `target_global_velocity_r`, `target_global_velocity_theta` | CM4 → Main / cm4_sim → simulator-cli / GUI_Qt → CM4 |
| `4` | `POSITION_TARGET_WITH_TERMINAL_VELOCITY_MODE` | `terminal_velocity_x`, `terminal_velocity_y` | crane → CM4 / crane → cm4_sim |

> **`CONTROL_MODE_ARGS` は union である。`CONTROL_MODE` を見ずに復号してはいけない。** mode 4 のパケットを mode 3 として復号すると、`terminal_velocity_x/y` が `r/theta` として読まれ、警告もなく暴走する。

CM4 は mode 4 を受け取って mode 3 に変換し、Main へ送る。Orion の G474 が実装しているのは mode 3 だけである。mode 4 を G474 へ素通ししてはいけない。

crane の ROS メッセージ（`crane_msgs/msg/RobotCommand.msg`）の制御モード番号は、ここに書いた値とは別物である。

### POLAR_VELOCITY_TARGET_MODE（3）

グローバル座標系の速度を、大きさと方向で指定する。

- `24..25`：`target_global_velocity_r`（range 32.767）。目標速度の大きさ [m/s]
- `26..27`：`target_global_velocity_theta`（range 32.767）。グローバル座標系での移動方向 [rad]
- `28..31`：未使用

### POSITION_TARGET_WITH_TERMINAL_VELOCITY_MODE（4）

- `24..25`：`terminal_velocity_x`（range 32.767）
- `26..27`：`terminal_velocity_y`（range 32.767）
- `28..31`：未使用

目標位置は mode_args ではなく固定フィールドの `TARGET_GLOBAL_POS_X/Y`（32..35）に入る。到達時の速度上限は `TERMINAL_VELOCITY`（36..37）に入る。

## CM4→Main パケット（72 バイト）

以下のパケット配置は Orion と Atlas で共通である。物理通信の速度・転送手順は機種別に扱い、Orion の接続設定は [基板の対外仕様](../stm/orion_board_spec.md#cm4-との通信)を参照。

| byte | 内容 |
|---|---|
| `0` | ヘッダ `0xFE`（コマンドの `HEADER` を上書きしたもの） |
| `1..63` | `RobotCommandSerializedV2` の byte 1..63 |
| `64..65` | ローカルカメラ x（big-endian） |
| `66..67` | ローカルカメラ y（big-endian） |
| `68..69` | ローカルカメラ radius（big-endian） |
| `70` | ローカルカメラ fps |
| `71` | チェックサム：byte 0..70 の総和 `& 0xFF` |

- Orion の UART の送信時間と占有率は [UART 占有率](../stm/orion_board_spec.md#uart-占有率)を参照。
- カメラ値の出どころと、途絶したときの扱いは [ブリッジ](../cm4/bridge.md#ローカルカメラ情報の挿入) を参照。
- Orion では、byte 1..4 が `FWVR` または `FWUP` のフレームは、制御指令ではなくファームウェアのバージョン照会・更新要求として扱われる（[MCU ファームウェア更新](firmware_update.md)）。

## CHECK_COUNTER

- crane は `RobotCommands` メッセージ 1 通ごとに 1 回インクリメントし、その値を全ロボット共通で入れる。200 を超えると 0 に戻る（1..200、0 を巡回）。world model が更新されていない周期は進めない。
- G474 は **`check_counter` が変化し続けていること**で AI との接続が生きていると判定する（`ai_comm.c` の `checkConnect2AI()`）。タイムアウトの値は [停止判定](safety_stop.md#g474-の通信タイムアウト) を参照。
- 1 バイトなので値は周期的に一巡する。**ロスを検出するためのシーケンス番号としては使えない。**
- G474 は受け取った値をテレメトリの byte 3 でそのまま返す（[フィードバックパケット](feedback_packet.md#ヘッダ)）。

### 経路ごとの採番

| CM4 の経路 | `check_counter` | 送信のきっかけ |
|---|---|---|
| 素通し（mode ≠ 4、または `--passthrough`） | crane から届いた値をそのまま流す | crane の値が変化したとき |
| 位置制御（mode 4） | **CM4 が採番する**（送信ごとに 0..200 を巡回） | `--tx-rate-hz` の時間ゲート |

位置制御の経路では、CM4 が生きていれば crane が落ちていても `check_counter` は変化し続ける。したがって G474 の `connected_ai` は crane が生きていることを意味しない。crane が落ちたときの停止は CM4 側で行う（[停止判定](safety_stop.md)）。
