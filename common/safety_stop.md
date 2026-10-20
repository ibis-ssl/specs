# 停止判定

どこが落ちたら誰がどう止めるかを、各層にまたがって定義する。安全停止の閾値と機器間の意味はこの文書を正本とし、制御器内部の状態やテストは各repoで扱う。

CM4 が送る停止コマンドの定義は Orion と Atlas で共通である。この文書の G474 の通信判定、走行停止条件、駆動出力が切れるまでの時間と惰走の説明は、Orion の `G474_Orion_main` で確認した実装に基づく。Atlas の STM 側の停止条件、応答時間、車輪・操舵の停止動作は未記載である。

## 何が落ちたら誰が止めるか

| 何が落ちたか | 止めるのは誰か | 時間の目安 | 止まり方 |
|---|---|---|---|
| crane（無線断・プロセス死）、mode 4 の位置制御中 | CM4 が `STOP_EMERGENCY` を立てる | 約 104 ms | 惰走 |
| crane、mode 3 の素通し中 | G474 の AI 接続タイムアウト（CM4 の送信が止まり `check_counter` が凍るため） | 250 ms | 惰走（`stop_emergency` と停止要求） |
| CM4（`ai_cmd_v2.out` の停止、UART 断） | G474 の AI 接続タイムアウト（`check_counter` が凍るため） | 250 ms | 惰走（`stop_emergency` と停止要求） |
| G474 feedback（mode 4 の位置制御中） | CM4（`FeedbackStale`） | `--feedback-timeout-ms`（既定 100 ms） | 惰走 |
| ビジョン喪失 | CM4（mode 4 の場合）と G474 の両方 | 即時 / 500 ms 超 | 惰走 |

素通し経路でも位置制御経路でも、crane に届く無線が劣化すると、まず `elapsed_time_ms_since_last_vision` の条件が成立する。

## G474 の通信タイムアウト

`Core/Src/ai_comm.c` の `commStateCheck()` が制御周期ごとに判定する。時間はすべて `system_time_ms`（2 ms 周期で 2 ずつ進む ms カウンタ）が基準。

| 判定 | 条件 | タイムアウト時の動作 |
|---|---|---|
| AI 接続（`connected_ai`） | `check_counter` が変化しない時間が **250 ms**（`AI_CMD_TIMEOUT 0.5 × MAIN_LOOP_CYCLE 500`）以上 | `connected_ai = false`。`resetAiCmdData()` で指令を停止側に戻す（`stop_emergency = true`、`is_vision_available = false`、キック・ドリブル・速度制限を 0）。`requestStop(1000)` |
| CM4 接続（`connected_cm4`） | チェックサムの正しい制御指令フレーム（`FWVR` / `FWUP` を除く）の受信が途絶えた時間が **500 ms**（`CM4_CMD_TIMEOUT 1.0 × 500`）以上 | `connected_cm4 = false`、`connected_ai` も false |
| 自己リセット | 一度 AI に接続したあと、`connected_cm4 = false` が **6 s**（3000 周期）続く | `NVIC_SystemReset()`。最後のフレームからおよそ 6.5 s 後になる。`MAIN_MODE_CMD_DEBUG_MODE` では行わない |

自己リセットの内部処理とデバッグはMain repoの文書を参照する。

## G474 の走行停止条件

`MAIN_MODE_FULL_AI_CONTROL` では、`Core/Src/state_func.c` の `maintaskRun()` で次のどれかが成り立つと、`omniStopAll()` で 4 輪の出力を 0 にする。

```c
sys->stop_flag || ai_cmd->stop_emergency || !ai_cmd->is_vision_available
  || ai_cmd->elapsed_time_ms_since_last_vision > 500
```

- `sys->stop_flag`：停止要求（`requestStop()`）、CAN 受信のタイムアウト（100 周期、約 200 ms）、エンコーダやマウスの初期化が終わっていないときなどに立つ。
- `stop_emergency`：指令の停止フラグ。AI 接続がタイムアウトしたときも `resetAiCmdData()` によって true になる。
- ビジョン：`is_vision_available` が偽、または最後に捉えてから 500 ms を超えたとき。500 ms は調整パラメータではなくファームウェアの定数。

