# AI・CM4・Mainの連携条件

この文書は各リポジトリの境界で満たす契約をまとめる。craneのROSパラメータ・launch構成・内部クラスの操作はcraneリポジトリ、CM4のCLIと診断は[Orion_CM4](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/overview.md)で管理する。

## 送信側の要件

- 指令は[65バイト制御パケット](../common/control_packet.md)として機体別に送る。先頭はCHECK_COUNTERで、機体IDではない。
- 実機の宛先は[ネットワーク仕様](../common/system_overview.md)、simの宛先は[シミュレーション契約](simulation.md#ポート)に従う。
- mode 4はCM4で位置制御し、OrionMainへはmode 3を渡す。
- 最後にビジョンで捉えてからの経過時間と遅延は、uint16の範囲に飽和させて符号化する。未知の観測を経過時間0として送らない。
- 位置制御ゲインは[設定パケット](../common/config_packet.md)を使う。指令本体の予約領域へ相乗りさせない。
- 指令と設定を複数機体へ送る場合は、それぞれの宛先へ送る。設定はsimの共通プロセスに対しては1回でよい。

## 受信側の要件

craneと生データデバッグツールは[共通feedback](../common/feedback_packet.md)の配置・単位を使う。位置、速度、yawのバイト位置を個別の旧定義から読み取らない。実機でもsimでもcraneはmulticastを受け、CM4内のunicast受信口を共有しない。

CM4のmode 4経路では指令カウンタをCM4が採番する。Mainが接続中であることだけではcraneの生存を判断できない。停止責務は[停止判定](../common/safety_stop.md)に従う。

## 対応版の確認

送信器、CM4、Main、PCデコーダ、シミュレータを同じパケット契約へ揃える。設定パケットはサイズ・versionの完全一致、feedbackは共通配置・CRCが必要である。長さが同じでも配置が違う受信実装は適合しない。

この文書は外部リポジトリへの配備済み状態を保証しない。各リポジトリの実装・結合試験・配布タグを確認する。4WSのSPIと共通状態パケットへの切替は[設計案](../proposals/README.md)であり、現行の接続条件へ混在させない。
