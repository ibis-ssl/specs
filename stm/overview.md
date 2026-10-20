# STM（Orion / Atlas）概要

`stm/` は Orion（4輪オムニホイール）と Atlas（4輪ステアリング）の Main 基板について、外部連携・制御動作・運用に関わる仕様を扱う。CM4 から送るパケット定義は共通とし、物理通信と受信後の制御は機種別にまとめる。

| 機種 | 駆動方式 | 文書 |
|---|---|---|
| Orion | 4輪オムニホイール | [Orion：STM 概要](orion_overview.md) |
| Atlas | 4輪ステアリング | [Atlas：STM 概要](atlas_overview.md) |

## 共通コマンド

CM4 から送るコマンドのレイアウト、符号化、座標系、単位、各フィールドの意味は [制御パケット](../common/control_packet.md) を正本とする。[CM4 との通信](cm4_interface.md) に関連文書と機種別の通信方式へのリンクをまとめる。

## 機種別の実装

コマンドを各輪の駆動へ変換する処理、オドメトリ、制御周期・ゲイン、通信設定、停止処理、テレメトリ、ブートローダー、書込み・診断手順は機種別に扱う。

| Orion の文書 | 内容 |
|---|---|
| [概要・操作手順](orion_overview.md) | モードスイッチ、LED、ビルドと書込み、デバッグ UART |
| [基板の対外仕様](orion_board_spec.md) | 外部通信、制御周期、起動・校正、保護条件 |
| [速度制御](orion_speed_control.md) | 4輪オムニの目標生成と各輪の出力計算 |
| [CM4 との UART](orion_uart_cm4.md) | 受信・送信条件と通信負荷 |
| [ブートローダー](orion_bootloader.md) | Flash 配置、A/B 起動判定、更新中の安全動作 |
| [診断機能](orion_diagnostics.md) | DRIVE_LOG、エンコーダーノイズ、静止時の角度誤差クリア |

既存の実装詳細は Orion のものとして記載している。Atlas の詳細仕様の記載状況は [Atlas：STM 概要](atlas_overview.md) を参照。
