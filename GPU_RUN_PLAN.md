# RTX 5090 実行予定（11/7 夜〜11/8 8:00）

5090 で回すのは、リポジトリが結果ファイルを必要としている次の2つだけです。

| # | 内容 | 作られるファイル | 使う場所 | 目安時間 |
|---|---|---|---|---|
| ① | 予測モデル（ニューラルネット）の学習 | `data/mirror/neural_forecaster.json` | Evidence のパネル F | 10〜20分 |
| ② | RL アドバイザーの学習（PPO） | `data/rl/policy.npz`, `data/rl/policy.json` | Play タブの AI 助言 | 朝まで |

①は短いので先に終わらせ、残りの時間をすべて②に使います（同時に回すと CPU と GPU を取り合って両方遅くなります）。

## タイムライン

| 時刻 | 作業 |
|---|---|
| 23:00 | 準備（取得・PyTorch のインストール・GPU 確認） |
| 23:15 | 動作確認（①②とも短いテスト） |
| 23:20 | ① 開始 |
| 23:40 | ① の結果を push、② 開始（`--hours 7.25`） |
| 7:00ごろ | ② が自動で終了（最終評価に数分） |
| 7:10 | ② の結果を push |
| 〜8:00 | 予備 |

## コマンド

### 準備（初回のみ）

```bash
git clone -b poker-mirror https://github.com/Aikk01/HackTheBurgh.git
cd HackTheBurgh
python -m venv .venv
.venv\Scripts\activate
pip install torch --index-url https://download.pytorch.org/whl/cu128
pip install -r requirements-train.txt
python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

`True NVIDIA GeForce RTX 5090` と出れば OK。RTX 50 シリーズは CUDA 12.8 版（`cu128`）の PyTorch が必須です。

### 動作確認

```bash
python scripts/train_neural_forecaster.py --quick
python -m pokersim rl-train --smoke
```

### ① 予測モデル

```bash
python scripts/train_neural_forecaster.py
git add data/mirror/neural_forecaster.json
git commit -m "Add neural forecaster results"
git push
```

### ② RL アドバイザー

`--hours` は「止めたい時刻 − 開始時刻」。例: 23:40 開始で 7:00 に止めるなら 7.25。

```bash
python -m pokersim rl-train --hours 7.25
```

終わったら:

```bash
git pull
git add data/rl/policy.npz data/rl/policy.json data/rl/runs/*/eval.json data/rl/runs/*/log.csv data/rl/runs/*/final_eval.json
git commit -m "Add trained RL advisor policy"
git push
```

## 寝る前の確認

- Windows の「電源とスリープ」で、電源接続時のスリープを「なし」にする（スリープすると学習が止まる）
- Windows Update を一時停止しておく（夜中の再起動を防ぐ）
- ② を始めて1時間ほどで `EVAL upd 50: mean ... bb/100` が出ていれば順調

## 何か起きたとき

- **② が途中で止まった**: 10分ごとに保存しているので続きから再開できる
  ```bash
  python -m pokersim rl-train --resume data/rl/runs/<日時>/latest.pt --hours <残り時間>
  ```
- **朝までに終わらない**: 過去最高の成績のモデルが常に `data/rl/policy.npz` に保存されているので、Ctrl+C で止めてそのまま push すればよい
- **① が20分を超えた**: いったん止めて ② を先に始める。① は朝の予備時間に回す

詳しい説明: ① は `PROGRESS.md` の「Handoff: GPU run」、② は `TRAINING.md`。
