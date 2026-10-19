# 転倒率評価スクリプト（eval_fall_rate.py / run_eval_all.sh）

衝突形状の近似（円柱・円錐 → 凸メッシュ）が学習済み方策の歩行能力に与える影響を調べるための評価スクリプト。
「近似あり／なしで学習した方策」を「元の形状の世界／近似した世界」の両方で評価する 2×2 の設計で、
方策そのものの能力が落ちたのか、近似した世界に過剰に適応したのかを区別する。

## ファイル構成

| ファイル | 役割 |
|---|---|
| `eval_fall_rate.py` | 方策を1本・世界を1つ指定して評価し、結果を CSV に1回分追記する |
| `run_eval_all.sh` | `eval_fall_rate.py` を「方策 × 世界」の数だけ順番に呼び出す |

どちらも `/workspace/isaaclab/scripts/reinforcement_learning/skrl/` に置く。
コマンドはすべて `/workspace/isaaclab` で実行する。

## 対象

- タスク: `Isaac-Velocity-Rough-Unitree-Go1-v0`
- 方策: 近似あり・なし × seed 42 / 43 / 44 の6本
- 世界: 元の形状・近似の2つ
- 合計: 6 × 2 = 12 回の評価

## 前提: 学習

評価には最終チェックポイント `agent_180000.pt`（7500 iter × 24 rollouts）を使う。
学習時はチェックポイントの保存間隔を指定しておく。

```bash
# 近似なし
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py --task Isaac-Velocity-Rough-Unitree-Go1-v0 \
  --num_envs 16384 --max_iterations 7500 --headless --seed 42 \
  agent.agent.experiment.checkpoint_interval=12000

# 近似あり（上と同じコマンドの末尾に追加）
  --kit_args "--/physics/collisionApproximateCylinders=true --/physics/collisionApproximateCones=true"
```

Hydra の上書きが反映されない場合は、agent の yaml で `checkpoint_interval` を直接書き換える。

## 使い方

### 1. 評価する方策を登録する

`run_eval_all.sh` の `RUNS` に、各方策のログディレクトリを書く。

```bash
declare -A RUNS=(
  [noapprox_s42]="logs/skrl/unitree_go1_rough/2026-10-06_05-58-42_ppo_torch"
  [noapprox_s43]="logs/skrl/unitree_go1_rough/YYYY-MM-DD_hh-mm-ss_ppo_torch"
  [noapprox_s44]="logs/skrl/unitree_go1_rough/YYYY-MM-DD_hh-mm-ss_ppo_torch"
  [approx_s42]="logs/skrl/unitree_go1_rough/2026-10-06_01-13-17_ppo_torch"
  [approx_s43]="logs/skrl/unitree_go1_rough/YYYY-MM-DD_hh-mm-ss_ppo_torch"
  [approx_s44]="logs/skrl/unitree_go1_rough/YYYY-MM-DD_hh-mm-ss_ppo_torch"
)
```

- 日時が `YYYY` のままの方策と、チェックポイントが見つからない方策は自動で飛ばされる
- iteration 数を変えて学習した場合は、`CKPT_NAME` も合わせて書き換える

### 2. 実行する

```bash
# 本番（16384台）
OUT=logs/eval_fall_rate.csv bash scripts/reinforcement_learning/skrl/run_eval_all.sh

# 動作確認（1024台、数分で終わる）
NUM_ENVS=1024 OUT=logs/eval_test.csv bash scripts/reinforcement_learning/skrl/run_eval_all.sh
```

| 環境変数 | 省略時の値 | 意味 |
|---|---|---|
| `NUM_ENVS` | `16384` | 評価に使うロボットの台数 |
| `OUT` | `eval_fall_rate.csv` | 結果を追記する CSV のパス |

> [!IMPORTANT]
> `OUT` は必ず `logs/` の下にする。`/workspace/isaaclab` 直下はコンテナの外にマウントされていないため、
> `./docker/container.py stop` などでコンテナを作り直すと消える。

### 3. seed を追加する

