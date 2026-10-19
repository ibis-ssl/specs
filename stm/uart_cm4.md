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


## テレメトリ送信

- 制御周期の中で 125 Hz に間引き、`sendRobotInfo()` が 128 B を DMA で送る。
- 次のときは送らない：デバッグ UART で `t` を押して止めているとき（もう一度押すと再開。FW 更新の応答には影響しない）、FW ゲートウェイの動作中（OFW2 の応答と取り合わないため）。
- `sendRobotInfo()` は `HAL_UART_Transmit_DMA()` の戻り値を見ていない。また static の送信バッファを作り直してから HAL に渡すので、前回の DMA が進行中だとデータを壊しうる。
