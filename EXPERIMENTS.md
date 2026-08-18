# VE 分阶实验（AutoDL，现有开关）

对照 [`PROBLEM.md`](PROBLEM.md)。**不改脚本**；只列环境变量。一次只动一个因子。

前置：代码在 `feat`/`VE` 分支；数据与权重仍在 `/root/autodl-tmp/`（`datasetA`、`pos_neg/best_sep`、`ve_models`、Moss ONNX、Qwen3-ASR）。

```bash
cd /root/media/VE   # 或实际克隆路径
source .env_ve
# 看全部变量: ./run_all.sh --help
```

验收数字只认 `$VE_OUT/reports/asr_cer/summary.md` 里的 **contest / RR / CER_total**，不认校准 JSON 里的代理 contest。

---

## P0 基线：Presence + mix

### P0a — VAD 开 + holdout 选 thr（推荐先跑）

```bash
FORCE_CALIB=1 HOLDOUT_FRAC=0.3 ENROLL_VAD=1 PIPELINE=mix ./run_all.sh
```

产物：

- 提取 / ASR：`/root/autodl-tmp/ve_mix_vad/`
- 校准：`/root/autodl-tmp/ve_presence_best/reports/presence_calib_eres2netv2_sep1_ls_vad_raw/`
- 看校准里的 `holdout.contest_score`（代理，仅参考）+ `ve_mix_vad/reports/asr_cer/summary.md`（真实）

默认已含：`PRESENCE_BACKEND=eres2netv2`、`USE_SEP=1`、`LANG_SPLIT=1`、raw 分数。

### P0b — 无 VAD 对照（只改 ENROLL_VAD）

```bash
FORCE_CALIB=1 HOLDOUT_FRAC=0.3 ENROLL_VAD=0 PIPELINE=mix ./run_all.sh
```

产物：`/root/autodl-tmp/ve_mix_novad/` 与 `…_ls_novad_raw/` 校准桶。与 P0a **并存**。

**P0 门：** 两套都有真实 ASR。VAD 仅当 `ve_mix_vad` 真实 contest **≥** `ve_mix_novad` 才作为后续默认；否则后续实验用 `ENROLL_VAD=0`。

冒烟（可选）：`LIMIT=64 SKIP_ASR=1` 只检查流程；正式对比 `LIMIT=0` 且跑 ASR。

---

## P1 拒识（同一 mix，只换门控）

P0 门控选定后，**固定** `ENROLL_VAD` 与 `PIPELINE=mix`。换 thr 模式或编码器时必须 `FORCE_CALIB=1`（换校准桶）。

### P1a — lang_split vs 全局 thr

```bash
# 全局单 thr（对照）
FORCE_CALIB=1 HOLDOUT_FRAC=0.3 LANG_SPLIT=0 ENROLL_VAD=<P0胜者> PIPELINE=mix ./run_all.sh
```

`LANG_SPLIT=0` 时校准目录标签为 `gthr` 而非 `ls`，与 P0 的 `ls` 桶不覆盖。  
**勿**再设 `VE_OUT` 为同一路径：`run_all` 默认 `ve_mix_vad` 会覆盖 extract。若 P0a 已占用 `ve_mix_vad`，对照请显式分开：

```bash
VE_OUT=/root/autodl-tmp/ve_mix_gthr_vad \
CALIB_DIR=/root/autodl-tmp/ve_presence_best/reports/presence_calib_eres2netv2_sep1_gthr_vad_raw \
FORCE_CALIB=1 HOLDOUT_FRAC=0.3 LANG_SPLIT=0 ENROLL_VAD=1 PIPELINE=mix ./run_all.sh
```

（`ENROLL_VAD=0` 时把路径里的 `vad` 改成 `novad`。）

### P1b — 换 Presence 编码器（仍 mix ASR）

```bash
VE_OUT=/root/autodl-tmp/ve_mix_campplus_vad \
PRESENCE_BACKEND=campplus \
FORCE_CALIB=1 HOLDOUT_FRAC=0.3 ENROLL_VAD=1 PIPELINE=mix ./run_all.sh
```

校准会落到 `presence_calib_campplus_sep1_ls_vad_raw/`。与 eres 比的是**真实 contest**，不是校准代理分。

resnet 同理：`PRESENCE_BACKEND=resnet34_lm` + 独立 `VE_OUT`。预期弱于 eres，只作确认不必全量。

### P1c — 灰带 eres∨camp

**现有 `run_all.sh` 没有灰带融合开关。** 本阶不要在全量上再扫 margin。离线起点：eres 主判、camp 在 thr 下方约 ±0.08 救援（见 encoder 报告）。

过 P1a/P1b 且仍要做融合时：先小补丁 PresenceGate，再用**冻结的 P0 mix 波形**只改 decision 重跑 ASR（或 `--no-resume` 按新 decision）。未过门前禁止改默认。

**P1 门：** 真实 contest **稳定 ≥ P0 胜者 +0.005** 才改默认门控。

共享校准、只换 TSE 时用 `SKIP_CALIB=1`（本文件 P0/P1 **不要**换 TSE）。

---

## P2–P4（诊断 / 难例，非正式默认）

| 阶 | 做什么 | 现有入口 |
|----|--------|----------|
| P2 | enroll 纯净度 | 校准 `scores.jsonl` 的 `enroll_vad` meta；不必再跑全量 ASR |
| P3 | CMD 多次 sep | `score_encoders_on_sep.sh` 或已有 `ve_gate_cmp/sep_streams`；只对难例 uid |
| P4 | TSE 难例 | `LIMIT=… PIPELINE=sep_route` 或 ps4；**禁止**无门控全量替换 mix |

P5（换 ASR 模型）：不排期。

---

## 禁止（与 PROBLEM.md §5 一致）

- 未过门把 `PIPELINE` 默认改成 ps4/wesep/sep_route
- `HOLDOUT_FRAC=0` 在全量上扫完 thr 当部署值并宣称泛化
- 用校准 JSON 的 contest 代替 `asr_cer` 写进结论
- P0/P1 过程中同时改 VAD + 编码器 + PIPELINE

---

## 记录模板

每跑完一轮在笔记里抄：

```text
VE_OUT=
ENROLL_VAD=  LANG_SPLIT=  PRESENCE_BACKEND=  HOLDOUT_FRAC=  FORCE_CALIB=
thr 文件=
RR=  FRR=  CER_total=  CER_accept=  contest=
相对 P0 Δcontest=
结论: Go / No-Go
```
