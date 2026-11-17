# ロボットシステム共通仕様

AI（crane）・CM4・Main・PC・シミュレータにまたがる通信契約と責務を管理する。上位インターフェイスの配置・意味・単位、機器の責務、適用機種、安全・互換性の正本はこのrepoに置く。AtlasのMain―周辺基板間のCAN接続詳細はMainFWで、機器内部の実装と操作は各実装repoで管理する。

## システム共通仕様

| 文書 | 定義する内容 |
|---|---|
| [システム構成](common/system_overview.md) | 機器の役割・機体番号・外部通信 |
| [制御パケット](common/control_packet.md) | AI→CM4指令、CM4→OrionMainのUART枠、符号化、モード、CHECK_COUNTER |
| [位置制御設定](common/config_packet.md) | AI→CM4のゲイン設定契約 |
| [Main共通feedback](common/feedback_packet.md) | Main→CM4→PCの配置・CRC・同期・単位 |
| [停止判定](common/safety_stop.md) | 各層の停止責務・タイムアウトと停止の意味 |
| [Orion CAN](common/orion_can.md) | MainとCANノードの通信・ID・バス割当 |
| [Atlasの責務・互換設計](proposals/atlas_can.md) | Main・ステア・電源の役割、校正情報の提供、安全・互換性。CAN詳細はMainFWへ参照 |
| [MCU更新](common/firmware_update.md) | CM4・Main・CANノード間の更新契約と給電条件 |
| [AI連携](pc/crane_integration.md) | 送受信側の適合条件と対応版の確認 |
| [シミュレーション連携](pc/simulation.md) | crane・CM4相当・simulator-cli間の接続契約 |

