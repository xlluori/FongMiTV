# WORKFLOW.md — 分支模型与上游同步流程

本仓库是 [FongMi/TV](https://github.com/FongMi/TV) 的 fork，在其基础上做本地定制开发。本文记录分支约定、日常开发与上游同步的完整操作，以及出问题时的恢复手段。

---

## 1. 分支模型

| 分支 | 角色 | 规则 |
|---|---|---|
| `release`（默认） | **上游镜像** | 永远只做 `upstream/fongmi` 的镜像，**不留任何自有提交** |
| `dev` | 日常开发 | 所有改动都在这里；以补丁形式叠在 `release` 之上 |
| `upstream/fongmi` | 上游只读引用 | 永不直接推送（push URL 已设为 `no_push`） |

```
FongMi/TV:fongmi  ──fetch+快进──▶  你的 release  ──rebase──▶  你的 dev
                                 （纯镜像，不提交）          （你的改动）
```

**为什么 `release` 必须干净**：只要它有一个自己的提交，GitHub 网页上的 `Sync fork` 就无法快进，只会弹出 *Discard commits / Open pull request* 二选一的冲突提示。保持纯镜像，同步永远是一次点击或一条 `--ff-only` 命令。

### 远端配置（一次性）

```bash
git remote add upstream https://github.com/FongMi/TV.git
git remote set-url --push upstream no_push        # 防止误推上游
```

当前状态应为：

```
origin    https://github.com/xlluori/FongMiTV.git   (fetch/push)
upstream  https://github.com/FongMi/TV.git          (fetch)   no_push (push)
```

---

## 2. 日常开发流程

```bash
git switch dev
# ……改代码……
git add -A
git commit -m "feat: 描述这次改动"
git push                     # dev 已跟踪 origin/dev，无需写远端名
```

约定：

- **一个提交只做一件事**。改动集中、逻辑独立，后续 rebase 时冲突面最小。
- 避免整文件格式化、大面积重命名、批量替换 —— 这类改动会让每次上游更新都在同一堆文件上重新冲突。

---

## 3. 同步上游更新

### 3.1 标准流程（推荐）

```bash
# ① 拉取上游最新提交（只下载提交与目录树，文件内容按需取，很快）
git fetch upstream

# ② 让镜像分支快进到上游（--ff-only：万一 release 有自有提交会直接报错，避免产生合并提交）
git switch release
git merge --ff-only upstream/fongmi
git push origin release

# ③ 把 dev 的改动重放到新的基线上
git switch dev
git rebase release

# ④ 如果 dev 已推送过，rebase 改写了历史，需要强推（务必用 --force-with-lease）
git push --force-with-lease origin dev
```

第 ② 步等价于 GitHub 网页上的 **Sync fork → Update branch**。

> `--force-with-lease` 会在远端出现你没见过的提交时拒绝推送，比 `--force` 安全得多。

### 3.2 什么时候改用 merge

如果 `dev` 已经和别人共享（有人基于你的 `dev` 开发），rebase 改写历史会打乱他们的工作。此时把第 ③ 步换成：

```bash
git merge release        # 保留历史，产生一个合并提交；不改写已推送的历史
git push
```

代价是历史变成网状，冲突在一次大合并中集中出现。**单人开发一律用 rebase。**

### 3.3 同步节奏

每次上游有新提交就同步（增量只有几条提交，几十 KB），比攒几个月再同步要轻松得多 —— 冲突越晚处理越难解。

---

## 4. 冲突处理（rebase 中）

### 4.1 看到冲突

```
CONFLICT (content): Merge conflict in app/src/main/java/.../SettingActivity.java
error: could not apply 3f2a1b9... feat: 自定义播放器参数
```

```bash
git status                 # 列出 "both modified" 的文件
```

打开冲突文件，会看到：

```
<<<<<<< HEAD                       ← 上游（rebase 的新基线）
    上游的写法
=======
    你的写法
>>>>>>> 3f2a1b9 (feat: 自定义播放器参数)   ← 你正在重放的提交
```

编辑成最终想要的样子，**删掉 `<<<<<<<`、`=======`、`>>>>>>>` 三行标记**。

### 4.2 逐个提交解决

```bash
git add <已解决的文件>
git rebase --continue        # 进入下一个提交；可能再次冲突，重复本步
```

### 4.3 其它出口

| 命令 | 作用 |
|---|---|
| `git rebase --abort` | 放弃本次 rebase，完全回到开始前的状态 |
| `git rebase --skip` | 跳过当前这个提交（内容不要了才用） |
| `git checkout --theirs <文件>` | 该文件直接采用**上游**版本 |
| `git checkout --ours <文件>` | 该文件直接采用**你的**版本 |

> rebase 时 `ours` = 上游基线，`theirs` = 你正在重放的提交，与直觉相反，注意区分。

### 4.4 让冲突自动复用（可选，强烈推荐）

```bash
git config --global rerere.enabled true
```

开启后，同一个冲突你解决过一次，git 会记住并在下次自动套用 —— 上游反复改动同一处时能省大量时间。

---

## 5. 备份 tag

同步、rebase、强推这类"改写历史"的操作前，先打个 tag 存档。**本地 tag 不占空间、不需要推送，出问题一条命令回去。**

```bash
# 打备份（带日期，便于区分）
git tag backup-$(date +%Y%m%d-%H%M)

# 查看所有备份
git tag -l 'backup-*' --sort=-creatordate

# 出问题了：回到某个备份
git switch dev
git reset --hard backup-20261031-2130

# 确认没问题后删除旧备份
git tag -d backup-20261031-2130

# 需要把备份同步到远端（可选，跨机器恢复用）
git push origin backup-20261031-2130
git push origin --delete backup-20261031-2130
```

推荐在同步脚本里固定写成：

```bash
git tag backup-$(date +%Y%m%d-%H%M) && git fetch upstream && git switch release && git merge --ff-only upstream/fongmi
```

---

## 6. 出事了怎么救

| 场景 | 恢复方法 |
|---|---|
| rebase 中途乱了 | `git rebase --abort` |
| rebase 已完成但结果不对 | `git reset --hard backup-xxx` |
| 提交"消失"了（reset / 强推后） | `git reflog` 找到那次的 hash，`git reset --hard <hash>` |
| 强推覆盖了远端 | `git reflog` 找到本地旧提交 → `git push --force-with-lease origin <hash>:dev` |
| 完全没打 tag 也没 reflog 记录 | 只能从远端拉回：`git fetch origin` 后按分支找回 |

**reflog 是最后一道保险**：默认保留 90 天的 HEAD 移动记录，几乎任何"误操作丢失提交"都能找回。

```bash
git reflog -20              # 最近的 HEAD 变化
git reset --hard HEAD@{3}   # 回到 3 步之前
```

---

## 7. 本仓库的特殊配置

### 7.1 部分克隆（partial clone）

本仓库以 `--filter=blob:none` 克隆：**提交历史完整**（rebase / merge 完全不受影响），但文件内容按需下载。

副作用：切到很久以前的分支、或查看旧提交的 diff 时，会触发联网下载并等待。离线时这类操作会失败，属正常现象。

```bash
# 想把所有历史文件一次补全（体积较大，网速好时再执行）
git fetch --refetch --filter=blob:limit=100m origin
```

### 7.2 TLS / HTTP 配置

国内网络 + 本地代理组合下，schannel 的证书吊销检查常报 `CRYPT_E_NO_REVOCATION_CHECK`。仓库级已配置：

```bash
git config http.schannelCheckRevoke false
git config http.sslVerify false
git config http.version HTTP/1.1
git config http.postBuffer 524288000
```

这些设置只对本仓库生效，不污染全局配置。

### 7.3 推送卡在 502

代理不稳定时 `git push` / `git fetch` 会报 `CONNECT tunnel failed, response 502`，**直接重试即可**，与仓库状态无关。可以加一个重试别名：

```bash
git config --global alias.retry '!f() { for i in $(seq 1 30); do git "$@" && return 0; echo "retry $i"; sleep 3; done; return 1; }; f'
# 用法
git retry push
git retry fetch upstream
```

---

## 8. 国内网络下的实用做法

GitHub 访问不稳是常态，尤其是 `CONNECT tunnel failed, response 502` 和克隆中途 `early EOF`。按可靠性排序：

### 8.1 本地代理（最稳，长期方案）

用 Clash Verge / v2rayN 一类工具，开启系统代理或 TUN 模式，并让 git 明确走它：

```bash
git config --global http.proxy  http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
# 取消
git config --global --unset http.proxy
git config --global --unset https.proxy
```

> 本项目当前环境里的 `127.0.0.1:57727` 是工具自带的内部代理端口，随会话存在，不作为长期依赖。

### 8.2 重试别名（必备兜底）

代理抖动往往几秒就恢复，单纯重试的收益极高：

```bash
git config --global alias.retry '!f() { for i in $(seq 1 30); do git "$@" && return 0; echo "retry $i"; sleep 3; done; return 1; }; f'
git retry push
git retry fetch upstream
```

### 8.3 镜像站加速拉取（应急，只读）

镜像站在原地址前拼接即可，**只用于 `clone` / `fetch`，不要用于 `push`** —— 推送会把凭据经过第三方服务器。

```bash
# 直接临时拉上游
git fetch https://gh-proxy.com/https://github.com/FongMi/TV.git fongmi:refs/remotes/upstream/fongmi

# 或加一个只读镜像远端，官方通道不通时改用它
git remote add mirror https://gh-proxy.com/https://github.com/FongMi/TV.git
git fetch mirror fongmi
```

实测（2026-10-31）:

| 方式 | 状态 | 说明 |
|---|---|---|
| `https://gh-proxy.com/https://github.com/...` | 可用 | 支持 git 协议、Archive、Raw、Release |
| `https://ghproxy.net/` | 不可用 | 连接失败 |
| `https://gitclone.com/github.com/...` | 不可用 | 返回 403 |
| SSH 走 443（`ssh.github.com:443`） | 不可用 | 当前网络下连接超时 |

镜像站是第三方服务，随时可能失效，不要作为唯一通道。

### 8.4 减少要传的数据

- 本仓库已是部分克隆（`--filter=blob:none`），历史齐全、文件按需取，这是最有效的一招。
- 拉取上游写 `git fetch upstream`，**不要**加 `--unshallow` / `--refetch`，否则会重新下载全部历史文件。
- 大文件（`libnode.so` 等）只在切换分支需要时才下载，尽量不要频繁在旧提交间来回切。

### 8.5 心态

日常开发（写代码、提交）完全离线可用。**推送失败不影响继续工作**，攒着等网络好再一次推上去即可。

---

## 9. 一页速查

```bash
# 日常
git switch dev && git add -A && git commit -m "..." && git push

# 同步上游
git tag backup-$(date +%Y%m%d-%H%M)
git fetch upstream
git switch release && git merge --ff-only upstream/fongmi && git push origin release
git switch dev && git rebase release
git push --force-with-lease origin dev

# 救命
git rebase --abort
git reset --hard backup-xxxx
git reflog
```
