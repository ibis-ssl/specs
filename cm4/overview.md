# CM4 概要

Orion_CM4 リポジトリのうち、Raspberry Pi CM4（BCM2711、Cortex-A72 ×4）上で動く部分をまとめる。リポジトリは実行場所で分かれている。

- `cm4/`：CM4 上で動く制御 API、UART ブリッジ、位置制御、カメラサーバー、ファームウェア更新ツール、セットアップ資材。生成した実行ファイルは `cm4/bin/` に置く。
- `host/`：ホスト PC で動く Python の CLI / GUI と管理 Web UI（[pc/](../pc/host_tools.md)）。

## 主要ディレクトリ

```text
cm4/
  lancher.py               制御 API（FastAPI）
  build.sh                 C++ バイナリの唯一のビルド定義
  setup.sh                 初回セットアップ
  update.sh                cm4-fleet deploy から呼ばれる更新処理
  control_server.service   systemd ユニット
  Dockerfile               cm4_sim イメージ
  bridge/
    forward_ai_cmd_v2.cpp       指令ブリッジ（ai_cmd_v2.out）
    forward_robot_feedback.cpp  feedback ブリッジ（robot_feedback.out）
    robot_packet.h              64 B 指令のレイアウト
    config_packet.h             位置制御設定パケット
    robot_feedback_packet.h     128 B feedback のレイアウト
    cm4_sim.cpp                 シミュレータ用の CM4 相当プロセス（ホスト PC 専用）
    robot_packet_layout_test.cpp, test_*.py
  control/
    position_controller.h / .cpp, test_position_controller.cpp
  camera/
    cam_server_v3.py, cam_server_v3.spec, default_hsv_config.json
  firmware/
    main_ab_updater.py, sub_can_updater_v2.py, fw_version_reader.py, all_can_updater.py
  bin/                     ビルド成果物（ai_cmd_v2.out, robot_feedback.out, cm4_sim.out, テスト）
  runtime/                 機体固有の設定（cam_server_v3_hsv.json など、Git 管理外）
```

## 制御 API

`cm4/lancher.py` は各 CM4 で動く FastAPI サーバーで、`wlan0` の IP の 8000 番で待ち受ける。

| エンドポイント | 動作 |
|---|---|
| `POST /start` | `cm4/bin/ai_cmd_v2.out`、`cm4/bin/robot_feedback.out`、`cm4/camera/dist/cam_server_v3` を起動する |
| `POST /stop` | 上記のプロセスを `pkill -f` で止める |
| `GET /status` | 制御ブリッジ（`ai_cmd_v2.out`）が動いているかを返す |

`robot_feedback.out` には `-n` で IP の末尾オクテット（`100 + N`）を渡す。`cm4_sim.out` はホスト PC 専用なので、起動対象にも `pkill -f` のパターンにも入れていない。

ホストからの操作は [ホスト PC 側ツール](../pc/host_tools.md#制御ツール) を参照。

## ビルド

`cm4/build.sh` は C++ バイナリの**唯一のビルド定義**である。`setup.sh` と `update.sh` の両方がこれを呼ぶ。sudo も apt も使わないので、ホスト PC（x86_64）でもそのまま動く。最後にテスト一式を実行する。

```bash
./cm4/build.sh                # ビルド + テスト
./cm4/build.sh --no-tests     # ビルドのみ
./cm4/build.sh --targets=sim  # cm4_sim.out に必要なものだけ（cm4/Dockerfile が使う）
```

- `position_controller.cpp` は `.o` を 1 つだけ作り、`ai_cmd_v2.out`、`cm4_sim.out`、テストの 3 か所でリンクする。同じソースであることが実機とシミュレータの挙動が一致する保証なので、オブジェクトも 1 つにしている。
- 5 本の `g++` は互いに独立しているので並列に実行する。
- Dockerfile は `./build.sh --no-tests --targets=sim` を呼ぶだけで、`g++` の行を持たない。そこにビルドを書き直すと、実機用の `setup.sh` / `update.sh` と食い違うバイナリを配ることになる。

## セットアップ

`cm4/setup.sh` は CM4 の初期セットアップ用。インターネットに出られる環境で一度実行する。

- APT パッケージを入れる。
- `pip install -e .` で Python の依存を入れる。
- `cm4/build.sh` を呼び、`cm4/bin/` に出力する。
- `cm4/camera/cam_server_v3.py` を PyInstaller で `cm4/camera/dist/cam_server_v3` にビルドする。
- `cm4/control_server.service` を `/etc/systemd/system/` に置く。

```bash
cd /home/ibis/Orion_CM4
chmod +x cm4/setup.sh
./cm4/setup.sh
```

詳しいオプションと手動ビルドの手順は、Orion_CM4 リポジトリの `SETUP.md` にある。2 台目以降と以後の更新は [フリート管理](../pc/fleet.md) で行う。

## テスト

`./cm4/build.sh` が次を実行する（CI も同じ）。実機の UART も STM32 も要らない。

- `robot_packet_layout_test.out`：crane の正本からのパケットレイアウトのずれを検出する（[制御パケット](../common/control_packet.md#正本と一致検査)）。
- `test_position_controller.out`：位置制御則の単体テスト。
- `test_cm4_sim.py`：cm4_sim の結合スモークテスト。
- `test_forward_ai_cmd_v2.py`：実機ブリッジの結合スモークテスト（`--debug` + pty）。1 件だけ `--debug` なしの実運用モードで動かし、位置制御の状態表示が実際に出ることを検査する。

テストの起動待ちは固定の `sleep` を使わない。`cm4_sim` は最初の出力データグラムを、`ai_cmd_v2` は起動バナーの最終行を待つ。

## 実機で確認すること

- `--passthrough` で、受けた指令をそのまま流せること。
- crane が mode 4 を送ったとき、`ai_cmd_v2.out` の表示に `mode 4` と `tarPos` が出ること。
- `--tx-rate-hz 500` での UART 占有率。G474 の `ORE`/`FE`/`NE`/`PE` と parser timeout のカウンタが増えないことを確認する（[CM4 との UART](../stm/uart_cm4.md#診断カウンタ)）。
- 安全停止のときの惰走距離と、crane を止めてから車輪が止まるまでの時間（[停止判定](../common/safety_stop.md#104-ms-は駆動力が切れるまでで止まるまでではない)）。

## ログ

ログは Git に入れない。`.gitignore` で `*.log` とルートの `runtime/` 全体を除外している。実行ログ、計測 CSV、パケットキャプチャ、一時 BIN はローカルに保存する。
