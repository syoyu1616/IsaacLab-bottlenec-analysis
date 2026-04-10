# Isaac Lab Profiling & Bottleneck Analysis

このリポジトリは、ロボット制御向けのシミュレーションに基づく強化学習フレームワーク「NVIDIA Isaac Lab」の性能ボトルネック解析（環境数スケーラビリティによる計算律速かメモリ律速の特定）を行うためのスクリプトと設定の差分を管理しています。

## 概要 (Overview)
Isaac Labを用いた大規模並列シミュレーション環境において、環境数（num_envs）の増加がハードウェアリソース（特にGPUのメモリバス帯域幅やキャッシュ効率）に与える影響を解析しました。
プロファイリングには `Nsight Systems` および `Nsight Compute` を使用しています。

## 実験環境 (Prerequisites)
公式のDockerコンテナ（`isaac-lab-base`）を使用し、以下の環境で動作確認を行っています。

* **GPU**: NVIDIA GeForce RTX 4090 (VRAM 24GB)
* **CPU**: Intel Xeon w7-2495X (48 Cores)
* **RAM**: 128GB
* **OS**: Ubuntu 22.04 LTS
* **NVIDIA Driver**: Ver 555.42.06
* **Framework**: Isaac Lab v2.2.1

## 対象タスク (Evaluated Tasks)
特性の異なる以下の3つのタスクを対象とし、環境数を **4096から最大32768まで2倍ずつ増加** させて検証を行いました。

1. **Unitree-G1 (Humanoid)**: 荒地での歩行タスク
   * 自由度: 37
   * 検証環境数: 4096, 8192, 16384, 32768
2. **Unitree-Go1 (Quadruped)**: 荒地での歩行タスク
   * 自由度: 12 (複雑な接触判定が多発)
   * 検証環境数: 4096, 8192, 16384 (※32768はメモリ枯渇により上限)
3. **Franka-PANDA (Arm)**: 戸棚を開けるタスク
   * 自由度: 9 (オブジェクトとの相互作用メイン)
   * 検証環境数: 4096, 8192, 16384, 32768


## Acknowledgements
This repository uses [NVIDIA Isaac Lab](https://isaac-sim.github.io/IsaacLab/) for the simulation environment.

If you use Isaac Lab in your work, please cite their paper:
```bibtex
@article{mittal2025isaaclab,
  title={Isaac Lab: A GPU-Accelerated Simulation Framework for Multi-Modal Robot Learning},
  author={Mittal, Mayank and others},
  journal={arXiv preprint arXiv:2511.04831},
  year={2025}
}
