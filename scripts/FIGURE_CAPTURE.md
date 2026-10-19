# タスク概要図（図1）の撮影手順

Isaac Lab の学習済み方策を動かして動画を撮り、フレームを切り出して予稿の図にするまでの手順。
対象は次の3タスク（いずれも skrl で学習）。

| 図 | タスク | `--task`（Play 用） | ログフォルダ |
|---|---|---|---|
| (a) ヒューマノイド | Unitree G1 不整地歩行 | `Isaac-Velocity-Rough-G1-Play-v0` | `logs/skrl/g1_rough/` |
| (b) 四足歩行 | Unitree Go1 不整地歩行 | `Isaac-Velocity-Rough-Unitree-Go1-Play-v0` | `logs/skrl/unitree_go1_rough/` |
| (c) アーム | Franka 引き出し開け | `Isaac-Open-Drawer-Franka-Play-v0` | `logs/skrl/franka_open_drawer/` |

全体の流れ：

1. コンテナ内で撮影用スクリプト（`play_capture.py` / `play_capture_franka.py`）を用意する
2. 学習済みチェックポイントを指定して動画を撮る
3. 動画をコンテナ → サーバ → ローカルへコピーする
4. ローカルで ffmpeg を使ってフレームを切り出し、3枚の縦横比をそろえて切り抜く
5. LaTeX に貼る

---

## 0. 前提

- Isaac Lab は Docker コンテナ `isaac-lab-base` 上で動かしている
- 学習は `train.py` で実施済み。チェックポイントは `logs/skrl/<実験名>/<日時>_ppo_torch/checkpoints/` にある
- `scripts/` と `logs/` はホストにマウントされているので、コンテナを削除しても消えない

```bash
# マウントされている場所の確認（ホスト側で実行）
docker inspect isaac-lab-base --format '{{range .Mounts}}{{.Destination}}{{"\n"}}{{end}}'
# → /workspace/isaaclab/scripts と /workspace/isaaclab/logs があれば OK
```

> **注意：rsl_rl と skrl は別物。** 学習に使ったライブラリの `play.py` を使う必要がある。
> ログが `logs/skrl/` にあるなら `scripts/reinforcement_learning/skrl/` 側を使う。

---

## 1. 撮影用スクリプトを用意する

### 1-1. 歩行タスク用（G1・Go1 共通）: `play_capture.py`

```bash
cd /workspace/isaaclab/scripts/reinforcement_learning/skrl
cp play.py play_capture.py
```

**skrl のバージョン差の修正**（新しい skrl では `act()` の引数が1つ増えているため）：

```bash
sed -i 's/runner.agent.act(obs, timestep=0, timesteps=0)/runner.agent.act(obs, None, timestep=0, timesteps=0)/' play_capture.py
grep -n "agent.act" play_capture.py   # act(obs, None, ... になっていれば OK
```

`gym.make(` の行のすぐ上に、同じインデントで次を追加する：

```python
    # ---- 撮影用設定 ここから ----
    env_cfg.viewer.resolution = (1920, 1080)
    env_cfg.viewer.origin_type = "asset_root"   # ロボットを追従
    env_cfg.viewer.asset_name = "robot"
    env_cfg.viewer.env_index = 0
    env_cfg.viewer.eye = (1.5, 1.5, 1.3)        # 見下ろす角度（地平線の平らな縁が映りにくい）
    env_cfg.viewer.lookat = (0.0, 0.0, 0.0)
    env_cfg.commands.base_velocity.debug_vis = False   # 速度コマンドの矢印を消す
    # 可視化用：難しめの箱型地形のみにする
    env_cfg.scene.terrain.terrain_generator.difficulty_range = (0.6, 1.0)
    tg = env_cfg.scene.terrain.terrain_generator
    tg.sub_terrains = {"boxes": tg.sub_terrains["boxes"]}
    # 可視化用：地形を広くする（Play 設定の 5x5 → 10x10 区画）
    tg.num_rows = 10
    tg.num_cols = 10
    # ---- 撮影用設定 ここまで ----
```

### 1-2. アーム用: `play_capture_franka.py`

歩行タスク用の設定（速度コマンド・地形）はアームのタスクに存在しないためエラーになる。別ファイルにする。

```bash
cd /workspace/isaaclab/scripts/reinforcement_learning/skrl
cp play.py play_capture_franka.py
sed -i 's/runner.agent.act(obs, timestep=0, timesteps=0)/runner.agent.act(obs, None, timestep=0, timesteps=0)/' play_capture_franka.py
```

`gym.make(` の行のすぐ上に追加：