Orionは4輪オムニ機体で、OrionMain（STM32G474）とのUART経路を現行対象とする。4WSのSPI・各輪制御・ボール基準制御と共通状態パケットは[未実装の設計案](proposals/README.md)に分離する。Atlasの記載範囲は[システム構成](common/system_overview.md#atlasの記載範囲)に示し、未照合の4WS実装への対応保証として扱わない。

## 実装文書への入口

| 担当repo | 管理する内容 |
|---|---|
| [Orion_CM4](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/overview.md) | CM4のプロセス・制御計算・カメラ・ビルド・設定・ホストツール・フリート管理 |
| [G474_Orion_main](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/overview.md) | Main内の制御演算・起動処理・ビルド・書込み・診断 |
| [4WS_MainFW](https://github.com/ibis-ssl/4WS_MainFW/blob/main/doc/overview.md) | 4WS Mainの実装状況・CAN互換範囲・内部ドライバ・検証結果。SPIと機体制御は未実装 |
| [4WS_G474_Controller](https://github.com/ibis-ssl/4WS_G474_Controller/blob/main/doc/overview.md) | DualDCMotorDriver_V4の2ステアFW・設定保持・診断・検証。Atlas新CAN設計への移行は未完了 |
| [crane](https://github.com/ibis-ssl/crane) | AI内の送受信実装・ROS設定・起動 |
| framework | simulator-cliの実装・物理モデル・起動方法は担当repoで管理 |

内部実装・操作の本文は担当repoで保守する。案内専用の個別ページは作らず、本目次または関連する仕様本文から実体へ直接リンクする。

## Orionの対外仕様

| 文書 | 内容 |
|---|---|
| [Main基板](stm/orion_board_spec.md) | 外部通信・制御周期・起動・保護条件 |
| [CM4とのUART](stm/orion_uart_cm4.md) | 指令の採用条件・応答・通信負荷 |
| [Mainブートローダー](stm/orion_bootloader.md) | Flash配置・起動状態・UART更新・安全動作 |

## 実装・操作の詳細

- CM4：[ブリッジ](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/bridge.md)、[位置制御](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/position_control.md)、[カメラ](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/camera.md)、[ホストツール](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/host_tools.md)、[フリート管理](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/fleet.md)。
- Main：[ハードウェア実装](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/board_spec.md)、[速度制御](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/hw_control.md)、[UART実装](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/uart_cm4.md)、[導入手順](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/bootloader.md)、[診断](https://github.com/ibis-ssl/G474_Orion_main/blob/main/doc/diagnostics.md)。

## 文書の参照方向と更新ルール

1. 上位インターフェイスとシステムの責務・安全・互換性はspecsで定義する。AtlasのMain―周辺基板間のCAN接続仕様はMainFWを正本とし、specsから参照する。項目ごとの正本を一つにし、実装コードから未確認の契約を推測しない。
2. specsから各repoへのリンクは、実装・操作の詳細、または担当repoが所有する接続仕様へ進む入口とする。各repoの全文やCAN定義表をspecsへ複製しない。
3. 共通インターフェイスを変更するときは、specsの作業ブランチで契約を更新し、関係repoの実装・文書・結合確認を揃える。
4. モジュール内部だけの変更は担当repoで完結させる。共通契約への影響がある場合だけspecsを更新する。
5. 現行仕様には現在の契約だけを書く。未実装案は`proposals/`、変更経緯・実機計測・過去の調査資料は各repoの作業ログ・履歴へ置く。
6. 日本語・UTF-8で記載する。コードの参照はファイル名・シンボル名で示す。外部repoの適合や配備済み状態を、未確認のまま断定しない。

## 参照方向

```text
各実装repoの文書 → specsの共通契約 → specs内の共通定義
specsの目次・関連仕様本文 → 各実装repoの詳細・操作
```

システム・上位契約への依存は上段の向きとする。下段は内部実装への案内に加え、AtlasではMainFWが所有するCAN接続仕様への参照も含む。同じ配置・単位・閾値を両側へ複製しない。内部実装に契約との差が残る場合は各repoで差と影響を記録し、契約に適合済みと断定しない。

## 仕様の所有と未決事項

| 内容 | 本文を置く場所 | 実装repoに残す内容 |
|---|---|---|
| 機器の役割、上位指令・帰還の意味/単位/座標系、停止責務、更新・互換性のシステム契約 | specsの`common/`と機種別対外仕様 | 対応API、実装済み範囲、非互換点、検証結果 |
| 未実装の上位通信・制御分担・停止責務・モード切替の案と未決事項 | specsの`proposals/` | 未実装という状態、実装上の障害、設計案へのリンク |
| AtlasのCAN ID・予約・ノード/軸/バス割当、DLC・byte配置・opcode・CRC・分割転送・周期・帯域 | MainFWの接続仕様 | ステア等は同じ仕様を参照し、実装差だけを記録する |
| Flash保存形式、RTT/開発RPC、GUI、ログ、操作・ビルド | 各実装repo | 内部実装と手順の本文 |
| ピン、HAL・DMA・IRQ、バッファ、モジュール構成、ビルド | 各実装repo | 内部設計・手順の本文 |
| 実測値、旧ソース調査、変更履歴 | 各実装repo | 日付・対象・条件・未確認範囲 |

共通契約に未定義の事項がある場合、実装から読み取った値を無条件に他機器の要件へ昇格させない。照合した対象と未確認範囲を明記する。未実装案では既知の要求と未決事項を分け、実装済みの仕様表へ混在させない。

変更時はspecsの作業ブランチで契約・適用範囲を更新し、実装repoの重複定義を参照へ置き換える。共通文書のリンク先・見出しと、実装repoが参照するspecsの版を確認する。ソースを確認するための照合情報と、契約を定義する参照先は区別する。

## 実装照合の範囲

CM4の制御UDP外枠、feedback配置・CRC、シミュレータ入力、PCデコーダは`Orion_CM4`の`bc1a9af`時点のコードを照合対象とする。これは外部repoへの配備済み状態を示さない。Main・crane・frameworkはそれぞれの実装とこの契約の結合確認が必要である。

## 新しい項目を置く際の判断基準

1. AI・CM4等の上位利用者が守る意味・責務・安全/互換性ならspecsへ置く。
2. AtlasのMainと周辺基板の間だけで完結する接続方法ならMainFWへ置く。複数基板に影響するという理由だけではspecsに入れない。
3. 基板や通信方式を交換しても維持する要件と、具体的なフレーム・保存・実装方法を分ける。

例えば「Mainが要求せず校正情報を取得できる」はspecs、「約1秒周期でどのCAN IDへどう分割するか」はMainFW、「DMAやFlashの実装」は各基板FWが所有する。
上位の起動・欠落判定に必要な取得期限はspecsへ、具体的な送信周期は接続仕様へ置く。
既存のOrion CAN・ブートローダー等は電源互換の参照として維持する。今回のAtlasの分類変更を理由に既存Orionの詳細を一括削除しない。
AtlasのCAN詳細は[MainFWの設計](https://github.com/ibis-ssl/4WS_MainFW/blob/main/doc/design/atlas_can.md)を参照する。