seed 42 を回し直さずに、同じ CSV に追記する手順。

1. 追加する seed の行の日時を、実際のログディレクトリ名に書き換える
2. 評価済みの seed の行の先頭に `#` を付けてコメントアウトする
3. 前回と同じ `OUT` を指定して実行する

```bash
OUT=logs/eval_fall_rate.csv bash scripts/reinforcement_learning/skrl/run_eval_all.sh
```

## eval_fall_rate.py を単体で実行する

エラーの確認や、1本だけ評価し直したいときに使う。

```bash
# 元の形状の世界
./isaaclab.sh -p scripts/reinforcement_learning/skrl/eval_fall_rate.py \
  --task Isaac-Velocity-Rough-Unitree-Go1-v0 --num_envs 1024 --headless \
  --checkpoint logs/skrl/unitree_go1_rough/<日時>_ppo_torch/checkpoints/agent_180000.pt \
  --label noapprox_s42 --out logs/eval_test.csv
```

近似した世界で評価するときは、末尾に次を追加する。

```bash
  --kit_args "--/physics/collisionApproximateCylinders=true --/physics/collisionApproximateCones=true"
```

| 引数 | 意味 |
|---|---|
| `--checkpoint` | 評価するチェックポイント（`.pt`） |
| `--label` | CSV に書く方策名（例: `approx_s43`） |
| `--out` | 結果を追記する CSV のパス |
| `--kit_args` | 近似フラグ。付けると近似した世界になる |

## run_eval_all.sh の内部動作

- 方策ごとに、`--kit_args` なし（元の形状）と、近似フラグ付き（近似）の2回を実行する
- 方策はラベル名でソートした順に実行する（毎回同じ順番になる）
- `isaaclab.sh` は Python がエラーで落ちても終了コード 0 を返すため、評価後に CSV の行が増えていなければ失敗とみなし、そこで全体を止める

## 評価の中身

- **世界の判定**: どちらの世界で評価したかは、スクリプトが PhysX の設定から自動で判定して `world` 列に書く
- **走破の判定**: 学習時のカリキュラムでレベルが上がる条件と同じ「スタート地点から 4 m 以上離れる」
- **再現性**: 評価用のシードを固定しているので、どの回も同じ地形・同じ初期配置で比較できる
- **行動**: `mean_actions`（確率的なサンプリングをしない平均値）を使うので、評価は決定的になる

## 結果の見方

CSV には、方策・世界・地形の種類・レベルごとに次の値が入る。

- 台数（`n`）
- 走破率・転倒率・時間切れ率
- そのレベルに対応する物理量（段差の高さなど）

失敗が1台でもあった行だけを取り出す。

```bash
awk -F, 'NR==1 || $10<1' logs/eval_fall_rate.csv | cut -d, -f1,3,4,5,9,10,11,12
```

出力される列は「方策・世界・地形・レベル・台数・走破率・転倒率・時間切れ率」。

### 地形についてのメモ

- `pyramid_stairs` は中央が高い（外へ歩くと下り）、`pyramid_stairs_inv` は中央が低い（外へ歩くと上り）
- `random_rough` は難易度のパラメータを使っていないため、レベルごとの差は出ない。図ではレベル全体の平均を1本の値として出す

## skrl のバージョンに関する修正

コンテナ内の skrl では API が変わっているため、`eval_fall_rate.py` は `play_capture.py` の書き方に合わせてある。

| 箇所 | 修正内容 |
|---|---|
| `runner.agent.set_running_mode("eval")` | メソッドが存在しないため削除（コメントアウト） |
| `runner.agent.act(obs, timestep=0, timesteps=0)` | 第2引数に `None` を追加 → `runner.agent.act(obs, None, timestep=0, timesteps=0)` |

skrl を更新して再びエラーが出た場合は、同じフォルダで動いているスクリプトの該当箇所を確認する。

```bash
grep -n "running_mode\|training_mode\|agent.act\|mean_actions" scripts/reinforcement_learning/skrl/play_capture.py
```
