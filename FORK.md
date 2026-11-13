# 二次开发工作流（本 fork 的维护说明）

本仓库是上游开源项目的 fork，用于二次开发，同时要能持续拿到上游更新。
**所有上游同步只走 `upstream`，自己的提交只落 `dev`。**
本文件由开发者和 AI 共同维护，动手前先读，改完更新第 11 节。

基线（2026-11-13，上游变动后由第 4 节流程刷新）：

| 项 | 值 |
| --- | --- |
| 上游（原作者） | `https://github.com/xiaoyang111222/cn-math-taxonomy.git` |
| 本 fork | `https://github.com/senlinzi520/cn-math-taxonomy.git` |
| `main` | `3ecd054`，纯上游镜像，不提交 |
| `dev` | `e57ff65`，开发线，已推送 |
| `sw.js` VERSION | `v20`（随上游变） |

## 1. 目标与边界

- 目标：在不动上游数据与核心逻辑的前提下加自己的功能，且每次上游更新都能低成本合并。
- 边界：`data/topics.json`、`data/dependencies.json`、`data/questions.json`、`data/cn-curriculum-standards.json` 是生成产物，不手改。
- 许可：上游为 **CC BY-NC 4.0**，可改可分发，**商业使用需向原作者单独授权**；保留 `LICENSE` 与署名，衍生作品同样 NC。

## 2. Git 拓扑

| 名称 | 地址 | 用途 | 规则 |
| --- | --- | --- | --- |
| `origin` | 本 fork | 自己的备份 | `dev` 的主推送目标 |
| `upstream` | 原作者 | 只取更新 | push 已改名为 `DISABLED_push_only_via_origin`，防止误推到作者仓库 |
| `main` | — | 上游镜像 | 只做 `--ff-only` 快进，永不在其上提交 |
| `dev` | — | 开发线 | 所有改动在这里；**已推送，禁止 rebase / force push** |

`.htaccess`、`nginx.htaccess` 是本机部署配置，写在 `.git/info/exclude` 里，不入库。

## 3. 已完成：第一批解耦（2026-11-13）

| 提交 | 文件 | 改了什么 | 为什么 |
| --- | --- | --- | --- |
| `e1c8615` | `assets/stats-config.js` | `endpoint` 置空 | 原值把访问事件发到作者的 `*.workers.dev`，属未经同意上报 |
| `51ab8cb` | `diagnose/share.js` | 分享二维码按 `location.pathname` 反推站点根目录 | 原值写死 `math.pansgo.com`，用户扫码会进作者站点；子路径部署也适用 |
| `c7ef7a2` | `scripts/deploy-cf.sh` | 账号 ID 与 Pages 项目名改环境变量 | 原值硬编码作者 Cloudflare 账号与项目名，误跑会往作者项目部署 |
| `e57ff65` | `.github/workflows/ci.yml`、`tests/test_ci_workflow.js` | `main` 与 `dev` 都触发 CI | fork 的 `dev` 也要跑数据校验；测试里的触发断言同步改 |

## 4. 日常操作

### 4.1 同步上游（建议每周，最长不超过两周）

```bash
git fetch upstream
git log --oneline HEAD..upstream/main     # 先看上游改了什么，再决定全量跟还是挑选
git merge upstream/main                    # 合进 dev
git checkout main && git merge --ff-only upstream/main && git checkout dev   # 镜像 main
```

### 4.2 只跟数据、不跟代码

```bash
git checkout upstream/main -- data/
```

注意：这会覆盖本地 `data/`。自有的点/题不能直接写在这四个文件里（见 6.1）。

### 4.3 只挑某几个提交

```bash
git cherry-pick <sha>
```

上游提交信息写得很细（改了什么、没改什么、为什么），适合按主题挑选。

### 4.4 冲突处理

冲突通常出现在 `data/*.json`、`sw.js`、`index.html`：

- 大 JSON 冲突**不要手改**。取上游版本，再跑自己的生成脚本：
  ```bash
  git checkout upstream/main -- data/questions.json
  python scripts/<自己的生成脚本>.py
  ```
- `sw.js` 冲突：保留上游的 `SHELL`/`DATA` 列表，再把自己的条目补进去，`VERSION` 加一。
- `index.html` 冲突：保留上游正文，只补自己的 `<script>` 行。

### 4.5 同步前打快照

```bash
git tag dev-2026-11-13
```

冲突解坏了可以 `git reset --hard dev-2026-11-13` 回退。

## 5. 禁止事项

1. 不在 `main` 上提交任何东西。
2. 不向 `upstream` 推送，也不给上游提 PR（等于把改动交给作者）。
3. 不对已推送的 `dev` 执行 `rebase`、`commit --amend`、`push --force`。
4. 不手改第 1 节点名的四个数据文件。
5. 不改上游 `README.md`、`CHANGELOG.md` 里的署名与来源说明。

## 6. 改动落点优先级（冲突成本从低到高）

