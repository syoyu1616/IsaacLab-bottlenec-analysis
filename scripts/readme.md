# Scripts Overview

このディレクトリには、Isaac Labのボトルネック解析に使用した実行スクリプトや、プロファイラ（Nsight Systems / Nsight Compute）の起動スクリプト、およびそれらの解説ドキュメントを格納しています。

## ファイル一覧

| ファイル名 | 役割 | 詳細解説 |
|---|---|---|
| `train_profiled.md` | 物理シミュレーションと方策更新を分離計測するため、NVTXマーカーを追加した学習スクリプトとその説明 | [`train_profiled.md`](./train_profiled.md) |
| `run_nsight_sys.md` | Nsight Systemsを用いたプロファイリングを実行するコマンドとその説明。 | [`run_nsight_sys.md`](./run_nsight_sys.md) |
| `run_nsight_com.md` | Nsight Computeを用いた詳細なボトルネック解析を実行するコマンドとその説明。 | [`run_nsight_com.md`](./run_nsight_com.md) |

## 使い方
各スクリプトの具体的な実行方法や、コードの変更意図については、それぞれのファイルに載せています。