```python
    # ---- 撮影用設定 ここから ----
    env_cfg.viewer.resolution = (1920, 1080)
    env_cfg.viewer.origin_type = "env"          # アームは固定なので環境原点を基準にする
    env_cfg.viewer.env_index = 0
    env_cfg.viewer.eye = (-0.6, -1.4, 1.2)      # アームの斜め後ろ・少し上から
    env_cfg.viewer.lookat = (0.5, 0.0, 0.4)     # 引き出しの取っ手あたり
    # 座標軸の矢印（フレームマーカー）を消す
    for name in ["ee_frame", "cabinet_frame"]:
        if hasattr(env_cfg.scene, name):
            getattr(env_cfg.scene, name).debug_vis = False
    # ---- 撮影用設定 ここまで ----
```

> キャビネットの裏側から撮ってしまうとアームが隠れる。引き出しはアーム側（-x 方向）を向いているので、カメラは x が負の側に置く。

### 1-3. バックアップ

上書き事故に備えて、完成したらコピーを取っておく。

```bash
cp play_capture.py play_capture.py.bak
cp play_capture_franka.py play_capture_franka.py.bak
```

---

## 2. 動画を撮る

コンテナ内（`/workspace/isaaclab`）で実行。`--checkpoint` を指定しないと最新のものが自動で読まれるが、明示したほうが確実。

```bash
# (a) G1
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play_capture.py \
  --task Isaac-Velocity-Rough-G1-Play-v0 \
  --num_envs 4 --headless --enable_cameras \
  --video --video_length 300 \
  --checkpoint logs/skrl/g1_rough/2026-10-05_05-37-25_ppo_torch/checkpoints/best_agent.pt

# (b) Go1
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play_capture.py \
  --task Isaac-Velocity-Rough-Unitree-Go1-Play-v0 \
  --num_envs 4 --headless --enable_cameras \
  --video --video_length 300 \
  --checkpoint logs/skrl/unitree_go1_rough/2026-10-01_01-55-27_ppo_torch/checkpoints/best_agent.pt

# (c) Franka
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play_capture_franka.py \
  --task Isaac-Open-Drawer-Franka-Play-v0 \
  --num_envs 4 --headless --enable_cameras \
  --video --video_length 300 \
  --checkpoint logs/skrl/franka_open_drawer/2026-10-07_05-18-41_ppo_torch/checkpoints/best_agent.pt
```

- `best_agent.pt` は学習中に報酬が最大だった時点のモデル。最終時点を使うなら `agent_<最大ステップ>.pt`
- 動画は `<チェックポイントの日時フォルダ>/videos/play/rl-video-step-0.mp4` に保存される（毎回同名で上書き）
- 300 ステップ ≒ 6 秒（1920×1080, 50fps）

---

## 3. 動画をローカルへ持ってくる

### コンテナ → サーバ（ホスト側で実行）

```bash
docker cp isaac-lab-base:/workspace/isaaclab/logs/skrl/g1_rough/2026-10-05_05-37-25_ppo_torch/videos/play/rl-video-step-0.mp4 ~/g1_rough.mp4
docker cp isaac-lab-base:/workspace/isaaclab/logs/skrl/unitree_go1_rough/2026-10-01_01-55-27_ppo_torch/videos/play/rl-video-step-0.mp4 ~/go1_rough.mp4
docker cp isaac-lab-base:/workspace/isaaclab/logs/skrl/franka_open_drawer/2026-10-07_05-18-41_ppo_torch/videos/play/rl-video-step-0.mp4 ~/franka_drawer.mp4
```

まとめてコピーする場合は `./docker/container.py copy`（`docker/artifacts/logs/` 以下にコピーされる）。

### サーバ → ローカル

いつもの方法（scp など）で上記の mp4 を持ってくる。

---

## 4. フレームを切り出して整える（ローカル / Windows PowerShell）

### ffmpeg のインストール

```powershell
winget install Gyan.FFmpeg
# インストール後、PowerShell を開き直して確認
ffmpeg -version
```

> PowerShell では行末の `\` による改行が使えないので、コマンドは1行で書く。

### 4-1. 候補フレームを書き出す

```powershell
# 1枚だけ（-ss は秒、小数も可）
ffmpeg -ss 00:00:01.5 -i g1_rough.mp4 -frames:v 1 -update 1 -q:v 2 g1_1.5s.jpg