| 优先级 | 做法 | 冲突风险 |
| --- | --- | --- |
| 1 | 新增文件：新目录、`data/custom-*.json`、`assets/custom-*.js` | 几乎为零 |
| 2 | 单点登记：`index.html` 一处 `<script>`、`sw.js` 的 `SHELL`/`DATA`/`VERSION` | 低，集中在几行 |
| 3 | 改上游既有文件（如 `assets/home.js`） | 中，需单独小提交 |
| 4 | 改生成数据（四个大 JSON） | 高，只能脚本化 |

### 6.1 新增内容的具体做法

- 新页面：新建目录（如 `custom/`）+ 首页放入口 + 在 `sw.js` 的 `SHELL` 里登记 + `VERSION` 加一。
- 新脚本：放 `assets/custom-*.js`，插入点只保留 `index.html` 里的一行。
- 新知识点/题：放 `data/custom-*.json`，前端 `fetch` 后与上游数据在内存里合并；或写 `scripts/custom_*.py` 读旧数据 → 追加 → 全量 dump（改动以脚本入版本库，数据文件本地重新生成）。
- ID 前缀：上游按学科占用 `cnm_`/`cnp_`/`cnc_`/`cne_`/`cns_`/`cnj_`/`cnk_` 等。自有内容另起前缀并在此登记，避免与上游将来撞车。

### 6.2 动 `sw.js` 时必须一起做的

改 `SHELL` 或 `DATA` 列表、或改了数据结构，都要把 `VERSION` 加一，否则老用户读到旧缓存。

## 7. 本地校验（提交前至少跑一遍）

```bash
python scripts/validate.py
python scripts/verify_questions.py --report %TEMP%\verify.json
python scripts/check_figures.py
node tests/test_ci_workflow.js
node tests/test_engine.js
```

- Windows 上没有 `python3`，用 `python`。
- `tests/test_pages_script_cache.js` 依赖 POSIX `PATH`（`stub:/usr/bin:/bin`），Windows 上跑不了，以 CI 结果为准；它校验的是 `scripts/deploy-cf.sh` 的参数顺序与缓存目录。
- CI 在 `main` 与 `dev` 推送时跑全套（`validate.py` / `verify_questions.py` / `check_figures.py` / 12 个 node 测试）。数据不合规会红，红就别合。

## 8. 已知事实与坑

| 事项 | 说明 |
| --- | --- |
| 数据规模 | `questions.json` 约 10MB、`topics.json` 3.2MB、`dependencies.json` 986KB，由 `scripts/*.py` 全量重写 |
| 无构建步骤 | 纯静态站，改完直接部署；部署脚本只拷白名单目录 |
| 部署环境变量 | `CLOUDFLARE_API_TOKEN`、`CLOUDFLARE_ACCOUNT_ID` 必填，`CF_PAGES_PROJECT` 选填（默认上游项目名，fork 必须覆盖） |
| 统计 | `assets/stats-config.js` 的 `endpoint` 已留空，本站不向第三方发事件；要统计填自己的地址 |
| 分享二维码 | 按当前站点地址生成，换域名或子路径部署都不用改代码 |
| 首页外链 | `index.html` 第 56/84/140 行、`diagnose/app.js` 第 1302 行仍指向 `pansgo.com`（作者的其它产品），待决策 |
| fork 的 Actions | 首次需在仓库 Actions 页手动启用；公开仓库免费 |
| 网络 | 到 `github.com:443` 间歇性超时，`git push` 失败先换网络或开代理 |
| 凭据 | 本机凭据由 Git Credential Manager 管理，登录过一次后可用；超时残留的 `git-credential-manager-core` 进程可直接结束 |

## 9. 待决策

1. 站点身份：名称、域名、署名方式（许可证要求保留原作者署名）。
2. 首页与诊断结果页的 `pansgo.com` 外链：删掉、换成自己的、或保留。
3. 品牌与配置是否集中到 `assets/site-config.js`（新建），避免散落在各页面。
4. 自有点/题的 ID 前缀。

## 10. 给后续 AI 的约定

- 动手前先读本文件与第 9 节，按第 6 节的优先级选落点。
- 优先新增文件，不复制粘贴上游大文件，不改上游正文措辞。
- 可配置项集中管理（建议 `assets/site-config.js`），不要散落在各 `index.html` 里。
- 一个解耦或一个功能一个提交，提交信息用中文写清「改了什么 / 没改什么 / 为什么」。
- 提交前跑第 7 节校验；跑不了的（如 Windows 下的 bash 测试）要说明原因。
- 改完更新第 11 节与本文件顶部的基线提交号。

## 11. 变更日志

| 日期 | 提交 | 内容 |
| --- | --- | --- |
| 2026-11-13 | `e1c8615` | 关闭指向上游的统计上报 |
| 2026-11-13 | `51ab8cb` | 诊断分享二维码改用本站地址 |
| 2026-11-13 | `c7ef7a2` | 部署脚本云账号与项目名改为环境变量 |
| 2026-11-13 | `e57ff65` | CI 支持 `dev` 分支 |
| 2026-11-13 | 本次 | 新增本文件，记录工作流、解耦约定与 AI 维护规则 |
