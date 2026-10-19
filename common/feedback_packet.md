# フィードバックパケット

Main（G474）が CM4 へ送る 128 バイトの状態パケット（テレメトリ）の形式と、それを受け取る側をまとめる。

この文書は Orion の `G474_Orion_main` で確認したフィードバックの仕様である。CM4 から送るコマンドの定義は Orion と Atlas で共通だが、ここに示す送信周期、パケットの内容、各センサやモータとの対応は Orion の記録で、Atlas のフィードバックは未記載である。

## 送信元と経路

送信元は G474 の `Core/Src/ai_comm.c` にある `sendRobotInfo()`。USART2（1 Mbps）から DMA で送る。

- 送信周期は 125 Hz（`ROBOT_INFO_CYCLE = 125`）。500 Hz の制御周期の中で間引いて送る。
- デバッグ UART で `t` を送ると送信を一時停止・再開でき、FW ゲートウェイの動作中は送信しない（[Orion CM4 との UART 実装](../stm/orion_uart_cm4.md)）。

```text
G474 sendRobotInfo()
  -> UART /dev/serial0
  -> CM4 robot_feedback.out（cm4/bridge/forward_robot_feedback.cpp）
       |-> UDP multicast 224.5.20.(100+N):(50100+N) -> crane_robot_receiver / host の feedback ツール
       `-> UDP unicast 127.0.0.1:(50100+N)            -> ai_cmd_v2.out（位置制御ループ）
```

CM4 側の転送処理の詳細は [ブリッジ](../cm4/bridge.md#forward_robot_feedbackcpp) を参照。

## パケット仕様

長さは 128 バイト固定。float はすべて STM32 の `float_to_uchar4()` の生バイト列（little-endian IEEE754）で送る。

### ヘッダ

| byte | 内容 |
|---|---|
| `0` | 同期バイト `0xAB` |
| `1` | 同期バイト `0xEA` |
| `2` | チェックサム：byte 3..127 の総和 `& 0xFF` |
| `3` | `check_counter`：受け取った指令の `check_counter` をそのまま返す |

- byte 2 のチェックサムは、送信直前に byte 3..127 から計算して書き込む。ソース中で最初に `buf[2] = 10;  // CRC, 10:dummy` と書いている行は、最後に上書きされる。
- byte 3 は指令の `check_counter` を反射したもの。mode 4 の位置制御経路では CM4 が `check_counter` を採番するので、ここを見れば「CM4 が出した指令がどこまで G474 に届いたか」が分かる（[制御パケット](control_packet.md#check_counter)）。シミュレータではこの対応は成り立たない（[シミュレーション](../pc/simulation.md#feedback-の違い)）。

### ペイロード

| byte | 名前 | 型 | 内容 |
|---|---|---|---|
| `4..7` | `imu_yaw_deg` | float | IMU の yaw [deg] |
| `8..11` | `battery_voltage` | float | `power_voltage[0]`（CAN `0x210`、右 BLDC 基板で測ったバッテリ電圧） |
| `12..13` | `ball_detection[2]` | uint8 ×2 | ボール検出（CAN `0x240`） |
| `14` | `tx_cycle_count` | uint8 | feedback を送るたびに 1 増えるカウンタ |
| `15` | `kick_state_div10` | uint8 | `kick_state / 10` |
| `16..17` | `current_error_id` | uint16 LE | 現在のエラー ID |
| `18..19` | `current_error_info` | uint16 LE | エラー info |
| `20..23` | `current_error_value` | float | エラー値 |
| `24..27` | `motor_current_x10[4]` | uint8 ×4 | 各モータの電流 ×10（CAN `0x230`〜`0x233`） |
| `28` | — | uint8 | 未使用（常に 0） |
| `29..32` | `temp_motor[4]` | uint8 ×4 | モータ温度 |
| `33` | `temp_fet` | uint8 | 電源系の FET 温度 |
| `34..35` | `temp_coil[2]` | uint8 ×2 | コイル温度 |
| `36..39` | `diff_angle_deg` | float | `imu_yaw_deg - deg(vision_global_theta)` |
| `40..43` | `capacitor_boost_voltage` | float | `power_voltage[6]`（CAN `0x216`、キャパシタ電圧） |
| `44..47` | `vision_based_position_x` | float | ビジョンとオドメトリを統合した推定位置 X [m] |
| `48..51` | `vision_based_position_y` | float | 同 Y [m] |
| `52..55` | `global_odom_speed_x` | float | オドメトリ速度（グローバル）X [m/s] |
| `56..59` | `global_odom_speed_y` | float | 同 Y [m/s] |
| `60` | `camera_pos_x_div2` | uint8 | ローカルカメラ x / 2 |
| `61` | `camera_pos_y` | uint8 | ローカルカメラ y |
| `62` | `camera_radius_div4` | uint8 | ローカルカメラ radius / 4 |
| `63` | `camera_fps` | uint8 | ローカルカメラ fps |
| `64..119` | `tx_value_array[14]` | float ×14 | [下表](#tx_value_array) |
| `120..127` | — | — | 未使用（0） |

CM4 は byte 44..51 を自機の現在位置として位置制御に使う。crane はこのフィールドを `ODOM_X/Y` と呼んでいるが、同じものである。

### カメラ値の復元

送るバイト数を減らすため、カメラ値は圧縮されている。

- `camera_pos_x = camera_pos_x_div2 * 2`
- `camera_radius = camera_radius_div4 * 4`
- `camera_pos_y` と `camera_fps` はそのまま使う。

カメラ値は、CM4 の `cam_server_v3` が検出して UART フレームで G474 へ送ったものが返ってくる（[ブリッジ](../cm4/bridge.md#ローカルカメラ情報の挿入)）。

### tx_value_array

`sendRobotInfo()` が `enqueueFloatArray()` に追加した順に、`buf[64..119]` へ格納される。

| index | 内容 |
|---|---|
| `0` | `mouse_odom_x` |
| `1` | `mouse_odom_y` |
| `2` | `mouse_global_vel_x` |
| `3` | `mouse_global_vel_y` |
| `4` | `output_vel_x` |
| `5` | `output_vel_y` |
| `6`〜`9` | `motor_feedback_0`〜`3` |
| `10` | `local_odom_speed_mvf_x` |
| `11` | `local_odom_speed_mvf_y` |
| `12` | `local_odom_speed_mvf_w` |
| `13` | `mouse_quality` |

## 受信側

| 受信側 | 場所 | byte 2 の扱い |
|---|---|---|
| CM4 位置制御 | `ai_cmd_v2.out` / `cm4_sim.out` | 見ない。byte 44..51 だけを使う |
| crane | `crane_robot_receiver`（`robot_feedback_protocol.hpp`） | 検査しない（定義名は dummy） |
| ホストツール | `host/lib/feedback/packet.py` | `is_checksum_valid()` が byte 3..127 の総和と比べる。実機のパケットなら一致する |

ホスト側ツールの使い方は [ホスト PC 側ツール](../pc/host_tools.md#robot-feedback-ツール) を参照。

### 既知の不整合（host 側デコーダ）

- `host/lib/feedback/packet.py` は `ball_detection=(data[12], data[13], data[14])` と 3 バイト読んでいる。`data[14]` は `tx_cycle_count` で、ボール検出ではない。
