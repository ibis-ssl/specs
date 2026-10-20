# STM（Orion / Atlas）概要

`stm/` は Orion（4輪オムニホイール）と Atlas（4輪ステアリング）の Main 基板について、機器間の対外仕様と、担当repoの実装・運用文書への入口を扱う。CM4 から送るパケット定義は共通とし、物理通信と受信後の制御は機種別にまとめる。

| 機種 | 駆動方式 | 文書 |
|---|---|---|
| Orion | 4輪オムニホイール | [Orion：STM 概要](orion_overview.md) |
| Atlas | 4輪ステアリング | [Atlas：STM 概要](atlas_overview.md) |

## 共通コマンド

CM4 から送るコマンドのレイアウト、符号化、座標系、単位、各フィールドの意味は [制御パケット](../common/control_packet.md) を正本とする。[CM4 との通信](cm4_interface.md) に関連文書と機種別の通信方式へのリンクをまとめる。

## 機種別の実装

外部通信・停止・更新契約はspecsで機種別に定義する。各輪の制御演算・内部状態・ビルド・書込み・診断操作の正本は担当repoに置き、以下の案内から参照する。

| Orion の文書 | 内容 |
|---|---|
| [概要・操作手順](orion_overview.md) | Main repoの操作手順への案内 |
| [基板の対外仕様](orion_board_spec.md) | 外部通信、制御周期、起動・校正、保護条件 |
| [速度制御](orion_speed_control.md) | Main repoの内部制御への案内 |
| [CM4 との UART](orion_uart_cm4.md) | 受信・送信条件と通信負荷 |
| [ブートローダー](orion_bootloader.md) | Flash 配置、A/B 起動判定、更新中の安全動作 |
| [診断機能](orion_diagnostics.md) | Main repoの診断文書への案内 |

機種別の契約と案内を区別して管理する。Atlas の詳細仕様の記載状況は [Atlas：STM 概要](atlas_overview.md) を参照。
