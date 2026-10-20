# ロボットシステム共通仕様

AI（crane）・CM4・Main・PC・シミュレータにまたがる通信契約と責務を管理する。パケットの配置・符号化・単位・適用機種・停止条件の正本はこのrepoに置く。機器内部に閉じる実装と操作の正本は、各実装repoの文書に置く。

## システム共通仕様

| 文書 | 定義する内容 |
|---|---|
| [システム構成](common/system_overview.md) | 機器の役割・機体番号・外部通信 |
| [制御パケット](common/control_packet.md) | AI→CM4指令、CM4→OrionMainのUART枠、符号化、モード、CHECK_COUNTER |
| [位置制御設定](common/config_packet.md) | AI→CM4のゲイン設定契約 |
| [Main共通feedback](common/feedback_packet.md) | Main→CM4→PCの配置・CRC・同期・単位 |
| [停止判定](common/safety_stop.md) | 各層の停止責務・タイムアウトと停止の意味 |
| [Orion CAN](common/orion_can.md) | MainとCANノードの通信・ID・バス割当 |
| [MCU更新](common/firmware_update.md) | CM4・Main・CANノード間の更新契約と給電条件 |
| [AI連携](pc/crane_integration.md) | 送受信側の適合条件と対応版の確認 |
| [シミュレーション連携](pc/simulation.md) | crane・CM4相当・simulator-cli間の接続契約 |
| [Mainとの通信](stm/cm4_interface.md) | 機種別の物理通信・対外仕様への入口 |

Orionは4輪オムニ機体で、OrionMain（STM32G474）とのUART経路を現行対象とする。4WSのSPI・各輪制御・ボール基準制御と共通状態パケットは[未実装の設計案](proposals/README.md)に分離する。`stm/`にあるAtlasの記載を、未照合の4WS実装への対応保証として扱わない。

## 実装文書への入口

| 担当repo | 管理する内容 |
|---|---|
| [Orion_CM4](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/overview.md) | CM4のプロセス・制御計算・カメラ・ビルド・設定・ホストツール・フリート管理 |
| [G474_Orion_main](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/overview.md) | Main内の制御演算・起動処理・ビルド・書込み・診断 |
| [crane](https://github.com/ibis-ssl/crane) | AI内の送受信実装・ROS設定・起動 |
| framework | simulator-cliの実装・物理モデル・起動方法は担当repoで管理 |

`cm4/`、`pc/host_tools.md`、`pc/fleet.md`はCM4 repoへの案内、`stm/orion_overview.md`・`orion_speed_control.md`・`orion_diagnostics.md`はMain repoへの案内とする。内部実装・操作の本文は担当repoで保守する。

## 文書の参照方向と更新ルール

1. 共通仕様の規範はspecs内の文書が定義し、各repoの実装文書から参照する。コードや各repoの文書を共通定義の正本に指定しない。
2. specsから各repoへのリンクは実装・操作の詳細へ進むための入口とする。各repoの全文をspecsへ複製しない。
3. 共通インターフェイスを変更するときは、specsの作業ブランチで契約を更新し、関係repoの実装・文書・結合確認を揃える。
4. モジュール内部だけの変更は担当repoで完結させる。共通契約への影響がある場合だけspecsを更新する。
5. 現行仕様には現在の契約だけを書く。未実装案は`proposals/`、変更経緯・実機計測・過去の調査資料は各repoの作業ログ・履歴へ置く。
6. 日本語・UTF-8で記載する。コードの参照はファイル名・シンボル名で示す。外部repoの適合や配備済み状態を、未確認のまま断定しない。

## 参照方向

```text
各実装repoの文書 → specsの共通契約 → specs内の共通定義
specsの案内ページ → 各実装repoの詳細・操作
```

仕様上の依存は上段の向きとする。下段は読者向けの案内であり、リンク先の内部実装を共通契約の正本にしない。同じ配置・単位・閾値を両側へ複製しない。内部実装に契約との差が残る場合は各repoで差と影響を記録し、契約に適合済みと断定しない。

## 実装照合の範囲

CM4の制御UDP外枠、feedback配置・CRC、シミュレータ入力、PCデコーダは`Orion_CM4`の`bc1a9af`時点のコードを照合対象とする。これは外部repoへの配備済み状態を示さない。Main・crane・frameworkはそれぞれの実装とこの契約の結合確認が必要である。
