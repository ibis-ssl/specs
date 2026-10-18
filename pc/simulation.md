# シミュレーション

ホスト PC で crane とシミュレータ（framework の `simulator-cli`）をつなぐときの構成をまとめる。CM4 の位置制御は `cm4_sim.out` が担当し、実機と同じソース（`cm4/control/position_controller.cpp`）をリンクする（[位置制御](../cm4/position_control.md)）。`simulator-cli` は G474 とロボットの物理を担当し、位置制御は行わない。

```text
crane --UDP:12345 65B mode4--> cm4_sim.out --UDP:12346 715B mode3--> simulator-cli
                                    ^                                    |
                                    +------ UDP 127.0.0.1:(50100+id) ----+
                               cm4_sim.out --multicast 224.5.20.(100+id):(50100+id)--> crane
```

- 機体ごとの 65 バイトの指令は、すべて同じ `cm4_sim.out` の受信アドレスへ送る。`cm4_sim.out` は先頭の ID で振り分ける。
- `cm4_sim.out` から `simulator-cli` への出力は、11 台分（65 B × 11）を含む 715 バイト固定。

> [!WARNING]
> crane の送信器はまだ 715 B を送っているので、現行の `cm4_sim.out` とはそのままではつながらない。[crane との連携](crane_integration.md#制御指令の転送方式) を参照。

## 起動

起動の順序は **simulator-cli → cm4_sim → crane**。

```bash
# 端末1: simulator-cli（framework、ibis ブランチ）
/home/hans/workspace/framework/build/bin/simulator-cli \
  -g 2020B --realism None --localhost \
  --ibis-port 12346 --ibis-feedback-port-base 50100

# 端末2: cm4_sim
./cm4/bin/cm4_sim.out --robot-ids 0 \
  --in-port 12345 --out-port 12346 --feedback-port-base 50100

# 端末3: crane を sim:=true で起動
```

`cm4_sim.out` はホスト PC 専用で、実機では動かさない。

### ポート

| 用途 | アドレス:ポート |
|---|---|
| crane からの機体別 mode 4（65 B） | bind `0.0.0.0:12345`（`--in-port`） |
| simulator-cli への mode 3（715 B） | `127.0.0.1:12346`（`--out-port`。simulator-cli の `--ibis-port` と揃える） |
| simulator-cli からの feedback | bind `127.0.0.1:(50100 + id)`（`--feedback-port-base`。simulator-cli の `--ibis-feedback-port-base` と揃える） |
| crane からの設定パケット | bind `0.0.0.0:12350`（`--config-port`） |
| feedback の再配信 | `224.5.20.(100 + id):(50100 + id)`（実機と同じ。`--feedback-relay-port-base`、既定 50100） |

### 注意点

- **feedback の受け口は 2 つに分かれている。** `cm4_sim` は simulator-cli からの feedback を `127.0.0.1:(50100 + id)` のユニキャストで受け、crane は再配信された multicast を受ける。crane の受信ソケットは `SO_REUSEPORT` 付きなので、crane が同じユニキャストを bind すると、単一送信元からの feedback はどちらか一方が全部取ってしまう（4-tuple ハッシュで振り分けられるため）。crane は常に multicast で受けるので、この取り合いは起きない。
- 再配信ポートの基準（`--feedback-relay-port-base`）は入力ポートとは独立している。crane の受信側の既定が `50100 + id` なので、再配信側は 50100 のままにする。
- **feedback の再配信の送出 IF は既定でループバックに固定している**（`--multicast-if`、既定 `127.0.0.1`）。省略して `IP_MULTICAST_IF` を設定しないと、OS が既定ルートの IF（開発 PC では Wi-Fi になりうる）を選ぶ。crane には multicast が Wi-Fi へ漏れて AP が過負荷になる問題があり、iptables の DROP は `224.5.23.0/24`（vision/referee）だけで、`224.5.20.0/24`（feedback）は対象外である。実ネットワークへ出したいときだけ `--multicast-if <ip>` で明示する。
- **出力パケットの `VISION_GLOBAL_X/Y` には feedback から得た実位置を入れる**（`--vision-echo feedback`、既定）。simulator-cli はコマンドの `vision_global_pos` を実位置と 0.5 m 以内（`IBIS_POSITION_MATCH_THRESHOLD`）で照合してチームを判定し、外れたコマンドを**黙って捨てる**ためである。パケットは届き続け `check_counter` も進むので、「ロボットだけが動かない」という分かりにくい症状になる。理由は 2 つあり、どちらか一方だけでも成立する。
  1. 劣化注入で crane から来た値が古くなる。`--rx-delay-ms 200` 程度で 3 m/s なら 0.6 m ずれ、全コマンドが捨てられる。
  2. 劣化注入がなくても、crane の world model の推定が 0.5 m ずれれば同じことが起きる。

  実機では crane から来た値をそのまま流す（[ブリッジ](../cm4/bridge.md#位置制御経路の出力)）。素通し経路（mode 3）でもシミュレータでは同じエコーがかかる。この破棄は `command dropped` の警告で確認できる。**起動直後に数行出てその後止まるのは正常**で、feedback を受けるまでは crane から来た値をそのまま流すブートストラップ期間だからである。出続ける場合は feedback の経路がつながっていない。
- `simulator-cli` のログに `POSITION_TARGET` の警告が出たら、mode の変換に失敗している。デバッグの最初の手掛かりにする。
- ワイヤ上のバイト列を直接見たいときは、framework の `data/scripts/ibis-packet-tap.py` を経路に挟む。crane → cm4_sim → simulator-cli のどの区間で何が化けたかを、こちら側にデバッグ表示を足さずに切り分けられる。
- 劣化注入（`--rx-delay-ms` / `--rx-jitter-ms` / `--rx-loss-rate` / `--seed`）は **crane → CM4 の入力側にだけ**かかる。mode 3 の素通しでも mode 4 の位置制御でも同じようにかかる。無線経路の劣化に対する振る舞いを見るときに使う。
- `cm4_sim` は 11 台を 1 プロセスで代行するので、[設定パケット](../common/config_packet.md#運用上の性質) はプロセス全体に適用される。

## Docker イメージ

イメージは `ghcr.io/ibis-ssl/orion-cm4-sim`。定義は Orion_CM4 の `cm4/Dockerfile`、push は `.github/workflows/cm4-sim-docker.yml`（main への push と `workflow_dispatch`。pull request ではビルドだけで push しない）。イメージに入るのは `cm4_sim.out` 1 本だけ（[CM4 概要](../cm4/overview.md#ビルド)）。

crane は `docker/scenario/docker-compose.yaml`（プロファイルなし）と `docker/dev/docker-compose.yaml`（プロファイル `sim-erforce`）でこのイメージを使う。イメージは commit SHA（`CM4_SIM_TAG`）で固定している。更新するときは `CM4_SIM_TAG` で試してから既定値を上げる。

compose との契約は次の 3 つで、崩すと一括起動が壊れる。

| 契約 | 実装 |
|---|---|
| `entrypoint: ["tini", "--"]` | runtime ステージで `tini` を入れている |
| `command: cm4_sim ...` | `bin/cm4_sim.out` を **`/usr/local/bin/cm4_sim`** として置く（名前が違う） |
| `network_mode: host` | 前提。独立した netns に置くと、127.0.0.1 上の simulator-cli と crane に届かない |

イメージの検証は、`simulator-cli` と crane を実際につないで行う。ローカルでビルドしたバイナリと同じ挙動になることを確かめる。配布するときは、認証情報なしで pull と起動ができることを確かめる。

## シミュレータと実機の差

ゲインを詰めるときは次の差に注意する。

- シミュレータ側の G474 相当は **125 Hz** で、実機の G474（500 Hz）より粗い。
- simulator-cli は `IS_VISION_AVAILABLE` を見ない。CM4 側で止めているので実害はないが、素通し経路では下流が止めないままになる。
- `cm4_sim` の素通し経路は、crane が落ちると `--command-timeout-ms`（既定 100 ms）で止まる。実機で同じ状況を止めるのは G474 の AI 接続タイムアウト（250 ms、[停止判定](../common/safety_stop.md#g474-の通信タイムアウト)）なので、素通し経路の停止は実機より速い。
- シミュレータの `SimRobot` も、最後の指令から一定時間で standby に入り、車輪の制御に達する前に return する。したがって能動制動と惰走の作り分けは実機と一致する。ただし standby までの猶予は実機の AI 接続タイムアウトより短いので、**指令が途絶えたときの惰走距離はシミュレータのほうが短く出る**。これが効くのは CM4 ごと落ちて G474 への送信が止まった場合だけで、mode 4 の位置制御では CM4 が送り続けるので、crane 断・feedback 断・ビジョン断はすべて `STOP_EMERGENCY` の経路に入る。
- mode 4 がシミュレータまで届いた場合の停止だけは能動制動のまま残している。この分岐は実機に対応するものがなく、CM4 が mode の変換に失敗しているという合図なので、惰走させると発見が遅れるという判断である。
- 実機の惰走距離は測っていない。duty 0 が空転か短絡制動かはモータ基板のファームウェアが決めるので、シミュレータの値は比較対象であって実機の予測値ではない。停止距離をシミュレータで測る手順は framework の `data/scripts/ibis-stop-distance.py` にある。

### feedback の違い

simulator-cli も実機と同じ 128 バイト形式を出し、`host/lib/feedback/packet.py` で復号して `robot-feedback-viewer` などで表示できる。ただし次が違う。

- **byte 2**：実機はチェックサムを入れるが、simulator-cli は定数 10 を入れる。`is_checksum_valid()` はシミュレータの出力では不一致になる。
- **byte 3**：実機では指令の `check_counter` を反射するが、シミュレータの `IbisFeedbackAdaptor` は指令を見ないので自走カウンタを送る。陳腐化の検出には使えるが、特定の指令とは対応しない。
- **byte 52..59（速度）**：`RadioResponse` から作られるキャッシュで、`RadioResponse` は指令が届いたときにしか作られない。指令が落ちている間（経路劣化によるロスや、0.5 m の照合で捨てられたとき）は、位置（byte 44/48、毎周期ビジョンから）は新しいのに、速度（byte 52/56）は捨てられる直前の値で止まる。ロボットが実際に止まったあとも、止まる前の速度を返し続ける。`cm4_sim` も `ai_cmd_v2.out` も byte 44..51 しか使わないので制御には影響せず、騙されるのは速度を見る診断だけである。速度フィールドを判定に使うツールは、値が更新されていることを確かめる。`STOP_EMERGENCY` による安全停止では指令は届き続けるので、速度は正しく 0 まで減衰する。
