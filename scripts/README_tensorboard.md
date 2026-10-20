# 学習ログの確認（TensorBoard）

近似あり・なし × seed 42 / 43 / 44 の6本の学習ログを、TensorBoard で方策名を付けて並べて見るための手順。
学習の途中で方策が崩れていないか（報酬の急落など）を確認する目的で使う。

## ファイル構成

| ファイル | 役割 |
|---|---|
| `collect_tb.sh` | TensorBoard のログ（イベントファイル）だけを、方策名のフォルダにコピーする |
| `tb_summary.py` | 6本の学習ログを、報酬のピーク・崩れた時点などの数値で要約する |

どちらも `/workspace/isaaclab/scripts/reinforcement_learning/skrl/` に置く。

## 対象の学習

| 方策名 | ログディレクトリ（`logs/skrl/unitree_go1_rough/` の下） |
|---|---|
| `approx_s42` | `2026-10-06_01-13-17_ppo_torch` |
| `approx_s43` | `2026-10-08_08-33-54_ppo_torch` |
| `approx_s44` | `2026-10-12_11-55-06_ppo_torch` |
| `noapprox_s42` | `2026-10-06_05-58-42_ppo_torch` |
| `noapprox_s43` | `2026-10-09_02-12-58_ppo_torch` |
| `noapprox_s44` | `2026-10-13_01-26-53_ppo_torch` |

> [!IMPORTANT]
> 元の `*_ppo_torch` フォルダの名前は変えない。名前を変えるのはコピーだけにする。
> - `run_eval_all.sh` や評価の README が、日時の名前でログを参照している
> - `play.py` で `--checkpoint` を省略したときの「最新の学習」は、日時の名前の順番で選ばれる

## 手順

### 1. ログを方策名のフォルダにまとめて、コンテナの外に出す

コンテナの外（ホスト側）の Isaac Lab のフォルダで実行する。

```bash
docker exec -w /workspace/isaaclab isaac-lab-base bash scripts/reinforcement_learning/skrl/collect_tb.sh
mkdir -p ../tensorboard/animal
docker cp isaac-lab-base:/workspace/isaaclab/logs/tb_runs/. ../tensorboard/animal
```

| 行 | やっていること |
|---|---|
| 1行目 | コンテナに入らずに `collect_tb.sh` を実行し、コンテナ内の `logs/tb_runs/<方策名>/` にイベントファイルをコピーする |
| 2行目 | コピー先のフォルダを作る（親フォルダがないと `docker cp` がエラーになる） |
| 3行目 | `tb_runs` の中身を `../tensorboard/animal` にコピーする |

- `isaac-lab-base` はコンテナ名。違う場合は `docker ps` で確認する
- コピーするのはイベントファイルだけなので、6本で数MB程度。チェックポイントや動画は含まない
- 何度実行しても上書きコピーされるだけなので、学習中のログを最新にしたいときも同じ手順でよい

できあがり:

```
../tensorboard/animal/
├── approx_s42/
├── approx_s43/
├── approx_s44/
├── noapprox_s42/
├── noapprox_s43/
└── noapprox_s44/
```

> [!WARNING]
> - `docker cp` のコピー元の末尾の `/.` を忘れると、2回目以降に `animal/tb_runs/...` という入れ子ができる
> - コンテナの中で `cp` を使って `/workspace/isaaclab` の外（`/workspace/tensorboard` など）にコピーしても、ホストからは見えず、コンテナを作り直すと消える。外に出すのは必ず `docker cp` で行う

### 2. TensorBoard を開く

```bash
tensorboard --logdir ../tensorboard/animal
```

左の一覧に `approx_s42` 〜 `noapprox_s44` の名前で表示される。

ヒューマノイドやアームのログも `../tensorboard/humanoid` などに置いて `--logdir ../tensorboard` で開くと、
`animal/approx_s42` のようにタスク名付きで並ぶ。

## 画面の見方

| 操作 | 内容 |
|---|---|
| 左上のタグの検索欄 | `Total reward\|Total timesteps\|terrain_levels\|base_contact\|Standard deviation\|Value loss` と入れると、見たいグラフだけに絞れる（正規表現） |
| 方策の一覧の検索欄 | `noapprox` で近似なしだけ、`_s43` で seed 43 の2本だけに絞れる |
| Smoothing | 0〜0.3 くらいに下げると、急な落ち込みがならされずに見える |
| 横軸の Step | timestep なので、24 で割ると iteration になる（180000 = 7500 iteration） |

