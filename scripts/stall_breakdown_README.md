# stall_breakdown.py

Nsight Compute（ncu）の Source ページ（SASS 表示）から書き出した CSV を読み込み、**ストール要因ごとに「どの種類のメモリ命令で起きているか」**を集計するスクリプト。

あわせて、カーネル先頭のスタック確保量（スレッドあたりのローカルメモリ使用量）も表示する。

---

## 1. 何がわかるか

| 出力 | 内容 |
|---|---|
| スタック確保量 | カーネル先頭の `IADD3 R1, R1, -0x...` から読み取った、スレッドあたりのスタックサイズ |
| 各ストール要因の割合 | `stall_lg` などの列ごとに、ストールサンプル全体に占める割合 |
| 各ストール要因の内訳 | そのストールが Local / Shared / Global / その他 のどの命令で起きているか |

---

## 2. 必要なもの

- Python 3（標準ライブラリのみ。追加インストール不要）
- ncu のレポート（`.ncu-rep`）。**命令単位のサンプリング（SourceCounters）を含めて計測したもの**

---

## 3. 手順

### 3.1 ncu で計測する

命令単位のストールを見るには `SourceCounters` セクションが必要。`--set full` でも可。

```bash
ncu -k regex:<カーネル名の一部> --launch-skip <初期化を避ける回数> -c 1 \
    --section SourceCounters --section WarpStateStats \
    -o <出力名>  <学習コマンド>
```

例（四足歩行の衝突判定カーネル）:

```bash
ncu -k regex:convexCore --launch-skip 200 -c 1 \
    --section SourceCounters --section WarpStateStats \
    -o animal_16384_flagless  ./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task Isaac-Velocity-Rough-Unitree-Go1-v0 --num_envs 16384 --headless
```

### 3.2 GUI で列を追加して CSV に書き出す

1. `.ncu-rep` を Nsight Compute の GUI で開く
2. 上部の **Page** を **Source** に切り替え、表示を **SASS** にする
3. 表のヘッダを右クリック（または列の設定）して、次の列が表示されていることを確認・追加する
   - `Warp Stall Sampling (All Samples)`（既定で表示されている）
   - `Address Space`（既定で表示されている）
   - `stall_lg`（LG Throttle）
   - `stall_long_sb`（Long Scoreboard）
   - `stall_mio`（MIO Throttle）
   - 他のストール要因も見たい場合は `stall_` で始まる列を追加する（自動で集計対象になる）
4. Source ページの書き出し機能で CSV として保存する

### 3.3 スクリプトを実行する

```bash
python3 stall_breakdown.py <CSV ファイル> [--lookback 80] [--top 6]
```

| オプション | 既定値 | 意味 |
|---|---|---|
| `--lookback` | 80 | Long Scoreboard の追跡でさかのぼる命令数 |
| `--top` | 6 | 各ストール要因で表示する分類の数 |

---

## 4. 集計の方法

### LG Throttle・MIO Throttle など（Long Scoreboard 以外）

これらのストールは、**キューに入れずに待たされている命令そのもの**に記録される。そのため、その命令の種類でそのまま分類する。

| 分類 | 対象の命令 |
|---|---|
| local | `LDL`, `STL`、Address Space が Local の汎用 `LD` / `ST` |
| shared | `LDS`, `STS`, `ATOMS`, `LDSM`、Address Space が Shared の汎用 `LD` / `ST` |
| global | `LDG`, `STG`, `ATOMG`, `RED`, `ATOM`、Address Space が Global の汎用 `LD` / `ST` |
| other:XXX | 上記以外（XXX は命令名） |

### Long Scoreboard（推定）

Long Scoreboard は、ロードの結果を**使う側**の命令（`MOV` や `FADD` など）に記録される。そこで次の手順で、待っているロード命令を推定する。

1. ストールした命令が読んでいるレジスタを取り出す
2. 直前の命令から最大 `--lookback` 命令さかのぼり、そのレジスタに値を書き込んだ命令を探す
3. 見つかった命令が L1TEX 経由のロード（`LDL`, `LDG`, 汎用 `LD`）なら、そのアドレス空間で分類する

**注意点**

- 分岐をまたぐ実行の流れは考慮していない（SASS を上から順に見ているだけ）
- 追跡できなかったものは `unresolved` に入る
- `--lookback` を変えて（例: 80 と 120）結果が大きく変わらないことを確認しておくと、推定の安定性を説明できる

---

## 5. 出力例

四足歩行（Unitree Go1）、16384 環境、近似なし、`convexCoreTrimeshNphase_Kernel32`:

```
カーネル: convexCoreTrimeshNphase_Kernel32
命令数（静的）: 20736, ストールサンプル合計: 3669991
スタック確保量: 7888 バイト/スレッド（IADD3 R1, R1, -0x1ed0, RZ）

== stall_lg: ストール全体の 26.26%
   local           83.39%
   global          15.48%
   shared           1.14%

== stall_long_sb: ストール全体の 44.50%
   （待っているロード命令をレジスタ依存からさかのぼって推定、最大 80 命令）
   local           77.98%
   unresolved      14.33%
   global           7.69%

== stall_mio: ストール全体の 21.45%
   shared          52.62%
   local           37.49%
   global           8.97%
   other:S2R        0.89%
   other:SHFL       0.02%
   other:MUFU       0.00%
```

### この結果の読み方

- **LG Throttle** の 8 割以上が Local アクセス命令 → スピルやスタックへのアクセスがメモリ命令のキューを詰まらせている
- **Long Scoreboard** の約 8 割が Local メモリからのロード待ち（推定） → グローバルメモリの帯域待ちではなく、スピルしたデータの読み戻し待ちが中心
- **MIO Throttle** は共有メモリ命令（特に `STS.128` が連続する区間）と Local アクセス命令が大半。特殊関数（`MUFU`）の寄与はほぼない
- 3 つを合わせると、ストール全体の約 3 分の 2 が Local メモリへのアクセスに関係している
- スタック確保量 7,888 バイト/スレッドは、非インライン関数の呼び出し（`CALL` / `RET`）によるレジスタ退避や、GJK/EPA の作業領域によるものと考えられる

---

## 6. 近似前後の比較

近似あり（`--kit_args="--/physics/collisionApproximateCylinders=true --/physics/collisionApproximateCones=true"`）で計測したカーネルの CSV にも同じスクリプトを実行し、次を比べる。

- スタック確保量が小さくなっているか（または `IADD3 R1, R1, -0x...` 自体がなくなっているか）
- Local 由来のストールが減っているか
- MIO Throttle・LG Throttle の割合が減っているか

近似後は衝突判定が別のカーネルに置き換わっている可能性があるため、同じ役割のカーネル同士を比べること。

---

## 7. 補足：個々の命令を GUI で確認する方法

集計結果の裏付けとして、代表的な命令のストール内訳を GUI で直接確認できる。

1. Source ページ（SASS 表示）で、上部の **Navigate By** から見たい指標（例: `stall_mio`）を選ぶ
2. 矢印ボタンで値の大きい命令に順に移動する
3. `Warp Stall Sampling (All Samples)` のセルにマウスを乗せると、その命令のストール要因の内訳が表示される

確認済みの例（上の出力例と同じレポート）:

| 命令 | ストール要因の内訳 |
|---|---|
| #1418 `STS.128 [R3+0xc00], R40` | MIO Throttle 97.6% |
| #20304 `RET.REL.NODEC R20, ...` | Long Scoreboard 99.9%（直前の `LDL` によるスタックからの読み戻し待ち） |