# 区間内を一定間隔で全部書き出す（例：4〜5秒を 0.04 秒刻みで25枚）
mkdir frames
ffmpeg -ss 4 -t 1 -i g1_rough.mp4 -vf fps=25 -q:v 2 frames\g_%02d.jpg
```

フレーム選びの目安：

- **G1 / Go1**：足が箱の上に乗っている・段差をまたいでいる瞬間。平らな所に立っている瞬間や転びそうな瞬間は避ける
- **Franka**：グリッパーが取っ手に届いて、引き出しがまだ閉じている瞬間（引き出しは最初の 1〜2 秒で一気に開く）

今回採用したフレーム：G1 = `g_13.jpg`、Go1 = `g_10.jpg`、Franka = `f_12.jpg`

### 4-2. 3枚の縦横比・ロボットの大きさをそろえる

3枚とも 16:10 で切り抜き、1280×800 に揃える。`crop=幅:高さ:左端x:上端y`（元画像は 1920×1080）。

```powershell
ffmpeg -i g_13.jpg -vf "crop=1152:720:354:360,scale=1280:800" -q:v 2 humanoid_g1.jpg
ffmpeg -i g_10.jpg -vf "crop=960:600:465:460,scale=1280:800" -q:v 2 quadruped_go1.jpg
ffmpeg -i f_12.jpg -vf "crop=1536:960:170:110,scale=1280:800" -q:v 2 arm_franka.jpg
```

撮り直してロボットの位置が変わった場合は、フレームを開いてロボット中心の座標 (cx, cy) を確認し、
`x = cx - 幅/2`, `y = cy - 高さ/2` で左上を決め直す。ロボットを大きく見せたいときは比率を保ったまま幅・高さを小さくする（例：960:600 → 800:500）。

> JPG が Acrobat で開かれて PDF 化を勧められる場合は、右クリック →「プログラムから開く」→「フォト」で開く。

---

## 5. LaTeX に貼る

```latex
\begin{figure*}[tb]
  \centering
  \begin{minipage}{0.32\textwidth}\centering
    \includegraphics[width=\linewidth]{figs/humanoid_g1.jpg}\\
    (a) ヒューマノイド（関節拘束が支配的）
  \end{minipage}\hfill
  \begin{minipage}{0.32\textwidth}\centering
    \includegraphics[width=\linewidth]{figs/quadruped_go1.jpg}\\
    (b) 四足歩行（衝突判定が支配的）
  \end{minipage}\hfill
  \begin{minipage}{0.32\textwidth}\centering
    \includegraphics[width=\linewidth]{figs/arm_franka.jpg}\\
    (c) アーム（関節・接触共に少なめ）
  \end{minipage}
  \caption{対象とした3種のタスク概要と物理的特性の仮説（可視化のため歩行タスクの地形は箱型に限定）}
  \label{fig:tasks}
\end{figure*}
```

- 3枚の縦横比が同じなので `width=\linewidth` だけで高さもそろう
- 画像形式の使い分け：レンダリング画像・写真 → JPEG、グラフ → PDF（ベクター）、文字主体のスクショ → PNG

---

## 6. 補足・注意点

- **地形を変えていることを明記する。** 計測はデフォルトの Rough 地形（階段・坂・デコボコ等の混合）で行っているが、図は箱型のみ・難易度 0.6〜1.0 に変更して撮っている
- **G1 の中腰姿勢** は学習結果として自然なもの（報酬に腰の高さの項がなく、転倒ペナルティ -200 が大きいため重心を低く保つ）
- **Franka は取っ手を握り続けず、引っかけて勢いで開ける** 挙動を獲得している。報酬が主に「取っ手への接近」と「引き出しの開き量」で決まるため。シミュレーションのバグではない

---

## 7. トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `NVML_ERROR_UNKNOWN` / `no CUDA-capable device is detected` | コンテナから GPU が見えなくなっている | `docker restart isaac-lab-base` → `./docker/container.py enter` → `nvidia-smi` で確認 |
| `FileNotFoundError: .../logs/rsl_rl/...` | skrl で学習したのに rsl_rl の `play.py` を使った | `scripts/reinforcement_learning/skrl/` 側を使う |
| `TypeError: PPO.act() missing 1 required positional argument: 'states'` | skrl のバージョン差 | 1-1 の `sed` で `act(obs, None, ...)` に修正 |
| `AttributeError`（`commands` / `terrain` 関連）でアームが動かない | 歩行用の撮影設定をアームに使った | `play_capture_franka.py` を使う |
| 奥のほうが平らに見える | 地形エリア外の平らな縁が映っている | `num_rows/num_cols` を増やす、カメラを見下ろす角度に、`env_index` を変える |
| ffmpeg で `does not contain an image sequence pattern` | 警告のみ（画像は出力されている） | `-update 1` を付けると消える |

> `./docker/container.py stop` は `docker compose down` でコンテナを削除する。`scripts/` と `logs/` 以外（コンテナ内で `pip install` したもの等）は消える。GPU の復旧だけなら `docker restart` で足りることが多い。