### 主なグラフ

| タグに含まれる文字列 | 意味 |
|---|---|
| `Total reward (mean)` | エピソードあたりの報酬の平均 |
| `Total timesteps (mean)` | エピソード長の平均（短いほど早く転倒・終了している） |
| `terrain_levels` | カリキュラムで到達している地形レベルの平均 |
| `base_contact` | 胴体の接地（転倒）で終了したエピソードの割合 |
| `Standard deviation` | 方策の行動のばらつき（探索の大きさ） |
| `Value loss` | 価値関数の損失 |

### 見るポイント

- **方策が崩れた時点**: 報酬とエピソード長が急に落ちたところ
  - 同じころに `Value loss` が跳ね上がっている → シミュレーションの異常値（接触計算の破綻など）が学習を壊した可能性
  - 同じころに `Standard deviation` が大きくなっている → 探索が暴走した可能性
- **一時的な落ち込み**: 最終的に回復していても、途中で崩れかけていたなら「学習が安定していた」とは書けない
- **近似あり・なしのエピソード長の差**: seed 42 で見えた「近似ありのほうがエピソードが約12%短い」差が、ほかのシードでも出ているか

### チェックポイントとの対応

`best_agent.pt` は、報酬の最高記録が出たあとの最初のチェックポイント保存で一緒に書かれる。
保存された順に並べると、報酬がピークだった時点がわかる。

```bash
ls -ltr --time-style=+%m-%d_%H:%M logs/skrl/unitree_go1_rough/<日時>_ppo_torch/checkpoints/
```

`best_agent.pt` のすぐ上にある `agent_*.pt` までの 500 iteration の間に、報酬がピークだった
（`agent_*.pt` の番号も timestep なので、24 で割ると iteration）。

## 数値で要約する（tb_summary.py）

グラフを見る代わりに、数値の表で6本を並べて比べたいときに使う。コンテナ内の `/workspace/isaaclab` で実行する。

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/tb_summary.py
```

出力は2つ。

**要約表**（方策ごとに1行）

| 列 | 意味 |
|---|---|
| `peak` / `@iter` | 報酬のピークと、その iteration |
| `final` | 最終時点の報酬 |
| `final/peak` | 最終値 ÷ ピーク。1 から大きく下がっていたら崩れている |
| `<50%@iter` | ピークのあとで、報酬がピークの半分を初めて下回った iteration（崩れた時点）。崩れていなければ `-` |
| `ep_len` / `terrain` / `std` | 最終時点のエピソード長・地形レベル・行動のばらつき |
| `max v_loss` / `@iter` | Value loss の最大値と、その iteration |

**推移表**（指標ごとに1つ）

報酬・エピソード長・地形レベル・転倒・行動のばらつき・Value loss を、500 iteration ごとに6本横並びで表示する。

- タグが見つからない方策は、記録されているタグの一覧を表示して飛ばす
- ログディレクトリがない方策は飛ばす

## 学習を追加したとき

seed 45 などを学習したら、次の3つのファイルの `RUNS` に同じ対応（方策名 → ログディレクトリ）を1行足す。

- `collect_tb.sh`
- `tb_summary.py`
- `run_eval_all.sh`

そのあと「手順 1」をもう一度実行する。

## 付録: コンテナ内で直接 TensorBoard を起動する

ログを外に出さずに見る方法。コンテナ内の `/workspace/isaaclab` で起動し、動かしっぱなしにする（tmux の別ウィンドウなどで）。

```bash
D=logs/skrl/unitree_go1_rough
./isaaclab.sh -p -m tensorboard.main --port 6006 --logdir_spec \
approx_s42:$D/2026-10-06_01-13-17_ppo_torch,approx_s43:$D/2026-10-08_08-33-54_ppo_torch,approx_s44:$D/2026-10-12_11-55-06_ppo_torch,noapprox_s42:$D/2026-10-06_05-58-42_ppo_torch,noapprox_s43:$D/2026-10-09_02-12-58_ppo_torch,noapprox_s44:$D/2026-10-13_01-26-53_ppo_torch
```

手元の Windows の PowerShell でサーバーへのトンネルを作り、ブラウザで `http://localhost:6006` を開く。

```powershell
ssh -N -L 6006:localhost:6006 <ユーザー名>@<サーバーのアドレス>
```

サーバーを共有していて 6006 番が使用中なら、両方のコマンドの `6006` を `6007` などに変える。
