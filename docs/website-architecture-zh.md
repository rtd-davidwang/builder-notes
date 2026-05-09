# RTD 官网架构与内容发布工作流

> 一份写给"也准备搭网站的朋友"的实操分享文档
> 关键词：静态网站、Hostinger、GitHub、Claude Code、自定义 skill、品牌一致性

---

## 一、为什么这样搭：设计取舍

搭一个公司官网，其实大部分痛苦不在"做出来"，而在"持续维护"。我在做 Run-True Decision (RTD) 官网（[www.run-true.com](https://www.run-true.com)）时，把这件事想清楚了几条原则：

1. **静态优先**。能用静态 HTML 解决的，就不引入动态后端。WordPress / Strapi / Next.js SSR 这类方案表面省事，长期都会变成"插件升级 + 数据库备份 + 安全补丁"的负担。
2. **代码即内容**。所有页面、博客、品牌资源都进 Git。"线上是什么样"等于"main 分支是什么样"，没有第二份 source of truth。
3. **AI 是创作和运维的协作者，而不是临时工**。把日常重复的写作、配图、品牌检查、部署流程沉淀成 Claude Code 的自定义 skill，让模型每次都能按同一个 SOP 工作，避免"每次都得重新解释一遍"。
4. **品牌规范要可执行**。RTD 有自己的颜色、字体、语气和文案禁忌（比如初创公司不能承诺合规、不能写竞品名）。这些不能只写在 PDF 里，要变成 Claude Code 调用时自动加载的 skill。

带着这四条原则，最后落地的就是下面这套三层架构。

---

## 二、高层架构：三大组件如何协作

```
┌──────────────────────────────────────────────────────────────────┐
│                    创作与维护层 (Claude Code)                     │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────────┐    │
│  │  /rtd-blog   │   │ /rtd-brand-  │   │ /ship-pr / /jdi    │    │
│  │  博客流水线  │   │ guidelines   │   │ 提交 → PR → 合并   │    │
│  │              │   │ 品牌守门员   │   │                    │    │
│  └──────────────┘   └──────────────┘   └────────────────────┘    │
│           │                  │                     │             │
│           └──────────────────┴─────────────────────┘             │
│                              │                                   │
│                              ▼                                   │
└──────────────────────────────┼───────────────────────────────────┘
                               │ git push / gh pr create
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                源码与 CI/CD 层 (GitHub)                          │
│                                                                  │
│   main 分支 ──► GitHub Actions (.github/workflows/deploy.yml)    │
│                  │                                               │
│                  ├─ Pagefind 构建全文搜索索引                    │
│                  ├─ SCP 同步 site/ → Hostinger public_html/      │
│                  └─ Smoke check：curl 9 个关键页面验证可达       │
└──────────────────────────────┼───────────────────────────────────┘
                               │ scp -P <SSH_PORT>
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                  托管层 (Hostinger Business)                     │
│                                                                  │
│   public_html/                                                   │
│   ├── index.html  /platform/  /blog/  /pricing/ ...              │
│   ├── assets/  (图片、CSS、JS)                                   │
│   ├── _pagefind/  (CI 生成的搜索索引)                            │
│   └── .htaccess  (URL rewrite, 安全头)                           │
│                                                                  │
│   域名指向 → www.run-true.com                                    │
└──────────────────────────────────────────────────────────────────┘
```

三层各司其职，且任何一层出问题都不会污染另外两层。

---

## 三、各层组件职责拆解

### 3.1 Hostinger（托管层）

**承担什么**：纯静态文件托管 + 域名 + HTTPS 证书。

**为什么选它**：
- 价格便宜（共享主机年费 < $50），对一个初创公司绰绰有余
- 自带 HTTPS、DNS 管理、邮件转发，一站式
- 支持 SSH/SCP，可以用脚本自动部署，不依赖控制台点点点
- 静态文件托管出问题概率极低，半年到一年才需要看一眼

**重要细节**：
- Hostinger 官方的 MCP API 对 addon domain 支持有限，部署优先走 SSH/SCP（更稳定可控）
- SSH 端口可能不是 22（共享主机常用非标准端口，比如 65002 之类，登录控制台查具体值）

### 3.2 GitHub（源码与 CI/CD 层）

**承担什么**：版本控制 + 自动部署。

**为什么选它**：
- Git 本来就是写作的最佳搭档：每篇博客都有 PR、有 review、有可回滚的历史
- GitHub Actions 免费额度对一个低频站点完全够用
- main 分支 = 生产环境的契约非常清晰

**核心配置**（`.github/workflows/deploy.yml`）：

```yaml
on:
  push:
    branches: [main]
    paths: ['site/**']   # 只有 site/ 改动才触发部署，文档/计划改动不浪费 CI

jobs:
  deploy:
    steps:
      - checkout
      - 装 Node.js
      - npx pagefind --site site --output-path site/_pagefind   # 全文搜索
      - scp -r site/. → Hostinger:public_html/                  # 同步
      - 循环 curl 9 个关键页面 → 全部 200 才算成功               # 验证
```

整套流程平均 4 分钟跑完，从 `git push` 到上线可见。

### 3.3 Claude Code（创作与维护层）

**承担什么**：写博客、做配图、查品牌一致性、生成提交、开 PR、上线后验证。

**为什么是这一层最关键**：
- 静态网站省了运维，但写内容、做图、对齐品牌依然是高频且耗时的事
- Claude Code 的 **skill 机制** 让每个工作流都可以沉淀为一个可复用的脚本，而不是每次重新对话
- skill 自带 SOP（标准操作流程）+ 模板 + 安全约束，模型不会"自由发挥"导致破坏

我目前在用的几个核心 skill：

| Skill | 作用 | 触发方式 |
|-------|------|----------|
| `/rtd-blog` | 端到端博客流水线（下一节详解） | 命令式 |
| `/rtd-brand-guidelines` | 颜色、字体、语气检查 | 任何创作动作自动加载 |
| `/ship-pr` | 提交 → push → PR → squash merge → 清理分支 | 命令式 |
| `/jdi` | 完整 SDLC：peer review → 实现 → commit review → 部署 | 命令式 |

**所有 skill 都存在 git 仓库里**（`.claude/skills/`），意味着它们也享受版本控制和 PR review，团队协作零成本扩展。

---

## 四、端到端：从一句话想法到上线博客

这是整套系统最爽的部分。完整流程一句命令触发：

```
/rtd-blog "Anthropic 发布金融 AI Agents 对 RTD FDE 的战略影响"
```

下面是这个命令背后真正发生的事：

```
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 0  Preflight                                             │
│  ─ 解析参数（标题、URL、文件、品类、slug、格式）                │
│  ─ 检查工具：Pillow、gh、Gemini MCP、git 是否干净               │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1  Research & Planning                                   │
│  ─ WebFetch / Read 参考资料                                     │
│  ─ 生成 Title / Slug / Category / Excerpt / Outline             │
│  ─ ⚠️ 用户检查点：确认大纲再继续                                │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 2  Write Draft (1200–1800 字)                            │
│  ─ 按 RTD 语气：先讲客户痛点，再讲方案；不夸大、不引战          │
│  ─ 自动加 TL;DR、内链、外链、CTA                                │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 3  Quality Gate (12 项自动检查)                          │
│  ─ 字数、链接、内容合规 + 品牌禁忌词检查（无"MAS-compliant"等）│
│  ─ 无竞品名、无具体规则数、无性能保证                           │
│  ─ 可选：调用 agent-bus 让 rtd-fde.worker 做技术 review         │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 4  Hero Image (Gemini 生图)                              │
│  ─ 根据文章主题生成 abstract 风格图                             │
│  ─ Pillow 自动压缩到 hero.jpg / hero.webp / hero-thumb.*        │
│  ─ 严格限制 ≤150KB（保证页面加载速度）                          │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 5  HTML Assembly                                         │
│  ─ 复制 post-template.html → site/blog/<slug>/index.html        │
│  ─ 替换所有 POST_* 占位符；移除 noindex                         │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 6  Index & Sitemap Update                                │
│  ─ site/blog/index.html 插入新卡片（含 category 标签、渐变色）  │
│  ─ site/sitemap.xml 新增 URL + 更新 blog 列表 lastmod           │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 7  Ship                                                  │
│  ─ git checkout -b blog/<slug>                                  │
│  ─ git commit + git push                                        │
│  ─ gh pr create → squash merge → main                           │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 8  Auto Deploy + Verify                                  │
│  ─ GitHub Actions 自动跑 (4 分钟左右)                           │
│  ─ Pagefind 索引重建 → SCP 同步 → 9 页 smoke check              │
│  ─ curl 验证新 URL 200 → 完成                                   │
└─────────────────────────────────────────────────────────────────┘
```

**实际体验**：从一句话 brief 到上线，全程大约 15-25 分钟。其中我自己只在两处需要"动脑子"：
1. Stage 1 末尾确认大纲（决定文章方向）
2. Stage 3 看 quality gate 报告（偶尔需要拍板有争议的措辞）

其余全部是 Claude Code 自动跑。

---

## 五、品牌一致性如何"自动"

这是最容易被低估的一环。很多人做品牌规范是写一份 PDF，然后没人看。

我的做法是：**把品牌规范变成 Claude Code 的 skill**。

`/rtd-brand-guidelines` 这个 skill 的内容是 RTD 整套视觉与文案规范的可执行版本：

```markdown
## RTD 颜色 token
- Deep Navy  #0B1D3A  → 英雄区背景、页头页脚
- Brand Navy #152A4E  → 段落底色、卡片
- Ocean      #1B6B93  → 图表、数据可视化
- Wave       #2E97CB  → 链接、高亮
- Accent Orange #FF6B35  → CTA 按钮

## 字体
Inter（Bold for headings, Regular 16px for body）

## 语气
- 先讲客户痛点，再讲产品
- 平实语言，避开 "AI-powered" 这种竞品都在用的口号
- 自信但谦虚：明确区分 "Available now" 和 "Roadmap"

## 硬性禁忌（任何公开内容）
✘ "MAS-compliant" / "OJK-certified"   → 改 "designed to support …"
✘ 具体规则数（银行模块）              → 用 "pre-configured templates"
✘ 性能保证 ("guaranteed <50ms")        → 用 "designed for millisecond-level …"
✘ 直呼竞品名                          → 用 "Global Platforms"
```

任何创作类 skill（`/rtd-blog`、`/linkedin-post` 等）都会**自动加载** `/rtd-brand-guidelines`。Claude 在写文案、选颜色、做配图时，规范是"内置"的，不是"可选的"。

加上 quality gate 里的硬性正则检查（`grep` 品牌禁忌词），双保险。

---

## 六、这套架构的实际收益

| 维度 | 实际表现 |
|------|----------|
| 月度运维成本 | < $5（Hostinger 共享主机分摊 + GitHub Actions 免费额度）|
| 单篇博客发布耗时 | 15-25 分钟（含品牌检查 + 上线验证）|
| 部署可靠性 | CI 自带 9 页 smoke check，失败回滚靠 git revert，5 分钟内恢复 |
| 安全面 | 无数据库、无 admin panel、无 wp-login，攻击面只剩 SSH（key-only）|
| 扩展性 | 加新页面 = 加一个 HTML 文件；加新工作流 = 加一个 skill 文件；都进 git |
| 团队协作 | 内容、设计、代码全部走 PR 流程，review/回滚都用同一套工具 |

唯一短板：**当博客超过 30 篇时**，纯静态 HTML 的卡片维护会变累，那时候考虑迁到 SSG（11ty / Astro / Hugo）。但目前 24 篇还远未到瓶颈。

---

## 七、如果你想类似搭一个：建议起步路径

按"由易到难、从必要到锦上添花"排：

### 第 1 步：托管 + 仓库（半天）
1. 在 Hostinger 买一个 Business plan（或同类 $5/月级别的虚拟主机）
2. 配置 SSH key，验证 `scp` 能从本机推到 `public_html/`
3. GitHub 建 repo，把网站源码（一个简单的 `index.html` 即可）放进去

> 💡 **如果你也想用 Hostinger（可选）**
> RTD 官网用的就是 Hostinger Business plan，跑了一年多稳定可靠。如果你打算从这里开始，可以用我的推荐码：
>
> 🔗 **链接**：<https://www.hostinger.com?REFERRALCODE=ZRQCLOUDMYI1>
> 🎟️ **推荐码**：`ZRQCLOUDMYI1`
>
> 走这个链接 / 用这个码注册，**你能拿到首次购买 20% 折扣，我能从 Hostinger 拿到一点推荐 credit**——属于双赢。当然完全不强求，DigitalOcean、Cloudflare Pages、Vercel、Netlify 都能跑这套流程，挑你顺手的就行。

### 第 2 步：CI 自动部署（半天）
1. 写 `.github/workflows/deploy.yml`：监听 main 分支，scp 同步整个 site/ 目录到 Hostinger
2. SSH 私钥用 base64 存到 GitHub Secrets
3. 加一个 curl smoke check，部署成功才算成功

### 第 3 步：装 Claude Code（半小时）
1. 安装 Claude Code CLI
2. 在 repo 根目录创建 `CLAUDE.md`，写清楚：
   - 项目是什么
   - 部署流程是什么
   - 内容/品牌的硬性约束（哪些词不能写）
3. 这一步做好了，后续每次对话 Claude 都自带 context

### 第 4 步：沉淀第一个 skill（1-2 天）
1. 在 `.claude/skills/` 下建一个目录，写 `SKILL.md`
2. 从最高频的工作流开始（对我来说是博客发布）
3. 把每一步拆成 stage，每个 stage 写清楚输入、输出、检查点
4. 配上模板文件（`post-template.html`、`card-template.html` 等）

### 第 5 步：品牌规范变 skill（1 天）
1. 整理颜色、字体、语气、禁忌词
2. 写成一个独立 skill（`/your-brand-guidelines`）
3. 在创作类 skill 里 reference 它

### 第 6 步：进阶——AI 配图、品牌守门员、agent-bus（按需）
- Gemini / Imagen 生图 + Pillow 压缩 → hero image 自动化
- quality gate 自动检查内容合规 + 品牌禁忌词
- 用 sub-agent 做 peer review、跨项目消息（这是更高级用法）

**给你的核心建议**：
> 不要一开始就追求"完美架构"。先用最朴素的"HTML + scp + 一个 CLAUDE.md"跑起来，发布前 5 篇博客。**然后看哪一步最痛**，再针对那一步写第一个 skill。

skill 是用出来的，不是设计出来的。我现在用的所有 skill 都是从"咦这一步又重复了"开始的。

---

## 八、可以直接复用的 vs. 需要自己定制的

| 部分 | 可复用 | 需要定制 |
|------|--------|----------|
| GitHub Actions deploy.yml | ✅ 改 SCP host/user/path 即可 | — |
| 静态网站基础结构（`site/` 目录） | ✅ 模板可参考 | — |
| /rtd-blog skill 流程框架 | ✅ Stage 划分通用 | 模板、品牌检查、内链规则 |
| /rtd-brand-guidelines | — | ❌ 必须替换为你的品牌 |
| Hostinger 部署路径 | — | ❌ 主域名 vs. addon domain 命令不同 |
| 内容合规 + 品牌禁忌词列表 | — | ❌ 你的业务约束不一样 |

---

## 附录：常用命令速查

```bash
# 触发一次完整博客流水线
/rtd-blog "你的博客主题"

# 手动同步整站（CI 出问题时的兜底）
scp -P <SSH_PORT> -i ~/.ssh/<DEPLOY_KEY_NAME> -r site/. \
    <SSH_USER>@<HOSTINGER_HOST>:<DEPLOY_PATH>/

# 看 CI 状态
gh run list --limit 5
gh run watch <run-id>

# 上线验证
curl -I https://www.run-true.com/blog/<slug>/
```

---

**最后**：这套系统最大的价值不是"用 AI 写博客"，而是**让 AI 接手所有重复劳动，让你只在真正需要判断的地方出手**。一个人能维护一个公司官网 + 高频博客 + 品牌一致性，不靠堆人，而是靠把流程沉淀成可复用的 skill。

如果你也准备搭，欢迎照着这条路径走，遇到坑可以随时聊。
