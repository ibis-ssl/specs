# Orion：Main 基板の対外仕様

Orion（4輪オムニホイール）の Main 基板について、上位との接続、制御周期、起動と保護に関わる仕様をまとめる。対象ファームウェアは `G474_Orion_main`。機種別の入口は [STM 概要](../README.md#orionの対外仕様)を参照。

## 対象と役割

- MCU: STM32G474RETx
- Main は CM4 の指令を受け、走行・キッカー・ドリブラーを制御し、状態を CM4 へ返す。
- Atlas の通信方式と記載状況は [システム構成](../common/system_overview.md#cm4-と-main-の通信)を参照。

## 制御周期

- 主制御周期は 2 ms（500 Hz、`MAIN_LOOP_CYCLE = 500`）。
- ロボット情報の送信周期は [フィードバックパケット](../common/feedback_packet.md#送信元と経路)を参照。
- 制御方法は [速度制御](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/hw_control.md)、停止条件とタイムアウトは [停止判定](../common/safety_stop.md)を参照。

## 外部通信

### CM4 との通信

- 通信方式の適用機種と採用状況は [システム構成](../common/system_overview.md#cm4-と-main-の通信)を参照。
- 通信設定: 1,000,000 bps、8N1。
- 受信するパケットは [CM4→Main パケット](../common/control_packet.md#cm4main-パケット72-バイト)、送信するパケットは [フィードバックパケット](../common/feedback_packet.md)に従う。
- 受信条件、テレメトリ送信の制約、通信負荷は [Orion：CM4 との UART](orion_uart_cm4.md)を参照。

### デバッグ UART

- 通信設定: 2,000,000 bps、8N1。
- 起動ログ、状態表示、デバッグ表示の切替、ゲイン調整に使う。操作方法は[Main開発文書](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/overview.md)を参照。

### CANノードとの通信

バスの設定・論理割当・送受信IDは[MainとCANノードの通信](../common/orion_can.md)を参照する。

## 起動と校正

- 起動時に IMU の yaw ジャイロをキャリブレーションし、角度をクリアする。姿勢角には yaw を利用する。
- アクチュエータを停止状態に設定し、キッカー充電と保護条件を設定する。
- 1 秒間の停止要求を設定して主制御を開始する。初期化中の停止条件は [停止判定](../common/safety_stop.md#g474-の走行停止条件)を参照。
- モードスイッチと LED の表示は [Orion 概要](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/overview.md)を参照。

## 保護条件

- 低電圧判定の基準は `LOW_VOLTAGE_LIMIT = 22.5` V。
- 起動時に電源基板へ送る保護設定は次のとおり。

| 項目 | 設定値 |
|---|---|
| min voltage | 15.0 |
| max voltage | 35.0 |
| max current | 50.0 |
| max FET temp | 90.0 |
| max solenoid temp | 90.0 |

通信途絶・初期化未完了・エラー時の動作は [停止判定](../common/safety_stop.md)を参照。
