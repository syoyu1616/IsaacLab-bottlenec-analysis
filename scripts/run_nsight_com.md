# Nsight Computeでのカーネルプロファイリング実行手順

NVIDIA Nsight Compute (`ncu`) を使用して、Isaac Labの強化学習スクリプト内の特定のCUDAカーネルを詳細にプロファイリングするための手順です。

## ⚠️ 実行前の注意事項

**Nsight Systems (`nsys`) のプロファイリングを実行するためにスクリプト等のコードを変更していた場合は、必ず元の状態に戻してから**本コマンドを実行してください。
（不要なフックやマーカーが残っていると、`ncu` のプロファイル結果に影響を与えたり、意図しないエラーの原因になる可能性があります）

---

## 実行コマンド

以下のコマンドをターミナルで実行します。

```bash
HOME=/tmp /opt/nvidia/nsight-compute/2024.2.1/ncu \
  --kernel-name "stepArticulation1TTGS" \
  -o stepArti_ \
  --launch-skip 5000 \
  --set full \
  -c 10 \
  -f \
  /workspace/isaaclab/_isaac_sim/python.sh scripts/reinforcement_learning/skrl/train.py \
  --task Isaac-Open-Drawer-Franka-v0 \
  --num_envs 8192 \
  --headless
```

## コマンドオプションの解説

プロファイラ (`ncu`) に関する主要なオプションの意味は以下の通りです。

* **`HOME=/tmp`**: 一時的にHOMEディレクトリを変更します（Isaac Sim等の特定の環境下での権限・キャッシュエラーを回避するため）。
* **`/opt/nvidia/nsight-compute/2024.2.1/ncu`**: Nsight Computeの実行パスを指定しています。
* **`--kernel-name "stepArticulation1TTGS"`**: プロファイル対象とする特定のCUDAカーネル名を指定します。
* **`-o stepArti_`**: 出力ファイル名のプレフィックスを指定します。
* **`--launch-skip 5000`**: プロファイルを開始する前に、指定したカーネルの最初の5000回の起動をスキップします（ウォームアップ期間を除外するため）。
* **`--set full`**: 全てのメトリクスを収集し、最も詳細なプロファイリング情報を取得します。
* **`-c 10`**: プロファイルする対象カーネルの実行回数を10回に制限します。
* **`-f`**: (Force) 既に同名の出力ファイルが存在する場合、上書きします。

## 実行結果

コマンドの実行が完了すると、カレントディレクトリにNsight Compute用のレポートファイル（通常は `stepArti_.ncu-rep` のような拡張子）が生成されます。

このファイルは、GUIツールの **NVIDIA Nsight Compute** を使って開くことで、メモリアクセスパターンやレジスタ使用量など、カーネルのボトルネックを詳細に分析することができます。
