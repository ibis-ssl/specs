# crane との連携

AI（crane、`ibis-ssl/crane`）とロボットの境界をまとめる。パケットの形式は [制御パケット](../common/control_packet.md)、[位置制御設定パケット](../common/config_packet.md)、[フィードバックパケット](../common/feedback_packet.md) にあり、ここでは crane 側の実装がそれをどう扱っているか、どこが食い違っているかを書く。

## 送信（crane_sender）

実装は `crane_sender/src/ibis_sender_node.cpp`、レイアウトは `crane_sender/include/crane_sender/robot_packet.h`（[正本](../common/control_packet.md#正本と一致検査)）。

- `HEADER` には `0x00` を入れる。
- `check_counter` は `RobotCommands` メッセージ 1 通ごとに進め、全ロボット共通の値を入れる（[CHECK_COUNTER](../common/control_packet.md#check_counter)）。
- ROS メッセージの `position_target_mode` が空でなければ wire の mode 4、それ以外は mode 3 で送る。mode 3 のときも `TARGET_GLOBAL_POS` に現在位置、`TERMINAL_VELOCITY` に 0 を入れる。
- `elapsed_time_ms_since_last_vision` は `nanoseconds() / 1e6` を 0〜65535 に clamp してから `uint16_t` にする。world model からロボットを引けない例外時は 65535 を入れる（fail-closed）。
- `latency_time_ms` も 65535 で飽和させる。
- 位置制御設定パケット（28 B、version 2）を最大 1 秒に 1 回、ポート 12350 へ送る。float は memcpy でホストのバイト順（x86 では little endian）のまま入れる。
- 現状の送信先は 715 B のブロードキャスト（[既知の不整合](#既知の不整合)）。`sim:=true` のときは `127.0.0.1` へ送る。

## 受信（crane_robot_receiver）

実装は `crane_robot_receiver/src/robot_receiver_node.cpp`、レイアウトは `crane_robot_receiver/include/crane_robot_receiver/robot_feedback_protocol.hpp`。

- 各機体の feedback を multicast `224.5.20.(100 + N):(50100 + N)` で受ける。グループとポートはパラメータで、ロボット ID もパラメータで明示的に渡す（ポート番号から逆算しない）。
- 実機でもシミュレーションでも常に multicast で受ける。ソケットは `SO_REUSEADDR` と `SO_REUSEPORT` を付けて `0.0.0.0:port` に bind する。
- byte 2 のチェックサムは検査しない（定義名は dummy）。
- byte 4..7 の yaw は度として扱う。byte 44..51 は `ODOM_X/Y` と呼んでいるが、G474 の `vision_based_position_x/y` と同じフィールドである。byte 64..119 は 14 個のデバッグ用 float で、先頭（64..79）を `MOUSE_ODOM` と `MOUSE_VEL` として読む。

## 既知の不整合

照合の基準コミット（[README](../README.md#照合の基準)）の時点で、次が食い違っている。

### 制御指令の転送方式

| | 方式 |
|---|---|
| 仕様（Orion_CM4 の実装） | 機体ごとに 65 B を `192.168.20.(100 + N):12345` へユニキャスト。設定パケットも機体ごとにユニキャスト |
| crane の送信器（develop 最新を含む） | 11 台分（各 `[robot_id][64 B]`、指令のない機体はゼロ埋め）を連結した 715 B を `192.168.20.255:12345` へブロードキャスト。設定パケットも `192.168.20.255:12350` へブロードキャスト |

- 現行の CM4（`ai_cmd_v2.out`）は 65 B 以外のデータグラムを捨てるので、**この組み合わせでは実機に指令が届かない。** crane の送信器を 65 B ユニキャストに切り替える必要がある。
- シミュレーションが今動いているのは、crane の compose が `cm4_sim` のイメージを `CM4_SIM_TAG=d7a2e07`（2026-09-15、Orion_CM4 がユニキャストに切り替えた `baa0442` より前）に固定しているからにすぎない。現行の `cm4_sim.out` も入力は 65 B なので、**crane 側を切り替えずにタグを上げると、シミュレーションも壊れる。**
- crane の `docs/network.md` と `docs/cm4_position_control.md` もブロードキャストと書いている。

### crane 側の文書の記述

- `docs/cm4_position_control.md` は CM4 の位置制御を「1000 Hz の PID」と説明している。実機の `ai_cmd_v2.out` は 1 kHz でポーリングし、制御指令は `--tx-rate-hz`（既定 100 Hz）で送る。1000 Hz なのは `cm4_sim` の `--rate-hz` の既定値である。
- crane の compose のコメントには「`crane_robot_receiver` は 50100 を直書きしている」とあるが、現在のポートはパラメータである（既定値は 50100+N のまま）。

### ホスト側デコーダ

- `host/lib/feedback/packet.py` の `ball_detection` が byte 14 まで読む（[フィードバックパケット](../common/feedback_packet.md#既知の不整合host-側デコーダ)）。
