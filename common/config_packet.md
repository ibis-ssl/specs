# 位置制御設定パケット（UDP 12350）

CM4 の位置制御ゲインを、crane から稼働中に変更するためのパケット。外部から設定できる項目と値の範囲をこの文書で定義する。制御則と既定値は[CM4位置制御](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/position_control.md)を参照する。CM4 を再起動する必要はない。

これは CM4 が処理する設定で、Orion と Atlas の STM 側のハードウェアや車輪制御を指定するものではない。以下の G474 に関する実装説明は Orion を指す。

65 バイトの制御パケットには相乗りさせず、**別ポートで 28 バイトのデータグラム**として運ぶ。64 バイトのレイアウトは crane / G474 / framework / CM4 の 4 者で一致させることを不変条件にしているうえ、使用中のコマンドの byte 28..31 と 38..63 がゼロとは限らないためである。別ポートにすれば G474 と framework に一切影響しない。

## 形式

通信契約の正本はこの文書で、CM4の実装は`cm4/bridge/config_packet.h`にある。送信側は対象機体の `192.168.20.(100 + N):12350` へユニキャストで送る。複数の機体に同じ設定を適用するときは、各機体へ個別に送る。

| byte | 内容 |
|---|---|
| 0..3 | magic `'O' 'C' '4' 'C'` |
| 4 | version（`2`） |
| 5 | robot_id（`0xFF` は全機に適用する値という意味。UDP は各機体へ個別に送る） |
| 6..7 | 予約（0） |
| 8..11 | `position_gain`（kp）float32 little endian |
| 12..15 | `deceleration` float32 little endian |
| 16..19 | `position_tolerance` float32 little endian |
| 20..23 | `integral_gain`（ki）float32 little endian |
| 24..27 | `derivative_gain`（kd）float32 little endian |

2 バイト固定小数ではなく素の float32 を使う。量子化を挟まないので、crane の表示値と CM4 の実効値を一致させられる。

## 受信時の検査

受信ポートは、実機の `ai_cmd_v2.out` もシミュレータの `cm4_sim.out` も `--config-port`（既定 12350）。受信、検証、適用、ログ出力はどちらも同じ `config_packet.h` を通るので、シミュレータで確かめた値は実機でも同じように扱われる。

- サイズは 28 バイト、version は `2` でなければならない。サイズ違いは `WrongSize`、version 違いは `UnsupportedVersion` として拒否し、理由をログに出す。
- 次の範囲を検査する。外れていたら**クランプせず、データグラムごと捨てて**拒否理由をログに出す。黙ってクランプすると、crane の表示と実機の実効値が食い違ったまま気付けない。

| フィールド | 範囲 |
|---|---|
| `position_gain`（kp） | `0 <= v <= 20` [1/s] |
| `integral_gain`（ki） | `0 <= v <= 20` [1/s²] |
| `derivative_gain`（kd） | `0 <= v <= 5` [無次元] |
| `deceleration` | `0 <= v <= 20` [m/s²] |
| `position_tolerance` | `0 <= v <= 1.0` [m] |

- 検査は**データグラム単位**で行う。ki だけが範囲外でも、kp を含めて何も適用しない。一部だけ適用すると、crane の表示と実機の実効値が食い違う。
- 拒否したときは現在のゲインを保持する。

> [!IMPORTANT]
> サイズと version は完全一致で検査するので、**crane と CM4 は対応する版を同時に配ること。** `cm4_sim` を使う場合は Docker イメージのタグ（crane の `CM4_SIM_TAG`）も合わせて更新する（[シミュレーション](../pc/simulation.md#dockerイメージ)）。

## 運用上の性質

- 設定が届かなくなっても最後の値を保持する。ゲインは安全のための信号ではないので、届かないからといって既定値に戻すとかえって挙動が飛ぶ。
- crane は同じ値を定期的に送ってよい（現状は最大 1 秒に 1 回）。CM4 は値が変わったときだけログに出す。
- 変更できるのは上表の5項目だけである。停止タイムアウト、ビジョン鮮度閾値、I項速度上限はこのパケットに含めない。
- CM4内の適用処理とsimでの機体別設定の方法は[CM4位置制御](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/position_control.md)と[sim運用](https://github.com/ibis-ssl/Orion_CM4/blob/main/doc/simulation.md)を参照する。
