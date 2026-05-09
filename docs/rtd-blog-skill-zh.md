# `/rtd-blog` Skill 实现深度解析

> 配套文档，写给"想自己也搭一个 AI 博客流水线"的开发者
> 配合 [《RTD 官网架构与内容发布工作流》](./website-architecture-zh.md) 阅读

---

## 一、为什么要把博客发布做成 "skill"，而不是每次对话

直接和 Claude 对话写博客的痛点：
- 每次都要重新解释品牌规范、字数要求、文件结构
- 经常漏步骤（比如忘记更新 sitemap、忘记生成 webp）
- 没有质量门控（quality gate），错别字 / 不符合品牌禁忌的话很容易上线
- 没法稳定复用：今天的"提示词 + 模板"明天就忘了

**Skill 解决的就是"把一次对话沉淀成可复用 SOP"** 这件事。

Claude Code 的 skill 系统约定：
- skill 文件放在 `.claude/skills/<skill-name>/SKILL.md`
- 文件头是 YAML frontmatter（声明 name、description、是否 user-invocable）
- 正文用自然语言写出 **每一步要做什么、用哪个工具、什么时候停下来等用户**
- 模板、辅助脚本放在同目录的 `templates/`、`scripts/` 下

Claude Code 启动时会扫描所有 skill 的 frontmatter，把 description 注入上下文。当用户输入 `/rtd-blog "..."`，Claude 会主动加载完整 SKILL.md 并按里面的步骤执行。

---

## 二、`/rtd-blog` 的目录结构

```
.claude/skills/rtd-blog/
├── SKILL.md                  # 主流程定义，约 500 行
└── templates/
    ├── post-template.html    # 单篇博客的 HTML 模板（含 POST_* 占位符）
    └── card-template.html    # 博客列表页的卡片模板（含 {SLUG}、{TITLE} 等占位符）
```

主程序还会调用项目里的两个文件：
- `scripts/process_hero_image.py` —— Pillow 图像后处理脚本
- `site/blog/post-template.html` —— 实际的页面模板（同 templates 下那份保持同步）

---

## 三、SKILL.md frontmatter 长什么样

```yaml
---
name: rtd-blog
description: End-to-end blog post creation — research, write, generate hero image, assemble HTML, update index, and open PR. Triggers on "/rtd-blog".
user_invocable: true
---
```

三个字段的作用：
- `name` —— 命令名，对应 `/rtd-blog`
- `description` —— Claude Code 启动时全局加载这一行，模型用它来判断"要不要触发这个 skill"
- `user_invocable: true` —— 允许用户直接 `/rtd-blog` 唤起；若为 false，则只能在其他 skill 里被引用

---

## 四、九个 Stage：每一步真正发生了什么

`/rtd-blog` 把博客发布拆成 9 个 stage（Stage 0–8），每个 stage 都有明确的 input、output、是否需要用户确认。

### Stage 0：Preflight（启动前自检）

**输入**：用户的命令字符串（含主题 + 可选 flags）

**做什么**：
1. 解析参数：`--url`（参考链接）、`--file`（参考文件）、`--category`、`--slug`、`--format`、`--draft`、`--no-image`、`--skip-approval` (`-s`)
2. 验证 category 是 4 个白名单之一（Fraud Prevention Insights / Banking Technology / Southeast Asia Market / Founder Stories）
3. 验证 slug 没被占用：`site/blog/<slug>/` 必须不存在
4. 检查工具链：
   ```bash
   python3 -c "from PIL import Image; print('Pillow OK')"
   gh auth status
   ```
5. 检查 Gemini MCP 工具可用（除非 `--no-image`）
6. 检查 git working tree 是否干净，不干净则警告并询问是否继续

**为什么这一步重要**：所有"晚发现的错误"都贵。Preflight 让流水线在 30 秒内 fail-fast，而不是写完两千字才发现 slug 重复。

---

### Stage 1：Research & Planning（调研 + 规划）

**做什么**：
1. 如果给了 `--url`：用 `WebFetch` 抓取并总结
2. 如果给了 `--file`：用 `Read` 读取并总结
3. 生成提案：
   - **Title**（< 70 字符，符合 RTD 语气）
   - **Slug**（kebab-case, 3-5 词）
   - **Category**
   - **Excerpt**（140-160 字符）
   - **Format**（prose / how-to / comparison / listicle）
   - **Outline**（按 format 不同结构不同）
4. ⚠️ **Human-in-the-loop 检查点 #1**：把提案交给用户，等待批准/修改

**为什么人在这里要介入**：标题、品类、大纲是文章方向的"宪法"，搞错了后面 1500 字白写。这是 ROI 最高的人工介入点。

---

### Stage 2：Content Writing（写正文）

