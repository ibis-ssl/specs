# CM4 との UART（USART2）

Main 基板の USART2 で CM4 と通信する部分の実装をまとめる。フレームの形式は [制御パケット](../common/control_packet.md#uart-フレームcm4--g47472-バイト)（受信 72 B）と [フィードバックパケット](../common/feedback_packet.md)（送信 128 B）、タイムアウトは [停止判定](../common/safety_stop.md#g474-の通信タイムアウト) を参照。

## 設定

- 1 Mbps、8N1、PB3 TX / PB4 RX（pin swap）。
- RX FIFO を有効にする（threshold 1/8）。生成コードの `MX_USART2_UART_Init()` は FIFO を無効にするが、同じ関数の USER CODE で再び有効にしている。
- **`HAL_UART_Init()` を起動処理で二度呼ばない。** 二度目の呼び出しで FIFOEN が消える。初期化は `MX_USART2_UART_Init()` だけで行う。
- TX は DMA1 Channel8 の normal モード。NVIC は `NVIC_PRIORITYGROUP_4` で、DMA1 Channel8 が優先度 0、USART2 が 1、テレメトリ送信を起動する TIM7 が 8。DMA の転送優先度 LOW は NVIC の優先度とは別物である。

## 受信

- HAL の 1 バイト受信の状態機械は使わない。RXNE/RXFNE とエラー割り込み（EIE）を起動時に直接有効にし、常時有効のままにする。
- `USART2_IRQHandler` は RX FIFO が空になるまで読み切り、1 バイトずつ `cm4_uart_rx_byte()` → `HAL_UART_RxCpltCallback()` のパーサへ渡す。パーサは `0xFE` を見つけてから 72 バイトを集め、チェックサムを確認する。
  - チェックサムが正しく byte 1..4 が `FWVR` / `FWUP` なら、バージョン照会・更新要求として扱う（[MCU ファームウェア更新](../common/firmware_update.md#更新モードへの切り替え)）。
  - それ以外でチェックサムが正しければ制御指令として取り込み、CM4 との接続のタイムスタンプを更新する。
- FW ゲートウェイの動作中は、受けたバイトを `fw_update_gateway.c` の 2 KB のリングキューへ入れ、メインループ側で OFW2 フレームとして解釈する。
- ISR は最後に PE/FE/NE/ORE をクリアし、残りの割り込み要因（通常テレメトリの送信完了など）を `HAL_UART_IRQHandler()` に渡す。

### ATOMIC_SET_BIT が必要な理由

USART2 の ISR の末尾と、`main.c` で受信を始めるところでは、CR3.EIE と CR1.RXNEIE_RXFNEIE を `ATOMIC_SET_BIT` で設定する（計 4 か所）。通常の `SET_BIT` を使ってはいけない。

- HAL の DMA 送信完了処理は、CR3.DMAT をクリアし CR1.TCIE をセットする。gState を READY に戻すのは、そのあとの USART2 の TC 処理である。
- 通常の read-modify-write だと「USART2 側が CR1 を読む → DMA 完了 IRQ が割り込んで TCIE をセット → USART2 側が古い CR1 に RXNEIE を加えて書き戻し、TCIE を消す」という順序が起こりうる。すると送信済みでも gState が BUSY_TX のまま残り、以後の送信が HAL_BUSY になる。CR3 側では、DMA 完了でクリアした DMAT を復活させうる。
- HAL 側だけが atomic でも、呼び出し側の通常の read-modify-write は守られない。`ATOMIC_SET_BIT`（CMSIS の LDREXW/STREXW による再試行）は、割り込みを一律に禁止せずにこの更新競合を防ぐ。

## テレメトリ送信

- 制御周期の中で 125 Hz に間引き、`sendRobotInfo()` が 128 B を DMA で送る。
- 次のときは送らない：デバッグ UART で `t` を押して止めているとき（もう一度押すと再開。FW 更新の応答には影響しない）、FW ゲートウェイの動作中（OFW2 の応答と取り合わないため）。
- `sendRobotInfo()` は `HAL_UART_Transmit_DMA()` の戻り値を見ていない。また static の送信バッファを作り直してから HAL に渡すので、前回の DMA が進行中だとデータを壊しうる。

## 診断カウンタ

デバッグ UART の `UART` ページに、次を表示する。

- `RX age`：最後に正しい指令フレームを受けてからの経過時間。UART の受信が止まったかどうかはこれで判断する。
- USART2 の割り込み回数、そのうち RX バイトを実際に読み出した割り込み回数、受信バイト数、FIFO を一度に読み出した最大数
- 72 バイトのフレームを組み立て終えた数と、チェックサムが正しかった数（物理層の受信とパーサの結果を比べられる）
- PE / FE / NE / ORE の回数
- テレメトリ送信の有効・無効

`AI_CMD` ページなどに出る `Dt` は、UART フレームの受信間隔ではなく、`check_counter` が最後に変化してからの時間である。

FW ゲートウェイの動作中は、ISR とキューのバイト数とローリングハッシュ、キューのオーバーフロー、magic / header / frame / CRC のエラー、パーサのタイムアウト、応答の成功・失敗を RAM に保持する。

### 途中フレームのタイムアウト（FW ゲートウェイ）

ゲートウェイのパーサは、受信途中のフレームが一定時間進まなければリセットする。このとき `uart_last_byte_tick` を先にスナップショットし、そのあと `HAL_GetTick()` を取得する。判定の直前に、スナップショットと現在の `uart_last_byte_tick` が一致する場合だけリセットする。逆の順序で評価すると、その間に RX IRQ が値を更新したとき符号なしの減算がアンダーフローし、受信途中のフレームを誤ってリセットする。
