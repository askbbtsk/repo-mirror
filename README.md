# repo-mirror

把一个 GitHub 仓库**原样搬到自己的独立仓库**，每 30 分钟自动同步一次。

搬三样东西：源码、git tag、Release（含附件、`prerelease` 标记、更新说明）。
客户端照旧能按原来的方式拿到更新，只是域名换成了你自己的。

**为什么不用 Fork**：fork 会进入同一个 fork network，上游被 DMCA 处理时所有 fork 一起遭殃——你以为的备份，是同一根绳上的蚂蚱。这个模板建的是**完全独立的仓库**，靠定时任务单向拉取，和上游不共享任何网络关系。

> 边界说清楚：这只解决「被上游牵连」，不解决「你自己侵权」。搬之前先看对方许可证和 GitHub 服务条款，责任在部署的人。

---

# 最低用法

三步，约 1 分钟，**不配任何 secret 也能跑**。

1. 点右上角 **Use this template → Create a new repository**（别点 Fork，理由见上）。建议建成 **Public**。
2. 新仓库 → **Settings → Secrets and variables → Actions → Variables → New repository variable**
   名字 `UPSTREAM`，值填上游，形如 `owner/repo`。**只需要这一个。**
3. 新仓库 → **Actions** 页，若提示被禁用就点 *I understand my workflows, go ahead and enable them*；
   然后 **mirror → Run workflow**，`release_limit` 填 `0` 跑一次（只同步源码和 tag，很快），确认没报错。

> 模板仓本身的 Actions 是**故意关掉**的（它没有 `UPSTREAM`，开着只会每 30 分钟报一次错）。用模板建出来的新仓库是**独立设置**，Actions 默认开启，跟模板仓无关。如果进去看到被禁用，按上面第 3 步点一下启用即可。

之后每 30 分钟自动跑，不用管了。