**做什么**：
1. 写完整的 HTML 正文，外层是 `<div class="prose max-w-none">`
2. 遵循 **answer-first 模式**（每个 `<h2>` 段第一句直接回答问题）—— 这是为 LLM citation readiness 优化的，让 ChatGPT/Perplexity 能引用你
3. 按 format 应用结构（prose 自由、how-to 用步骤、comparison 用表格、listicle 用编号）
4. 强制结尾 CTA blockquote（Founder Stories 类别可豁免）
5. 自动套用品牌约束：不写过度合规承诺、不写性能保证、不点名竞品

**目标字数**：1200-2000 词（8-12 分钟阅读）。

---

### Stage 2.5：Quality Gate（质量门控）

**做什么**：11 项自动检查打分卡

| # | 检查项 | 类型 | 规则 |
|---|--------|------|------|
| 1 | 字数 | 硬 | < 1000 或 > 2500 直接 fail |
| 2 | `<h2>` 数量 | 硬 | ≥ 3 |
| 3 | Answer-first 模式 | 软 | 每段首句 ≤ 50 词 |
| 4 | 内链 | 软 | 至少 1 个站内链接 |
| 5 | 外链 | 软 | 至少 1 个 https 外链 |
| 6 | CTA blockquote | 硬（除 Founder Stories）| 必须含 "Run-True Decision" 文本和联系链接 |
| 7 | Format 结构 | 硬 | listicle 要 5-10 个编号、how-to 要 ≥ 3 步、comparison 要 ≥ 2 个 option |
| 8 | 禁忌词 | 硬 | grep 项目自定义禁忌词列表（合规过度承诺、性能保证用词、竞品名等）|
| 9 | 竞品名 | 硬 | 一组竞品黑名单（项目内部维护） |
| 10 | 占位符残留 | 硬 | grep "POST_" 必须为 0 |
| 11 | 文件大小（hero 图）| 硬 | hero.jpg ≤ 150KB, hero-thumb ≤ 50KB |

**Output 示例**：
```
Quality Gate: 6/7 passed, 1 warning
  ✓ Word count: 1,450 words
  ✓ Headings: 5 <h2> sections
  △ Answer-first: 4/5 sections have concise leads
  ✓ Internal links: 2 found
  ✓ External refs: 3 found
  ✓ CTA present
  ✓ Format (prose): no constraint
→ Proceeding to user review
```

硬检查 fail 会自动重写，最多 2 次循环，否则升级到用户。

⚠️ **Human-in-the-loop 检查点 #2**：草稿 + 评分卡一起给用户看，等待批准。

---

### Stage 3：Hero Image Generation（封面图生成）

这是整个流水线最有趣也最容易翻车的一步。

**Prompt 模板**：
```
Generate a 16:9 editorial illustration for a fintech blog post titled "{title}".
Style: Abstract, conceptual, modern digital art.
Color palette: <BRAND_PRIMARY_HEX>, <BRAND_SECONDARY_HEX>,
               <BRAND_HIGHLIGHT_HEX>, <BRAND_ACCENT_HEX>.
Rules: No text in image. No human faces. No stock photo style.
Theme: {category-specific theme — see table below}
```

**按品类切换主题**：

| 品类 | 主题方向 |
|------|----------|
| Fraud Prevention Insights | Digital security shields, network nodes, alert signals |
| Banking Technology | Server infrastructure, data pipelines, system architecture |
| Southeast Asia Market | Regional maps, cultural motifs, financial networks |
| Founder Stories | Journey paths, building blocks, growth trajectories |

**调用**：
1. 用 `gemini-generate-image` MCP 工具生图（先用 `ToolSearch` 加载 schema）
2. 跑后处理脚本：
   ```bash
   python3 scripts/process_hero_image.py \
       tmp/gemini-output/<filename> \
       site/blog/<slug>/
   ```
3. 这个脚本生成 4 个文件：
   - `hero.jpg`（≤ 150KB，主图）
   - `hero-thumb.jpg`（≤ 50KB，列表卡片用）
   - `hero.webp`（现代浏览器更小更快）
   - `hero-thumb.webp`
4. 验证文件大小（用 `stat -f%z`）
5. 超出预算就回到第 1 步，用更简单的 prompt 重试（少元素、多留白）

⚠️ **Human-in-the-loop 检查点 #3**：用 `open site/blog/<slug>/hero.jpg` 在系统里打开图，用户看，最多重试 3 次。

**为什么人必须看图**：图像生成的失败是"无法被自动识别的"——颜色对了，构图也合理，但和文章主题完全错位（比如写支付欺诈生成了一张抽象艺术品）。这种错位只有人能判断。

---

### Stage 4：HTML Assembly（页面组装）

纯机械步骤，不需要用户确认。

