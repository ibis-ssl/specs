# Orion・Atlas ロボット ドキュメント

ibis の Orion（4 輪オムニホイール）と Atlas（4 輪ステアリング）の仕様書をまとめたリポジトリ。CM4 から送るパケット定義は共通とし、物理通信は Orion が UART（現行）、Atlas が SPI（採用予定）である。STM 側は外部連携・制御動作・運用に関わる仕様を機種ごとに記載する。現在の STM 実装、フィードバック、停止動作、MCU 更新の詳細は Orion の記録で、Atlas の詳細は未記載である。

## 構成

| ディレクトリ | 内容 | 対応するリポジトリ |
|---|---|---|
| [common/](common/) | 複数の層にまたがる仕様：共通コマンドと、適用機種を明記した構成・フィードバック・停止判定・FW 更新 | すべて |
| [cm4/](cm4/) | CM4 上で動くブリッジ、位置制御、カメラ、セットアップ | `ibis-ssl/Orion_CM4` の `cm4/` |
| [stm/](stm/) | 共通コマンドの入口と、Orion・Atlas の機種別 Main ファームウェア仕様 | Orion：`ibis-ssl/G474_Orion_main`。Atlas：未記載 |
| [pc/](pc/) | ホスト PC のツール、フリート管理、crane やシミュレータとの連携 | `ibis-ssl/Orion_CM4` の `host/`、`ibis-ssl/crane`、framework |

### common/

- [システム構成](common/system_overview.md)：制御機の役割、通信経路、機体番号、IP とポートの一覧
- [制御パケット](common/control_packet.md)：`RobotCommandSerializedV2` と UDP・UART の枠、`check_counter`
- [位置制御設定パケット](common/config_packet.md)：crane から CM4 へ送るゲイン設定（28 B）
- [フィードバックパケット](common/feedback_packet.md)：Orion の G474 が送る 128 B のテレメトリ
- [停止判定](common/safety_stop.md)：CM4 の停止指令と、Orion の停止動作・タイムアウト
- [MCU ファームウェア更新](common/firmware_update.md)：Orion の Main と CAN ノードを CM4 から更新する手順

### cm4/

- [概要](cm4/overview.md)：ディレクトリ、制御 API、ビルド、テスト
- [ブリッジ](cm4/bridge.md)：`ai_cmd_v2.out` と `robot_feedback.out`
- [位置制御](cm4/position_control.md)：`position_controller`
- [カメラ](cm4/camera.md)

### stm/

- [概要](stm/overview.md)：共通部分と機種別仕様の入口
- [CM4 との通信](stm/cm4_interface.md)：共通パケットと機種別の通信方式
- [Orion 概要](stm/orion_overview.md)：ビルドと書込み、モードスイッチ、LED、デバッグ UART
- [Orion 基板の対外仕様](stm/orion_board_spec.md)：外部通信、制御周期、起動・校正、保護条件
- [Orion 速度制御](stm/orion_speed_control.md)
- [Orion ブートローダー](stm/orion_bootloader.md)
- [Orion CM4 との UART 実装](stm/orion_uart_cm4.md)
- [Orion 診断機能](stm/orion_diagnostics.md)
- [Atlas 概要](stm/atlas_overview.md)：4 輪ステアリング。STM 側の詳細は未記載

### pc/

- [ホスト PC 側ツール](pc/host_tools.md)
- [フリート管理](pc/fleet.md)
- [crane との連携](pc/crane_integration.md)：既知の不整合もここに書く
- [シミュレーション](pc/simulation.md)：`cm4_sim`、Docker イメージ、シミュレータと実機の差

## 文書のルール

- この文書群を仕様の正本とする。`G474_Orion_main/doc` と `Orion_CM4/doc` にある同種の文書は旧版で、内容が食い違う場合はこちらが正しい。
- 現在の仕様と、採用予定と明記した計画を扱う。過去の形式、変更の経緯、日付付きの計測結果は各リポジトリの作業ログに書く。未実装の計画を現行実装として書かない。
- 1 つの事実は 1 か所にだけ書き、ほかの文書からはリンクする。たとえば `check_counter` は [制御パケット](common/control_packet.md#check_counter)、タイムアウトは [停止判定](common/safety_stop.md)、ポートは [システム構成](common/system_overview.md#機体番号とネットワーク) にだけ書く。
- コードを参照するときは `file:line` ではなく、ファイル名と関数名・シンボル名で書く（行番号はすぐずれるため）。
- 日本語・UTF-8 で書く。コードや設定との食い違いを見つけたら、実装を確かめて文書を直す。
- CM4 から送るコマンドの定義は共通とし、STM 側の実装と動作は適用機種を明記する。Orion の確認結果を Atlas の仕様として扱わない。
- 基板仕様には外部との通信、制御・保護の動作、操作・更新に必要な事項を記載する。IO 割り当て、周辺回路の設定、ロボット内の CAN フレーム ID などの内部実装は、各ファームウェアのリポジトリで管理する。

## 照合の基準

この文書群の既存実装の記載は、次のコミット時点のコードで確認している。STM 側の照合対象は Orion で、Atlas の STM 実装は未照合である。次に食い違いを調べるときは、ここからの差分を見る。

| リポジトリ | ブランチ | コミット |
|---|---|---|
| G474_Orion_main | main | `154f89a` |
| Orion_CM4 | main | `e03b0f6` |
| crane | develop | `9d516f633` |
| framework | ibis | `a52b6bd` |
