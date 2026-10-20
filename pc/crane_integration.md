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
- 指令は機体ごとに 65 B（`[robot_id][64 B]`）を 1 データグラムずつ送る。指令のない機体には送らない。ブロードキャストはしない。
- 宛先はパラメータ `unicast_to_robot_ip` で切り替える。`sim:=false`（`true`）では機体 N の CM4 `192.168.20.(100 + N):12345` へ、`sim:=true`（`false`）では全機体を `target_address`（`127.0.0.1:12345`、`cm4_sim`）へ送る。
- 送信は機体ごとに独立していて、1 台へ送れなくても他の機体への送信は続く。送れたかどうかは `/sent_robot_commands` の planning factor `SenderSent` に機体ごとに残る。
- 位置制御設定パケットは、直近に指令を送った機体それぞれのポート 12350 へ送る。byte 5 は `0xFF`。シミュレーションでは宛先がすべて同じ `cm4_sim` なので 1 回だけ送る。

## 受信（crane_robot_receiver）

実装は `crane_robot_receiver/src/robot_receiver_node.cpp`、レイアウトは `crane_robot_receiver/include/crane_robot_receiver/robot_feedback_protocol.hpp`。

- 各機体の feedback を multicast `224.5.20.(100 + N):(50100 + N)` で受ける。グループとポートはパラメータで、ロボット ID もパラメータで明示的に渡す（ポート番号から逆算しない）。
- 実機でもシミュレーションでも常に multicast で受ける。ソケットは `SO_REUSEADDR` と `SO_REUSEPORT` を付けて `0.0.0.0:port` に bind する。
- byte 2 のチェックサムは検査しない（定義名は dummy）。
- byte 4..7 の yaw は度として扱う。byte 44..51 は `ODOM_X/Y` と呼んでいるが、G474 の `vision_based_position_x/y` と同じフィールドである。byte 64..119 は 14 個のデバッグ用 float で、先頭（64..79）を `MOUSE_ODOM` と `MOUSE_VEL` として読む。

## 既知の不整合

照合の基準コミット（[README](../README.md#照合の基準)）の時点で、次が食い違っている。

### 715 B を前提にした受信側

crane の送信は 65 B のユニキャストに切り替えた（`cm4_sim` のイメージも `CM4_SIM_TAG=e03b0f6` に更新済み）。次の受信側は 715 B のままなので、crane から直接はつながらない。

- framework の `simulator-cli` は 715 B しか受けない。crane は必ず `cm4_sim` を挟んでつなぐ（match-vs-tigers の compose にも `cm4_sim` を入れた）。
- grSim の `IbisCommandReceiver` も 715 B を前提にしている。

### ホスト側デコーダ

- `host/lib/feedback/packet.py` の `ball_detection` が byte 14 まで読む（[フィードバックパケット](../common/feedback_packet.md#既知の不整合host-側デコーダ)）。
