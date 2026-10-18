# ホスト PC 側ツール

Orion_CM4 リポジトリの `host/` にある、ホスト PC で実行する Python ツールをまとめる。`host/apps/host_lancher.py` と `host/apps/cam_viewer.py` は Windows と Linux の両方で動かす。機体番号 `N` とポートは [システム構成](../common/system_overview.md#機体番号とネットワーク) を参照。

## セットアップ

```powershell
uv sync
```

通常は `pyproject.toml` の entry point から実行します。

## 制御ツール

### `cm4-control`

CM4 側の `cm4/lancher.py` に対する HTTP クライアントです。

```powershell
uv run cm4-control status --ip 192.168.20.103
uv run cm4-control scan
uv run cm4-control start --ip 192.168.20.103
uv run cm4-control stop --ip 192.168.20.103
```

Python module として直接実行する場合:

```powershell
uv run python -m host.apps.cm4_control_cli scan
```

### `host-launcher`

`cm4-control` と同じ処理を使う Qt GUI です。`192.168.20.100` から `192.168.20.112` までの CM4 を監視・操作します。

```powershell
uv run host-launcher
```

## カメラツール

### `cm4-camera`

CM4 側カメラサーバーの HTTP API と multicast 座標を扱う CLI / 共通ライブラリです。

```powershell
uv run cm4-camera config --machine-no 10
uv run cm4-camera get-params --machine-no 10
uv run cm4-camera frame --machine-no 10 --image-name raw --output raw.jpg
uv run cm4-camera params --machine-no 10 --hsv-min 0 100 100 --hsv-max 15 255 255
uv run cm4-camera coords --machine-no 10 --timeout 1.0
uv run cm4-camera roi-calibrate --machine-no 10 --left 90 --top 180 --width 40 --height 40
```

### `cam-viewer`

CM4 側カメラサーバーの raw/mask 画像、座標、HSV 設定を確認する Qt GUI です。詳細は [カメラ](../cm4/camera.md#カメラデバッグ-gui) を参照。

```powershell
uv run cam-viewer --machine-no 10
```

## robot feedback ツール

### `robot-feedback-receiver`

CM4 から送信される robot feedback の UDP multicast を受信し、128 バイトパケット（[フィードバックパケット](../common/feedback_packet.md)）をデコードして標準出力へ出します。同期バイトとチェックサムの検証結果、電圧、姿勢、エラー情報、モーター電流、カメラ座標を出し、`--json` では全フィールドを JSON Lines で出します。GUI や Rerun に依存しないので、通信とパースだけを確かめるときに使います。

```powershell
uv run robot-feedback-receiver --machine-no 3
uv run robot-feedback-receiver --machine-no 3 --max-packets 10
uv run robot-feedback-receiver --machine-no 3 --max-packets 1 --receive-timeout 5
uv run robot-feedback-receiver --machine-no 3 --json
```

### `robot-feedback-viewer`

robot feedback を Qt GUI で時系列表示します。受信とパースは `host/lib/feedback/` に任せ、GUI は現在値と時系列グラフの表示だけを行います。

```powershell
uv run robot-feedback-viewer --machine-no 10
uv run robot-feedback-viewer --machine-no 10 --interface-ip 192.168.20.200
```

### `robot-feedback-rerun`

robot feedback を Rerun に記録・表示します。

```powershell
uv run robot-feedback-rerun --machine-no 3
uv run robot-feedback-rerun --machine-no 3 --max-packets 10
uv run robot-feedback-rerun --machine-no 3 --max-packets 1 --receive-timeout 5
uv run robot-feedback-rerun --machine-no 3 --no-spawn
```

## 指令タイミング診断ツール

### `command-timing-probe`

安全な停止状態の指令（既定は mode 3）を PC から周期送信し、送信時刻を CSV に記録します。CM4 側の受信タイミングと突き合わせる診断に使います。

```powershell
uv run command-timing-probe --host 192.168.20.108 --robot-id 8 --seconds 30 --output .	iming_run1
```

### `analyze-command-timing`

コマンド時刻診断の複数の run を、時計ドメインごとに集計して JSON と短い表を出力します。

## Web 管理 UI

`host/robot-manager/` は、複数台の CM4 をブラウザから操作する管理 Web UI です。Docker で動かします（`docker-compose.yaml` の build context は `./host/robot-manager`）。

```powershell
docker compose up --build
```

## フリート管理ツール

### `cm4-fleet`

複数台の CM4 へ OTA アップデート・SSH 鍵配布・設定配布を一括で行う CLI です。詳細は [フリート管理](fleet.md) を参照してください。

```powershell
uv sync --extra fleet
uv run cm4-fleet bootstrap --all
uv run cm4-fleet deploy --all
uv run cm4-fleet status --all
```

## ファイル配置

- `host/lib/cm4_control_client.py`
- `host/apps/host_lancher.py`
- `host/lib/cm4_camera_client.py`
- `host/apps/cam_viewer.py`
- `host/lib/feedback/packet.py`
- `host/lib/feedback/receiver.py`
- `host/apps/robot_feedback_receiver_cli.py`
- `host/apps/robot_feedback_viewer.py`
- `host/apps/robot_feedback_rerun.py`
- `host/apps/cm4_fleet_cli.py`
- `host/lib/fleet/`
- `host/apps/command_timing_probe.py`
- `host/apps/analyze_command_timing.py`
- `host/robot-manager/`
