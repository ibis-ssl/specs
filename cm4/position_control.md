# 位置制御

crane は位置指令（mode 4）を送り、CM4 が位置制御ループを閉じて速度指令（mode 3）を G474 へ渡す。制御則と安全停止の判定は `cm4/control/position_controller.h` / `.cpp` にある。transport に依存しない作りで、実機のブリッジ（`ai_cmd_v2.out`）とシミュレータ用の `cm4_sim.out` が**同じソースを**リンクする。コピーを作らないことが、実機とシミュレータの挙動が一致する唯一の保証である。

CM4 が生成する速度コマンドの定義は Orion と Atlas で共通である。以下の G474 feedback を入力にする構成と、STM 側の定数に合わせた設定は Orion との接続で確認した内容を示す。Atlas のフィードバックと STM 側の制御動作は未記載である。

```text
実機: crane --UDP 65B mode4--> ai_cmd_v2.out [位置ループ] --UART mode3--> G474
                                    ^
                                    | 127.0.0.1:(50100+N) 128B feedback
                               robot_feedback.out <--UART-- G474
```

シミュレータでの構成は [シミュレーション](../pc/simulation.md) を参照。

## 入力

- 目標位置：指令の `TARGET_GLOBAL_POS_X/Y`（byte 32..35）
- 終端速度（フィードフォワード）：mode 4 の `terminal_velocity_x/y`（byte 24..27）
- 速度上限：`LINEAR_VELOCITY_LIMIT`（byte 14..15）と `TERMINAL_VELOCITY`（byte 36..37）
- 現在位置：**G474 feedback の byte 44..51（`vision_based_position_x/y`）**。crane のパケットに入っている `vision_global_pos` ではループを閉じない。それは今回ループの外へ出そうとしている無線経路そのものだからである。
- `position_tolerance` は 65 バイトの指令に載らないので、CM4 側の設定値（既定 0.01 m、`--tolerance`）。

## 既定値

| パラメータ | 既定値 | 変え方 |
|---|---|---|
| `position_gain`（kp） | 2.0 | `--kp`（起動時の初期値）、[設定パケット](../common/config_packet.md) |
| `integral_gain`（ki） | 0 | 設定パケット |
| `derivative_gain`（kd） | 0 | 設定パケット |
| `deceleration` | 3.0 m/s² | `--decel`、設定パケット |
| `position_tolerance` | 0.01 m | `--tolerance`、設定パケット |
| `integral_velocity_limit` | 0.5 m/s | 変更不可 |
| `command_timeout_ms` | 100 | `--command-timeout-ms` |
| `feedback_timeout_ms` | 100 | `--feedback-timeout-ms` |
| `vision_age_limit_ms` | 500 | 変更不可（G474 の定数と一致させる） |

稼働中は crane からの設定パケットで kp / ki / kd / deceleration / position_tolerance を上書きできる。再起動は要らない。

## 制御則

`e` を目標位置と現在位置の偏差、`ff` を終端速度とすると、`computePositionControl()` は次のように計算する。

```text
ff を TERMINAL_VELOCITY でノルム制限（TERMINAL_VELOCITY > 0 のとき）
|e| <= position_tolerance かつ |ff| ≈ 0 なら AtTarget（速度 0）

v = kp·e + ki·∫e dt − kd·v_meas + ff
制動エンベロープ = sqrt(|ff|² + 2·deceleration·|e|)
v = clampNorm(v, min(linear_velocity_limit, 制動エンベロープ))
```

- `v_meas` は feedback の位置を時間微分し、一次遅れフィルタをかけた実測速度。
- 出力は mode 3 の極座標（`target_global_velocity_r = |v|`、`target_global_velocity_theta = atan2(vy, vx)`）にして G474 へ送る。

### PID

- **既定は `ki = kd = 0`**。積分項と微分項は適用しない。
- **状態の持ち主**：積分と微分の状態（`PositionControllerState`）は、呼び出し側がロボットごとに持つ。`cm4_sim` で全機の積分が混ざるのを防ぐためである。
- **feedback を受けたときだけ計算する**：微分と積分は、零次ホールドの feedback が更新されたときだけ計算する。微分は微分先行形（`-kd × 実測速度`）にして、目標が変わったときのキックを防ぐ。
- **アンチワインドアップ**：速度上限（制動エンベロープを含む）に飽和している間は積分を止め、I 項が単独で出せる速度を `integral_velocity_limit` でクランプする。
- **状態のリセット**：安全停止のときや、制御していないとき（素通しモードなど）は状態をリセットする。`AtTarget` に到達したときは、定常偏差を消すために積分を保持する。

### フィードフォワードの上限

`terminal_velocity_x/y` はフィードフォワードとして速度に直接足される。未設定のシグネチャ（次節）は検出できるが、「もっともらしいが間違っている」終端速度は検査では見分けられない。

その場合でも**出力の大きさは `linear_velocity_limit` で頭打ちになる**。`linear_velocity_limit` 自体が未設定なら `max(0, -32.767) = 0` となり、停止側に倒れる。`test_position_controller.cpp` の `testFeedforwardCannotExceedVelocityLimit` で固定している。`linear_velocity_limit = 0` の解釈については [制御パケット](../common/control_packet.md#linear_velocity_limit--0-の扱い) を参照。

### 未設定値の扱い

range 付きの 2 バイトのフィールドはゼロ埋めすると `-32.767` に化ける（[制御パケット](../common/control_packet.md#ゼロ埋めはゼロ値にならない)）。`position_controller` は `|v| >= 32`（と NaN）を「未設定のシグネチャ」として扱う。

- 終端速度が未設定なら 0 とみなし、P 制御を続ける。
- 目標位置か現在位置が未設定なら `InvalidCommand` で止まる。

## 安全停止

停止理由（`StopEmergency`、`CommandStale`、`FeedbackStale`、`VisionUnavailable`、`VisionStale`、`InvalidCommand`）とその順序、止まり方は [停止判定](../common/safety_stop.md#cm4-の安全停止mode-4-の位置制御経路) にまとめている。判定は `position_controller` の中にあるので、実機と `cm4_sim` は必ず同じ判定を通る。

「位置制御が効いていない」ように見えたときは、まず `VisionStale` を疑う。`elapsed_time_ms_since_last_vision` は無線が劣化すると最初に成立する条件である。
