# CM4 からの共通コマンド

Orion（4輪オムニホイール）と Atlas（4輪ステアリング）は、CM4 から送る制御コマンドの定義を共有する。パケットのレイアウト、符号化、座標系、単位、各フィールドの意味は [制御パケット](../common/control_packet.md) を参照。

| 共通仕様の参照先 | 内容 |
|---|---|
| [制御モード](../common/control_packet.md#制御モード) | CM4 が位置指令 mode 4 を速度指令 mode 3 に変換して Main へ送る |
| [UART フレーム](../common/control_packet.md#uart-フレームcm4--main72-バイト) | CM4 から送るコマンド、ローカルカメラ情報、チェックサムの配置 |
| [CM4 ブリッジ](../cm4/bridge.md) | CM4 側の受信、位置制御への分岐、UART 送信 |

## 機種別の受信・送信実装

| 機種 | 実装文書 |
|---|---|
| Orion | [CM4 との UART（USART2）](orion_uart_cm4.md) |
| Atlas | [Atlas：STM 概要](atlas_overview.md)（受信・送信実装は未記載） |

MCU 側の UART ピン、FIFO・DMA・割り込み、タイムアウト処理、ファームウェア更新要求、Main から返すテレメトリは機種別の実装として扱う。これらの既存の記載は Orion を対象とする。
