# Orion ロボット ドキュメント

ibis の Orion ロボットの仕様書をまとめたリポジトリ。ロボットは AI PC（crane）、CM4、Main 基板（STM32G474）、CAN 上の各基板で構成され、ここではその全体の仕様を扱う。

## 構成

| ディレクトリ | 内容 | 対応するリポジトリ |
|---|---|---|
| [common/](common/) | 複数の層にまたがる仕様：システム構成、パケット、停止判定、FW 更新 | すべて |
| [cm4/](cm4/) | CM4 上で動くブリッジ、位置制御、カメラ、セットアップ | `ibis-ssl/Orion_CM4` の `cm4/` |
| [stm/](stm/) | Main 基板ファームウェア：ハード仕様、速度制御、ブートローダー、UART、診断 | `ibis-ssl/G474_Orion_main` |
| [pc/](pc/) | ホスト PC のツール、フリート管理、crane やシミュレータとの連携 | `ibis-ssl/Orion_CM4` の `host/`、`ibis-ssl/crane`、framework |

### common/

- [システム構成](common/system_overview.md)：制御機の役割、通信経路、機体番号、IP とポートの一覧
- [制御パケット](common/control_packet.md)：`RobotCommandSerializedV2` と UDP・UART の枠、`check_counter`
- [位置制御設定パケット](common/config_packet.md)：crane から CM4 へ送るゲイン設定（28 B）
- [フィードバックパケット](common/feedback_packet.md)：G474 が送る 128 B のテレメトリ
- [停止判定](common/safety_stop.md)：どこが落ちたら誰が止めるか、各タイムアウトの値
- [MCU ファームウェア更新](common/firmware_update.md)：CM4 から Main と CAN ノードを更新する手順

### cm4/

- [概要](cm4/overview.md)：ディレクトリ、制御 API、ビルド、テスト
- [ブリッジ](cm4/bridge.md)：`ai_cmd_v2.out` と `robot_feedback.out`
- [位置制御](cm4/position_control.md)：`position_controller`
- [カメラ](cm4/camera.md)

### stm/

- [概要](stm/overview.md)：ビルドと書込み、モードスイッチ、LED、デバッグ UART
- [ハードウェア仕様](stm/board_spec.md)
- [速度制御](stm/speed_control.md)
- [ブートローダー](stm/bootloader.md)
- [CM4 との UART](stm/uart_cm4.md)
- [診断機能](stm/diagnostics.md)

### pc/

- [ホスト PC 側ツール](pc/host_tools.md)
- [フリート管理](pc/fleet.md)
- [crane との連携](pc/crane_integration.md)：既知の不整合もここに書く
- [シミュレーション](pc/simulation.md)：`cm4_sim`、Docker イメージ、シミュレータと実機の差

## 文書のルール

- この文書群を仕様の正本とする。`G474_Orion_main/doc` と `Orion_CM4/doc` にある同種の文書は旧版で、内容が食い違う場合はこちらが正しい。
- 書くのは**現在の仕様だけ**。過去の形式、変更の経緯、日付付きの計測結果は各リポジトリの作業ログに書く。まだ実装していない案は仕様として書かない。
- 1 つの事実は 1 か所にだけ書き、ほかの文書からはリンクする。たとえば `check_counter` は [制御パケット](common/control_packet.md#check_counter)、タイムアウトは [停止判定](common/safety_stop.md)、ポートは [システム構成](common/system_overview.md#機体番号とネットワーク) にだけ書く。
- コードを参照するときは `file:line` ではなく、ファイル名と関数名・シンボル名で書く（行番号はすぐずれるため）。
- 日本語・UTF-8 で書く。コードや設定との食い違いを見つけたら、実装を確かめて文書を直す。

## 照合の基準

この文書群の内容は、次のコミット時点のコードで確認している。次に食い違いを調べるときは、ここからの差分を見る。

| リポジトリ | ブランチ | コミット |
|---|---|---|
| G474_Orion_main | main | `154f89a` |
| Orion_CM4 | main | `e03b0f6` |
| crane | develop | `9d516f633` |
| framework | ibis | `a52b6bd` |
