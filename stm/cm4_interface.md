# CM4 との通信

Orion と Atlas は、CM4 から送るパケットの定義を共有する。パケットのレイアウト、符号化、座標系、単位、各フィールドの意味は [制御パケット](../common/control_packet.md) を正本とする。物理通信方式と採用状況の正本は [システム構成](../common/system_overview.md#cm4-と-main-の通信)を参照。

| 共通仕様の参照先 | 内容 |
|---|---|
| [制御モード](../common/control_packet.md#制御モード) | CM4 が解釈・生成するコマンドの意味 |
| [CM4→Main パケット](../common/control_packet.md#cm4main-パケット72-バイト) | コマンド、未定義領域、チェックサムの配置 |
| [CM4 位置制御](../cm4/position_control.md) | CM4 側の制御則とコマンド生成 |

## 機種別の受信・送信実装

| 機種 | 実装文書 |
|---|---|
| Orion | [UART の受信・送信処理](orion_uart_cm4.md)、[CM4 ブリッジ](../cm4/bridge.md) |
| Atlas | [SPI の採用予定・記載状況](atlas_overview.md#cm4-との通信) |

通信設定と転送手順は機種別に扱う。停止・更新・Main から返すテレメトリについても適用機種を区別し、Orion の既存実装から Atlas の仕様を推測しない。
