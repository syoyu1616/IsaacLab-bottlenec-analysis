## プロファイリングの実行手順 (Nsight Systems)

Isaac LabのベースとなるPython実行スクリプト（`_isaac_sim/python.sh`）を直接編集し、実行時に `nsys profile` が自動でフックされるように変更します。

### 1. 実行スクリプトの書き換え
コンテナ内で `nano _isaac_sim/python.sh` を開き、末尾のコードを以下のように変更します。

**【変更前】**
```bash
# Show icon if not running headless
export RESOURCE_NAME="IsaacSim"
# WAR for missing libcarb.so
export LD_PRELOAD=$SCRIPT_DIR/kit/libcarb.so
$python_exe "$@" $args || error_exit
```

**【変更後】**
```bash
# Show icon if not running headless
export RESOURCE_NAME="IsaacSim"
# WAR for missing libcarb.so
#export LD_PRELOAD=$SCRIPT_DIR/kit/libcarb.so
#$python_exe "$@" $args || error_exit
exec /nsys_new/bin/nsys profile -o h1_train_profile_python_only \
                            --trace=osrt,cuda,nvtx \
                            -s cpu \
                            --python-sampling=true \
                            --delay 60 \
                            --duration 30 \
                            --kill=sigkill \
                            $python_exe "$@" $args || error_exit
```
※注: /nsys_new/bin/nsys は各自の環境に合わせてNsight Systemsのパスに変更してください


### Nsight Systems (`nsys`) オプションの解説

* **`--trace`**: 基本は `cuda` と `osrt` を指定します。特定のコード部分の実行（強化学習のデータ生成と学習の比率など）を見る場合は `nvtx` も必須です。
* **`--delay`**: Isaac Labの初期設定（ウォームアップ）にかかる時間を計測から外すために設定します。目的のタスクによって要調整です。
* **`--duration`**: プロファイリングの計測時間です。出力ファイルは非常に重くなるため、最初は短い時間（30秒など）から始めることを推奨します。
* **`--kill=sigkill`**: 指定した `duration` 経過後にプロセスを終了させます。これが無いと、毎回タスクを手動でキルする手間がかかります。

### 2. 学習タスクの実行
スクリプトの変更を保存した後、以下のコマンドでシミュレーションを実行します。

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py --task Isaac-Velocity-Rough-Unitree-Go1-v0 --num_envs 16384 --max_iterations 15000 --headless
```

### Isaac Lab 実行オプションの解説

* **`--task`**: 動かすロボットとタスクの設定です（※動かすロボットによっては `skrl` ディレクトリに無い可能性があるので適宜変更してください）。
* **`--num_envs`**: 環境数です。使用しているGPUのメモリ（VRAM）によって上限が決まるため、少なめから始めることを推奨します。
* **`--max_iterations`**: 学習回数です。学習の収束結果を見たい場合に設定します（デフォルトの設定では収束しないことが多いです）。
* **`--headless`**: 画面描画を行わないモードです。シミュレーションの描画処理自体が、純粋な学習性能のボトルネック（阻害要因）になり得るため、プロファイリングの際は付与することを強く推奨します。