`sys->stop_flag` が立っている周期だけ `clearSpeedContrlValue()` が呼ばれ、速度制御の内部状態を実際のオドメトリ速度に合わせる。停止指令やビジョン喪失だけで出力が 0 になった周期には呼ばれない（[Orion 速度制御](../stm/orion_speed_control.md#停止条件)）。

エラーが起きたときはメインモードを `MAIN_MODE_ERROR` にし、モータを止め、CAN でエラーを送り、手動で電源をリセットするまで待つ。

## CM4 の安全停止（mode 4 の位置制御経路）

mode 4 を受けたとき、CM4 の `position_controller` は次のどれかで速度指令を 0 にし、`STOP_EMERGENCY`（byte 22 bit3）を立てる。上から順に判定し、最初に成り立った条件を停止理由（`reason`）にする。

| 条件 | `reason` | 既定 |
|---|---|---|
| crane が `STOP_EMERGENCY` を立てた | `StopEmergency` | — |
| crane からのパケットが途絶えた | `CommandStale` | `--command-timeout-ms 100` |
| G474 feedback が途絶えた（起動直後のまだ受信していない状態を含む） | `FeedbackStale` | `--feedback-timeout-ms 100` |
| crane がビジョンでこのロボットを見失っている（`is_vision_available` = 0） | `VisionUnavailable` | — |
| crane のビジョンがこのロボットを最後に捉えてから時間が経ちすぎた | `VisionStale` | 500 ms（G474 の定数と同じ。500 は動き、501 で止まる） |
| 目標位置・現在位置が物理的にありえない値 | `InvalidCommand` | — |

安全停止のときは `KICK_POWER` / `DRIBBLE_POWER` / `ENABLE_CHIP` も落とす。古いキック指令を撃ち続けないためである。判定は `position_controller` の中にあるので、実機のバイナリと `cm4_sim` が同じ経路を通る。詳細は [位置制御](../cm4/position_control.md) を参照。

### ビジョンの 2 条件を CM4 でも見る理由

crane が見失っている間の `target_global_pos` は「見えていないロボット」についての推測値なので、そこへ向かって走らせてはいけない。G474 は同じ 2 条件で止まるので、CM4 も同じ判定をしないと「G474 なら止まる状況で CM4 だけが走らせ続ける」ことになる。

`vision_age_limit_ms`（500 ms）に CLI オプションを用意していないのは意図的である。現地で違う値を設定できると「CM4 は走らせているのに G474 は止めている」状態を作れてしまう。CM4 は G474 と同じ 2 バイトを同じ `uint16_t` として読み、巻き戻りの補正などはしない。実機と同じ判定になることのほうが重要だからである。

crane は `elapsed_time_ms_since_last_vision` を 0〜65535 に飽和させてから入れ、world model からロボットを引けない例外時には 65535 を入れる（fail-closed）。それでも、`is_vision_available` が主な防壁で、`VisionStale` は無線劣化を早く捉える補助として両方を見る。

## crane が落ちてから車輪の駆動力が切れるまで

mode 4 の位置制御経路では、crane が沈黙しても CM4 は `check_counter` を進めながら送信を続ける（停止中も `--tx-rate-hz` で送る）。したがって G474 の `connected_ai` は真のままで、車輪が止まる理由は CM4 が立てた `STOP_EMERGENCY` である。250 ms の AI 接続タイムアウトはこの経路では効かない。

`--command-timeout-ms`（既定 100 ms）に次が加わる。

| 要素 | 時間 |
|---|---|
| 1 kHz ポーリングの検出遅れ | 最大 1 ms |
| UART 72 バイト @ 1 Mbps | 0.72 ms |
| G474 のメインループ 500 Hz | 最大 2 ms |

**合計でおよそ 104 ms**。停止指令は `--tx-rate-hz` のゲートを待たない（停止理由が変わった周期は、レートに関係なくすぐ送る）。実機で測るときは「100 ms ちょうど」ではなく、この予算と照らし合わせる。

`--passthrough` では `check_counter` が crane から来た値なので、crane が落ちると G474 の 250 ms 判定で止まる。

## 104 ms は「駆動力が切れるまで」で、「止まるまで」ではない

**安全停止では実機も惰走する。** G474 の停止分岐は `omniStopAll()` を呼び、4 輪のモータ出力を 0 にする。Main からモータ基板へ送る停止指令にはブレーキフラグがない。出力 0 で空転するか短絡制動になるかはモータ基板のファームウェアが決めるので、Main 側からは確定できない（実機で測る項目）。

同じ「止まれ」でも 2 通りある。

| CM4 が送るもの | G474 が通る経路 | 挙動 |
|---|---|---|
| mode 3 で `r = 0`、`STOP_EMERGENCY` なし | `speedControl` → `omniMoveIndiv` | 車輪の制御が効く（能動制動） |
| `STOP_EMERGENCY` あり | `omniStopAll` | 駆動力ゼロ（惰走） |

CM4 の安全停止はすべて `STOP_EMERGENCY` を立てるので、crane 断・feedback 断・ビジョン断はすべて下の段（惰走）になる。約 104 ms で駆動力が切れ、そこからは慣性で転がる。惰走距離は能動制動の数倍になる。

実機で「crane を止めてから車輪が止まるまで」を測るときは、**駆動力が切れる時刻（約 104 ms、予算と照らし合わせる対象）と、機体が静止する時刻（惰走距離のぶん後）**を分けて記録する。測るときはロボットが壁や他の機体に当たらない向きにし、開始速度が実際に出ていることを確かめる。塞がれた向きでも「それらしい」値が出るので、数値だけでは異常と分からない。

シミュレータとの違いは [シミュレーション](../pc/simulation.md#実機との停止動作の差) を参照。
