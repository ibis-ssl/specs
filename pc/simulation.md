# シミュレーション連携

Orionのcrane・CM4相当プロセス・frameworkのsimulator-cliを接続する契約を定義する。4WSの物理モデルや操舵制御の対応を示すものではない。CM4相当プロセスのビルド、CLI、劣化注入、診断は[Orion_CM4の運用文書](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/simulation.md)を参照する。

## 経路とパケット

```text
crane --機体別UDP 65B--> cm4_sim --UDP 715B--> simulator-cli
                            ^                     |
                            +--loopback feedback--+
                            |
                            +--multicast feedback--> crane
```

craneからCM4相当への入力は[制御パケット](../common/control_packet.md)に従い、先頭はCHECK_COUNTERである。機体は受信ポートで識別する。CM4相当からsimulator-cliへの出力は65バイトのスロット11個、合計715バイトとする。

各出力スロットは`[robot_id 1B][指令64B]`で、ここでは先頭がIDである。担当外・未受信スロットはIDを`0xFF`、指令を全ゼロにする。mode 4はCM4相当でmode 3へ変換する。craneの65バイト入力を715バイト受信側へ直接接続しない。

## ポート

| 用途 | 既定値 |
|---|---|
| crane → CM4相当 | `127.0.0.1:(12400 + ID)`、担当IDごとに受信 |
| CM4相当 → simulator-cli | `127.0.0.1:12346` |
| simulator-cli → CM4相当 | `127.0.0.1:(50100 + ID)`、unicast |
| crane → 位置制御設定 | `127.0.0.1:12350`、プロセス全体に適用 |
| CM4相当 → crane | [実機と共通のfeedback multicast](../common/system_overview.md) |

feedbackのunicastはCM4相当だけが受け、craneは再配信multicastを受ける。同じunicastへcraneをbindさせない。`SO_REUSEPORT`を付けて同じ受信口を共有すると配送先が分かれ、制御に必要なfeedbackを片方が受け取れなくなる。

## feedbackと制御の条件

- simulator-cliは[共通128バイト配置とCRC](../common/feedback_packet.md)を生成する必要がある。CM4相当は不正フレームを制御と再配信へ渡さない。
- byte 3はシミュレータの自走カウンタで、特定の指令の反射とは対応しない。
- CM4相当はfeedbackの推定位置で位置制御を閉じる。
- simulator-cliが指令のビジョン座標と実位置を0.5 m以内で照合するため、CM4相当は既定で`VISION_GLOBAL_X/Y`をfeedback由来の位置へ差し替える。feedbackを得る前は上流値を使う。実機のCM4はこの差し替えを行わない。
- 位置が毎周期更新されても、シミュレータの速度は指令受信時の`RadioResponse`由来のキャッシュである。指令が捨てられている間、速度だけが古い値で残り得る。診断で速度を使う場合は更新を確認する。

## 実機との停止動作の差

実機の停止条件は[停止判定](../common/safety_stop.md)を参照する。CM4相当のmode 3素通しは指令タイムアウトで停止側の出力を作る。実機ではMainのAI接続タイムアウトが担当するため、停止までの時間は一致しない。

シミュレータのMain相当は125 Hz、OrionMainは500 Hzで動く。シミュレータのstandby猶予と物理モデルは実機の停止距離を保証しない。能動制動と駆動力ゼロを区別して評価する。

## Dockerイメージ

イメージは`ghcr.io/ibis-ssl/orion-cm4-sim`。crane側と同じ接続条件を満たすタグを選び、craneの`CM4_SIM_TAG`で固定する。イメージ更新時は送信器・feedback生成側の対応版も確認する。

| 利用側の契約 | イメージの提供内容 |
|---|---|
| `entrypoint: ["tini", "--"]` | `tini`がPATH上にある |
| `command: cm4_sim ...` | `cm4_sim`がPATH上にある |
| `network_mode: host` | 同じホストのloopbackでcrane・simulator-cliと通信 |

イメージの作成・配布と起動例はCM4側の運用文書に置く。対応未確認の外部実装を、この契約へ適合済みとは扱わない。