最低用法下**没有的东西**：git tag 大概率推不上去（权限原因，见 [已知限制](#已知限制)），Release 的 tag 会建在 `main` 最新提交上（tag 名对、指向的提交不对）。**这两点都不影响更新器工作**——它比的是 Release 的 tag 名和版本号。

---

# 最好用法

在最低用法之上加三件事，让它「完全体」。

### 1. 配 `SYNC_TOKEN`，把 git tag 也搬全

GitHub 的默认令牌**没有权限创建 `.github/workflows/` 下的文件**，而上游历史里只要带过 workflow，它的 tag 就会被整条拒掉。

建一个 **classic PAT**，勾 `workflow` + `repo`，存进 **Secret** `SYNC_TOKEN`。之后 tag 会走这个令牌推，全部到位。

> 用 classic PAT，不要用 fine-grained——fine-grained 目前没有 workflow 写权限这一项。

### 2. 首次跑一次全量，别让历史缺口留着

`RELEASE_LIMIT` 默认只同步最近 6 个 Release。首次部署时手动跑一次 `mirror → Run workflow`，`release_limit` 填个大数（比如 `50`）。之后定时任务只补新增的，开销很小。

### 3. 按需收紧范围

- `TAG_PATTERN` 设成 `v*`，跳过上游那些 `nightly-2024...` 之类的无关 tag
- 上游默认分支不是 `main` 的，设 `UPSTREAM_BRANCH`（老仓库常见 `master`）
- 不想要 git tag 的，`SYNC_TAGS=false` 直接跳过那一步

### 验收清单

部署完确认这四条，全中就算成功：

| 检查 | 怎么查 |
| --- | --- |
| 源码到位 | 仓库根目录能看到上游的文件（`README.md` 已是上游那份） |
| `.github/workflows/sync.yml` 还在 | 文件没被上游覆盖 —— 这是最容易踩的坑，见下 |
| Release 数量和标记对 | Releases 页的 `Latest` 指向最新正式版，预览版带 `Pre-release` 标签 |
| 定时任务没被停 | Actions 页没有黄色 banner 提示 workflow 被禁用 |

---

# 配置参考

**变量**（Settings → Variables）：

| 名字 | 默认 | 说明 |
| --- | --- | --- |
| `UPSTREAM` | 无，**必填** | 上游，`owner/repo` |
| `UPSTREAM_BRANCH` | `main` | 上游默认分支 |
| `TARGET_BRANCH` | `main` | 本仓库接收同步的分支 |
| `RELEASE_LIMIT` | `6` | 同步最近多少个 Release；`0` = 只同步源码和 tag |
| `TAG_PATTERN` | `*` | 只同步匹配的 tag |
| `SYNC_TAGS` | `true` | `false` 完全跳过 tag 同步 |
| `KEEPALIVE` | 开启 | `false` 关掉保活提交 |

**Secret**（Settings → Secrets）：

| 名字 | 用途 |
| --- | --- |
| `SYNC_TOKEN` | 推 tag 用，需要 `workflow` 权限。不配则 tag 降级为警告 |
| `UPSTREAM_TOKEN` | 上游是私有仓库，或想避开匿名 API 限流时填 |

**手动运行的两个输入**（只对本次生效）：`upstream`（临时换上游）、`release_limit`（临时改数量）。

---

# 它是怎么工作的

一个 job，八步：

```
检查配置 → 检出本仓 → 配 git 身份
  → 同步源码（rsync --delete，排除 .git / .github / .mirror-keepalive）
  → 同步 tag（git fetch --tags + push --force）
  → 同步 Release（gh api 列表 + gh release create 逐个重建，带附件）
  → 保活（50 天没提交就补一个）
  → 收尾自查（打日志）
```

三个关键设计，都是踩过坑才有的：

**`.github` 目录永远不同步。** 否则本仓库的 workflow 会被上游内容覆盖，下一轮就没人来同步了。这也是为什么源码那步用 `rsync --exclude='.github'` 而不是 `git push --mirror`。

**Release 是「只增不改」。** 已存在的 tag 直接跳过，不更新说明、不替换附件。上游改了旧 Release，这里不会跟着变；要重来只能手动删掉那个 Release 再等下一轮。

**`latest` 是显式钉的。** 不传 `--latest` 时 GitHub 会把「最后创建的那个」自动标成 latest，一轮建多个 Release（首次全量）就会标到旧版本上。所以每个都显式写 `--latest` / `--latest=false`，末尾再兜底钉一次。

---

# 已知限制

**同步后你自己的 README 会被上游覆盖。** `rsync --delete` 会删掉上游没有的文件——包括这个 README 和 `LICENSE`。这是预期行为，不是 bug。想留自己的说明，放到 `.github/` 下（该目录不参与同步），或者写进仓库 Description。

**时间不保证准。** GitHub 的 `schedule` 高峰期普遍延迟几分钟到十几分钟，实际间隔 30~45 分钟。要精确只能自己拿外部定时器打 `workflow_dispatch`。

**60 天会被停。** 仓库连续 60 天没有任何提交时 GitHub 自动停掉定时任务。这里带了保活：最近一次提交超过 50 天就补一个 `.mirror-keepalive` 提交把计时器按回去。

**tag 需要额外权限。** 见 [最好用法](#最好用法) 第 1 条。没配 `SYNC_TOKEN` 时这一步降级成警告，不中断整轮。

**配额。** 公开仓库 Actions 分钟数不计量，随便跑。私有仓库每月 2000 分钟，48 次/天 × 1~3 分钟 ≈ 1400~4300 分钟，**很可能超额**——要么用公开仓库，要么把 cron 调稀。Release 附件还会占本仓存储和流量。

**单个附件上限 2 GiB**，这是 GitHub 的硬限制，超了同步会失败。

**`.github` 里如果上游有 `workflow_dispatch` 之外的东西**（比如 issue 模板），也不会同步——整个目录都被排除。

---

# 排错

| 现象 | 原因 | 处置 |
| --- | --- | --- |
| 跑完仓库还是空的 | `UPSTREAM` 没设或写错 | 看日志「检查配置」那步 |
| `refusing to allow a GitHub App to create or update workflow` | 默认令牌没 workflow 权限 | 配 `SYNC_TOKEN`（警告级，不阻塞） |
| `403` / `rate limit` | 匿名 API 限流 | 配 `UPSTREAM_TOKEN` |
| Release 附件没下来 | 超过 2 GiB，或上游附件权限 | 日志里有具体报错 |
| `Latest` 指向旧版本 | 首次全量时的历史遗留 | 再跑一轮会自己纠正（末尾有兜底） |
| Actions 页黄色 banner | 60 天无提交被停 | 点 Enable；或确认保活步骤没被关掉 |
| 同步完 workflow 没了 | 有人去掉了 `--exclude='.github'` | 从本模板重新拿一份 `sync.yml` |

---

# 给 AI agent 的执行契约

如果你是被要求「用这个模板镜像某个仓库」的 agent，按下面的做，别自由发挥。

**输入**：`owner/repo`（上游）、可选 `release_limit`、可选是否要 git tag。

**步骤**（`gh` 已认证的前提下）：

```bash
# 1. 从模板建仓（不要 fork，不要 clone 上游再推）
gh repo create <你的账号>/<新仓名> --template askbbtsk/repo-mirror --public
# 若 gh 版本不支持 --template，改用网页 Use this template，或
#   git clone https://github.com/askbbtsk/repo-mirror 后改 remote 推空仓

# 2. 设唯一必填变量
gh variable set UPSTREAM -R <你的账号>/<新仓名> -b "<owner/repo>"

# 3. 先跑轻量一轮验证链路（release_limit=0 只走源码和 tag）
gh workflow run sync.yml -R <你的账号>/<新仓名> --ref main -f release_limit=0

# 4. 等它结束，退出码 0 才算过
gh run list -R <你的账号>/<新仓名> --limit 1 --json status,conclusion

# 5. 验收：源码到位 + workflow 还在
gh api repos/<你的账号>/<新仓名>/contents/ -q '.[].name'
gh api repos/<你的账号>/<新仓名>/contents/.github/workflows -q '.[].name'   # 必须仍有 sync.yml

# 6. 再跑一次全量补历史
gh workflow run sync.yml -R <你的账号>/<新仓名> --ref main -f release_limit=50
```

**验收断言**（缺一不可）：

- 第 4 步 `conclusion == "success"`
- 第 5 步根目录出现上游的文件
- `.github/workflows/sync.yml` 仍然存在
- Releases 页 `Latest` 指向上游最新**非预发布**版本

**禁止**：

- ❌ 不要用 Fork —— 达不到「独立仓库」的目的
- ❌ 不要 `git clone` 上游再手动推 —— 会丢掉 Release 对象和附件
- ❌ 不要把任何 token 写进文件或 commit，只能放 Secret
- ❌ 不要把 cron 改到 5 分钟以内（GitHub 最小粒度）或把 `timeout-minutes` 设得比间隔大
- ❌ 不要为了「让 tag 也成功」而删掉 `.github` 排除项 —— 那会让 workflow 在下一轮被上游覆盖，整个镜像停摆
- ❌ 看到 tag 步骤的 warning 不要去「修」它 —— 那是预期内的降级，配 `SYNC_TOKEN` 才算修

**改频率**：编辑 `.github/workflows/sync.yml` 的 `on.schedule.cron`（UTC）。改完 `timeout-minutes` 必须仍然小于间隔。

---

# 许可证

MIT，见 `LICENSE`。
