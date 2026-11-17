# システム共通の設計案

このディレクトリは未実装のインターフェイス案を扱う。現行仕様の正本は[specsの入口](../README.md)から参照する。案を採用して実装した場合は、実装と照合して該当する現行仕様へ反映する。

- [制御モード5～8と4WS通信](control_modes.md)
- [CM4共通状態パケット](cm4_status_packet.md)
- [Atlasの責務・互換設計](atlas_can.md)：Main・2基板4軸ステア・電源の責務、安全、校正情報の提供。ID/周期/フレームは[MainFWの接続設計](https://github.com/ibis-ssl/4WS_MainFW/blob/main/doc/design/atlas_can.md)で管理する。

実験記録と過去の計画は各実装リポジトリの作業ログ・履歴に置く。
