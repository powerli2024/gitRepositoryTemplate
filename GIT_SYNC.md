# 本地 ↔ GitHub ↔ AutoDL 同步

本仓库只含 **VE 代码**（不含 dataset / 模型 / 跑数）。  
数据与权重仍放 `/root/autodl-tmp/`。

## 一次配置

### 本地（Windows）

```powershell
cd d:\media\VE
git init
git add -A
git status   # 确认没有 .env_ve、ve_out、模型
git commit -m "Initial VE sync for AutoDL"
# 在 GitHub 新建空仓库后：
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git branch -M main
git push -u origin main
```

### AutoDL

```bash
# 首次（若已有旧拷贝，先备份再换 git 目录）
mkdir -p /root/media
cd /root/media
# 私有库用 token / SSH；公有库可直接 clone
git clone https://github.com/<你的用户名>/<仓库名>.git VE
cd VE
cp -n .env_ve.example .env_ve   # 若还没有
chmod +x *.sh
```

依赖：`sep_route` / `USE_SEP=1` 仍需要同级 `../VM/scripts`（mossformer2_onnx）。VM 可另仓或继续手动同步。

## 日常更新

**本地改完推送：**
```powershell
cd d:\media\VE
git add -A
git commit -m "简述改动"
git push
```

**AutoDL 拉取：**
```bash
cd /root/media/VE
git pull --ff-only
# 若有本地改过的 .sh 冲突：git stash -u && git pull --ff-only && git stash pop
```

## 注意

| 进 git | 不进 git |
|--------|----------|
| `scripts/`、`*.sh`、`README`、`.env_ve.example` | `.env_ve`、模型、`ve_*` 跑数、`datasetA` |

不要把 `/root/autodl-tmp/ve_*` 放进仓库。跑数清理见先前归档命令，与 git 无关。