**做什么**：
1. 复制 `site/blog/post-template.html` 到 `site/blog/<slug>/index.html`
2. 用 `Edit (replace_all=true)` 替换所有占位符：
   - `POST_TITLE` → 标题
   - `POST_SLUG` → slug
   - `POST_EXCERPT` → 摘要
   - `POST_DATE` → `YYYY-MM-DD`
   - `POST_DATE_DISPLAY` → `Mon DD, YYYY`
   - `POST_CATEGORY` → 品类显示名
3. 验证：`grep -c "POST_" <file>` 必须返回 0
4. 删掉 `<meta name="robots" content="noindex">`（除非 `--draft`）
5. 给 `<article>` 加 `data-pagefind-body` 属性（让 Pagefind 知道哪部分是正文，搜索时只索引这块）
6. 把 Stage 2 的正文塞进 `<!-- REPLACE THIS SECTION WITH POST CONTENT -->` 和 `<!-- END POST CONTENT -->` 之间

---

### Stage 5：Blog Index Card（列表页插卡）

**做什么**：
1. 读 `templates/card-template.html`（结构如下）：
   ```html
   <a href="/blog/{SLUG}/" class="fade-up blog-card ..." data-category="{CATEGORY_SLUG}">
     <div class="h-48 ... bg-gradient-to-br {GRADIENT}">
       <svg>...{ICON_PATH}...</svg>
       <picture>
         <source srcset="/blog/{SLUG}/hero-thumb.webp" type="image/webp">
         <img src="/blog/{SLUG}/hero-thumb.jpg" ...>
       </picture>
     </div>
     <div class="p-6">
       <span>{CATEGORY_DISPLAY}</span>
       <time datetime="{ISO_DATE}">{DATE_DISPLAY}</time>
       <h2>{TITLE}</h2>
       <p>{EXCERPT}</p>
     </div>
   </a>
   ```
2. 替换占位符 + 按品类映射到 `{CATEGORY_SLUG}` / `{GRADIENT}` / `{ICON_PATH}`：

   | 品类 | gradient | icon |
   |------|----------|------|
   | Fraud Prevention Insights | `from-deep-navy to-ocean` | 盾牌 |
   | Banking Technology | `from-brand-navy to-wave` | 服务器 |
   | Southeast Asia Market | `from-accent-orange to-warm-orange` | 警示三角 |
   | Founder Stories | `from-ocean to-deep-navy` | 闪电 |

3. 把卡片插到 `site/blog/index.html` 的 `<p id="no-results">` 行之后（永远在最前面，作为最新帖）

---

### Stage 6：Sitemap Update（更新站点地图）

**做什么**：
1. 在 `site/sitemap.xml` 的 `</urlset>` 之前插入：
   ```xml
   <url>
     <loc>https://www.run-true.com/blog/<slug>/</loc>
     <lastmod>YYYY-MM-DD</lastmod>
     <changefreq>monthly</changefreq>
     <priority>0.7</priority>
   </url>
   ```
2. 把 `/blog/` 页本身的 `<lastmod>` 也更新到今天

---

### Stage 7：Quality Checklist + Commit + PR

**自动验证清单**（10 项硬检查，任一失败就停）：
- [ ] `grep "POST_"` 返回 0
- [ ] `grep "data-pagefind-body"` ≥ 1
- [ ] `grep "noindex"` 返回 0（非 draft）
- [ ] hero 图 4 个文件都存在（非 `--no-image`）
- [ ] OG meta 指向正确的 hero.jpg
- [ ] 列表卡片已插入
- [ ] sitemap 已更新
- [ ] 文件大小合规
- [ ] `git diff --cached` 不含 .env 或 API key
- [ ] 禁忌词扫描（合规过度承诺、性能保证用词、竞品名）

⚠️ **Human-in-the-loop 检查点 #4**：起本地 server 让用户在浏览器看一眼。
```bash
cd site && python3 -m http.server 8080 &
open "http://localhost:8080/blog/<slug>/"
```

确认后：
```bash
git checkout -b blog/<slug>
git add site/blog/<slug>/ site/blog/index.html site/sitemap.xml
git commit -m "feat(blog): add post — <Title>"
git push -u origin blog/<slug>
gh pr create --title "..." --body "..."
```

⚠️ **Human-in-the-loop 检查点 #5**：把 PR URL 给用户，由用户决定 merge（触发部署）还是再改。

---

### Stage 8：Post-Publish Verification（上线后验证）

PR merge 后：
1. `git checkout main && git pull`
2. `gh run watch` 看 GitHub Actions 部署
3. CI 完成后 `curl -I https://www.run-true.com/blog/<slug>/` 验证 200
4. 同时验证 hero 图也是 200

---

## 五、Human-in-the-Loop（人在回路）的完整地图

整条流水线一共 5 个用户检查点，刻意安排在"机器决策代价最高"的地方：

