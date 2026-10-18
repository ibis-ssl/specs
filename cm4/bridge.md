# ブリッジ

CM4 上で、Wi-Fi の UDP と G474 の UART をつなぐ 2 つのプロセスをまとめる。パケットの形式は [制御パケット](../common/control_packet.md) と [フィードバックパケット](../common/feedback_packet.md)、ポートは [システム構成](../common/system_overview.md#機体番号とネットワーク) を参照。

```text
crane --UDP:12345 65B--> ai_cmd_v2.out --UART 72B--> G474
                          ^       ^
       127.0.0.1:8890 7B  |       | 127.0.0.1:(50100+N) 128B
          cam_server_v3 --+       +-- robot_feedback.out <--UART 128B-- G474
                                          |
                                          `--> 224.5.20.(100+N):(50100+N) multicast
```

## forward_ai_cmd_v2.cpp

`ai_cmd_v2.out` として動く。AI からの指令とローカルカメラの情報を受け、UART で G474 へ送る。mode 4 を受けたときは位置制御ループを閉じて mode 3 に変換する。

### 入力

| 入力 | ポート | 内容 |
|---|---|---|
| AI 制御パケット | UDP `12345`（`--ai-cmd-port`） | 65 バイト固定。`recv()` に `MSG_TRUNC` を付けて実際の長さを調べ、65 バイト以外は捨てる。先頭の ID が自機と違うもの、コマンドがすべてゼロのものも採用しない |
| ローカルカメラ | UDP `8890`（`--local-cam-port`） | 7 バイト（`CAM_BUF_SIZE`） |
| 位置制御設定 | UDP `12350`（`--config-port`） | 28 バイト固定（[位置制御設定パケット](../common/config_packet.md)） |
| G474 feedback | `127.0.0.1:(50100 + N)`（`--feedback-port`） | `robot_feedback.out` が UART から読んだ 128 バイト。位置制御ループを閉じるのに使う |

ロボット ID `N` は `wlan0` の IPv4 末尾オクテットから 100 を引いて求める。求められないとき、または 0〜10 の範囲外のときは**動かずに終了する**。位置制御では ID が feedback の bind ポートも決めるので、黙って 0 に落ちると「自分の G474 へ送りながら 0 号機の feedback で位置ループを閉じる」機体をまたいだ制御になる。テストでは `--robot-id` で明示できる。

### UART 送信

- ポート：`/dev/serial0`。`--serial-port` で変えられる。
- baudrate：既定 `1000000`。`-s` で変えられる。
- 72 バイトのフレームを組み立てて送る（[UART フレーム](../common/control_packet.md#uart-フレームcm4--g47472-バイト)）。先頭を `0xFE` に上書きし、末尾にチェックサムを付ける。

### 2 つの経路

受けたパケットの `CONTROL_MODE` で経路が分かれる。**この 2 つは意図的に統合していない。**

| 条件 | 動作 | 送信のきっかけ |
|---|---|---|
| mode ≠ 4、または `--passthrough` 指定 | 64 バイトをそのまま転送する | crane の `CHECK_COUNTER` が変化したとき |
| mode 4 | 位置制御ループを閉じて mode 3 を作る | `--tx-rate-hz`（既定 100 Hz）の時間ゲート。停止理由が変わった周期はすぐ送る |

`check_counter` の扱いは [制御パケット](../common/control_packet.md#経路ごとの採番) を参照。

`--passthrough` は mode 4 が来ても強制的に素通しする。G474 は mode 4 を処理しないので、mode 4 を診断するときは `--debug` と併用し、実機の UART へは送らない。

### 位置制御経路の出力

- 受けた 64 バイトをコピーし、`CHECK_COUNTER` / `CONTROL_MODE` / `CONTROL_MODE_ARGS` だけを差し替えて作る。ゼロから組み立てると、`target_global_theta` / `angular_velocity_limit` / `kick_power` / `dribble_power` / flags を取りこぼす（G474 も simulator-cli もこれらをすべて使う）。
- 安全停止のときは速度を 0 にして `STOP_EMERGENCY` を立て、キック・ドリブル・チップを落とす（[停止判定](../common/safety_stop.md#cm4-の安全停止mode-4-の位置制御経路)）。
- `VISION_GLOBAL_X/Y`（2..5）は crane から来た値のまま流す。G474 がビジョン融合に使うので、CM4 の推定値を書き戻すと自己帰還になる。
- 制御則は [位置制御](position_control.md) を参照。

### 送信レートとポーリング

- メインループは `usleep(1000)` の 1 kHz ポーリング。UDP は非ブロッキングで読み、受信がなければ前回のバッファがそのまま残る。
- 位置制御経路の UART 送信レートは `--tx-rate-hz`、既定 100 Hz（UART 占有率 7.2%）。crane からの指令が途絶えても停止指令を送れるよう、時間ゲートで送信する。
- `--tx-rate-hz 500` では占有率が約 36% になる。使う前に G474 の `ORE`/`FE`/`NE`/`PE` カウンタを確認する。

### ローカルカメラ情報の挿入

`cm4/camera/cam_server_v3.py` は検出したカメラ情報を `127.0.0.1:8890` へ 7 バイトで送る（[カメラ](camera.md#stm32-feedback-用ローカルカメラ-udp)）。`forward_ai_cmd_v2.cpp` はこれを UART フレームの byte 64..70 に入れる。

- `0..1`：x、`2..3`：y、`4..5`：radius（いずれも big-endian）、`6`：fps
- カメラの更新レートは STM32 への送信周期より低いので、最後に受けたカメラ情報を短時間保持して使う。
- 一定時間更新がないとき、またはカメラがつながっていないときは、カメラ領域を 0 で埋める（`x=0, y=0, radius=0, fps=0`）。

G474 はこの値を feedback の byte 60..63 に入れて返す。

## forward_robot_feedback.cpp

`robot_feedback.out` として動く。G474 から UART で受けた 128 バイトのパケットを転送する。`-n` で渡した値 `n`（`cm4/lancher.py` 経由では IP の末尾オクテット `100 + N`、既定 100）から宛先を決める。

| 宛先 | 用途 |
|---|---|
| UDP multicast `224.5.20.n:(50000 + n)` | crane とホストツール |
| UDP unicast `127.0.0.1:(50000 + n)` | 同じ CM4 上の `ai_cmd_v2.out`（位置制御ループ） |

機体番号 `N` で書けば、どちらも `50100 + N` である。

### loopback unicast

`ai_cmd_v2.out` は、mode 4 で位置制御ループを閉じるために自機の現在位置（feedback の byte 44..51）を必要とする。しかし **`/dev/serial0` の読み手は増やさない**。2 つのプロセスで読むと取り合いになるためである。そこで `forward_robot_feedback.cpp` が multicast で再配信するのと同時に loopback へも unicast で送り、`ai_cmd_v2.out` がそれを受ける。

- `ai_cmd_v2.out` は **`127.0.0.1` に bind する**（`INADDR_ANY` ではない）。ポート番号が multicast の再配信と同じなので、`INADDR_ANY` だと構成によっては自分の再配信のコピーまで位置制御の入力に混ざる。
- unicast は unicast のソケットへ、multicast は multicast のソケットへしか配送されないので、同じポート番号でも取り違えは起きない。
- シミュレータ（`cm4_sim.out`）も同じポートを同じ方法で bind する。制御プロセスの feedback 受信コードとポート番号は実機とシミュレータで同じになる。
