# シミュレーションと学習の分離計測 (NVTXマーカーの導入)

Nsight Systemsを用いて、Isaac Labにおける「物理シミュレーション（データ収集）」と「方策の更新（学習）」の実行時間を分離して計測するための設定方法です。

## 目的
デフォルトのスクリプトでは、シミュレーションと学習の処理境界がプロファイラ上で見えにくいため、PyTorchのNVTXマーカー（`torch.cuda.nvtx`）を明示的に挿入し、各処理の実行時間を視覚的に解析可能にします。

## 変更手順
対象ファイル: `scripts/reinforcement_learning/skrl/train.py`

上記のファイルを編集し、およそ200行目付近（`runner = Runner(env, agent_cfg)` の直後）に以下のコードを追加・修正します。

```python
    # wrap around environment for skrl
    env = SkrlVecEnvWrapper(env, ml_framework=args_cli.ml_framework)  # same as: `wrap_env(env, wrapper="auto")`

    # configure and instantiate the skrl runner
    # [https://skrl.readthedocs.io/en/latest/api/utils/runner.html](https://skrl.readthedocs.io/en/latest/api/utils/runner.html)
    runner = Runner(env, agent_cfg)

    # =============== 追加ここから ===============
    import torch

    # 1. 物理シミュレーション時間の計測 (env.step をラップ)
    # env.step 内で Isaac Sim の物理ステップと観測の取得が行われます
    original_env_step = env.step
    def profiled_env_step(actions):
        torch.cuda.nvtx.range_push("Physics Simulation") # マーカー開始
        result = original_env_step(actions)
        torch.cuda.nvtx.range_pop()                      # マーカー終了
        return result
    env.step = profiled_env_step

    # 2. 方策更新時間の計測 (agent._update をラップ)
    # skrl の PPO では _update メソッドで学習計算が行われます
    if hasattr(runner.agent, "_update"):
        original_agent_update = runner.agent._update
        def profiled_agent_update(*args, **kwargs):
            torch.cuda.nvtx.range_push("Policy Update")
            result = original_agent_update(*args, **kwargs)
            torch.cuda.nvtx.range_pop()
            return result
        runner.agent._update = profiled_agent_update
    else:
        # デバッグ用: もし _update も見つからない場合はメソッド一覧を表示して確認
        print("[WARNING] '_update' method not found in agent.")
        print("Available methods:", [m for m in dir(runner.agent) if not m.startswith("__")])
    # =============== 追加ここまで ===============

    # load checkpoint (if specified)
    if resume_path:
        print(f"[INFO] Loading model checkpoint from: {resume_path}")
        runner.agent.load(resume_path)

    # run training
    runner.run()
    # close the simulator
    env.close()
```

## プロファイリングの確認方法

上記の変更を加えて Nsight Systems でプロファイリングを実行すると、NVTX行に指定したマーカーが表示されます。
<img width="886" height="39" alt="スクリーンショット 2026-04-09 161240" src="https://github.com/user-attachments/assets/20ac7428-cf31-4472-a6a1-8d4427f3b845" />

上図のように、Physics Simulation と Policy Update のブロックが明確に分かれて表示され、データ生成と学習の実行時間の比率を一目で確認できるようになります。
