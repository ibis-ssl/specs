# STM（Orion / Atlas）概要

`stm/` は Orion（4輪オムニホイール）と Atlas（4輪ステアリング）の Main 基板ファームウェアの文書を扱う。CM4 から送るコマンドの定義までは共通とし、受信後の駆動制御と基板ごとの実装は機種別にまとめる。

| 機種 | 駆動方式 | 文書 |
|---|---|---|
| Orion | 4輪オムニホイール | [Orion：STM 概要](orion_overview.md) |
| Atlas | 4輪ステアリング | [Atlas：STM 概要](atlas_overview.md) |

## 共通コマンド

CM4 から送るコマンドのレイアウト、符号化、座標系、単位、各フィールドの意味は [制御パケット](../common/control_packet.md) を正本とする。[CM4 からの共通コマンド](uart_cm4.md) に関連文書と機種別の受信実装へのリンクをまとめる。

## 機種別の実装

コマンドを各輪の駆動へ変換する処理、オドメトリ、制御周期・ゲイン、CAN ID、MCU・ピン・ペリフェラル、停止処理、テレメトリ、ブートローダー、書込み・診断手順は機種別に扱う。

| Orion の文書 | 内容 |
|---|---|
| [概要・操作手順](orion_overview.md) | モードスイッチ、LED、ビルドと書込み、デバッグ UART |
| [ハードウェア仕様](orion_board_spec.md) | MCU、クロック、ピン、ペリフェラル、CAN ID、NVIC |
| [速度制御](orion_speed_control.md) | 4輪オムニの目標生成と各輪の出力計算 |
| [CM4 との UART](orion_uart_cm4.md) | USART2 の受信・送信の実装 |
| [ブートローダー](orion_bootloader.md) | Flash 配置、A/B 起動判定、安全 IO |
| [診断機能](orion_diagnostics.md) | DRIVE_LOG、エンコーダーノイズ、静止時の角度誤差クリア |

既存の実装詳細は Orion のものとして記載している。Atlas の詳細仕様の記載状況は [Atlas：STM 概要](atlas_overview.md) を参照。