| # | 阶段 | 用户决定什么 | 跳过的代价 |
|---|------|--------------|------------|
| 1 | Stage 1 大纲 | 文章方向是否正确 | 写错方向 1500 字白费 |
| 2 | Stage 2.5 草稿 | 内容质量是否过关 | 错别字、语气偏差上线 |
| 3 | Stage 3 配图 | 图片是否切题 | 上线后被吐槽配图诡异 |
| 4 | Stage 7 浏览器预览 | 排版是否破版 | 移动端样式可能翻车 |
| 5 | Stage 7 PR merge | 是否真的发布 | 不可逆的"已发布" |

**`--skip-approval` (`-s`) 模式**：
- 把所有 5 个检查点跳过
- Quality gate 硬失败时直接报错而不是问用户
- 适用场景：批量回填旧文、写明显是初稿的草稿、调试 skill 本身
- **不推荐用于真正要上线的内容**——人的眼睛是最后一道质量门

---

## 六、可复用的 vs. 必须自己重写的部分

| 部分 | 复用难度 | 备注 |
|------|----------|------|
| Stage 0–8 的整体框架 | ⭐ 直接抄 | 9 段式流水线对任何博客都适用 |
| Quality gate 的检查思路 | ⭐ 直接抄 | 字数/标题/链接/格式检查通用 |
| Hero 图生成 prompt 模板 | ⭐⭐ 改色板 | 把 RTD 颜色 token 换成你的品牌色 |
| Pillow 后处理脚本 | ⭐ 直接抄 | `process_hero_image.py` 通用 |
| post-template.html | ⭐⭐ 改样式 | 结构通用，CSS 类名换成你的 |
| card-template.html | ⭐⭐ 改样式 | 同上 |
| 品类映射表 | ⭐⭐⭐ 必须改 | 你的博客品类不一样 |
| 禁忌词列表 | ⭐⭐⭐ 必须改 | 你的合规约束不一样（除非业务领域高度相似） |
| Stage 7 commit/PR 流程 | ⭐ 直接抄 | git/gh 命令通用 |
| GitHub Actions deploy.yml | ⭐⭐ 改部署目标 | 框架通用，改 SCP host/path |

---

## 七、给想自己实现一份的人：起步路径

1. **先用最小版本**：只写 Stage 0、1、2、4、7。把 quality gate、配图、sitemap 都先省掉。能从一句话生成一个完整 PR 就算成功。

2. **加 quality gate**：写完前 5 篇博客后，看哪些"低级错误"反复出现（漏链接、字数过少、占位符没替换……）把它们写成自动检查。这是 skill 真正变好的开始。

3. **加配图自动化**：等手工配图变成"每周 1 小时的负担"再加。Gemini / Imagen / DALL·E 都行，但记得做尺寸压缩——浏览器加载速度比图片好看更重要。

4. **加 human-in-the-loop**：每次 skill 自动决策错的地方，就在那里加一个用户检查点。检查点不是越多越好，是 ROI 最高的地方才加。

5. **持续迭代**：skill 是用出来的。每次 fail 都是一次"sk ill 的 issue ticket"。

---

## 八、常见踩坑

| 坑 | 解法 |
|----|------|
| Hero 图首次生成 162KB，超 150KB 预算 | Pillow 脚本里降 quality 到 78 重压一次；prompt 加 "more whitespace, fewer elements" 重生 |
| 模板占位符没替完 | Stage 7 强制 `grep -c "POST_"` 必须为 0，发现就 fail |
| 博客列表卡片插错位置盖了首页 | scp 时只同步 `site/blog/index.html` 到 `public_html/blog/`，**绝不**同步到 `public_html/` 根目录 |
| Quality gate 自动重写陷入死循环 | 限制最多 2 次重写，超了升级到用户 |
| WeasyPrint 生成的 PDF 在 macOS Preview 里乱码 | 改用 Chrome headless 的 `--print-to-pdf`（PDFKit 兼容性问题） |
| Gemini 偶尔生成带文字的图（虽然 prompt 说了 No text）| 生成后人眼检查；模型行为不是 100% 可控 |

---

## 九、衡量 skill 成功的指标

不是"跑通了"，而是：

- **从想法到上线时间** —— 我的目标是 < 30 分钟，目前 15-25 分钟
- **人工介入次数** —— 5 个检查点是上限，少了风险大，多了打断节奏
- **每次跑完用户的"想改的地方"** —— 持续接近 0，说明 skill 已经内化了你的偏好
- **skill 被多少其他 skill 调用** —— 比如 `/rtd-blog` 内部会调用 `/rtd-brand-guidelines`，组合大于堆砌

---

**最后一句话**：写 skill 的本质不是让 AI 替你做事，而是把"你脑子里的工作流程"翻译成 AI 能稳定执行的 SOP。一旦翻译好，下次的认知成本就趋近于零。
