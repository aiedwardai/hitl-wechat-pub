---
name: hitl-wechat-pub
description: "Human-in-the-loop controlled WeChat auto publishing skill with manual approval workflow."
version: 2.0.0
author: 16Miku
license: MIT
source: https://github.com/16Miku/wechat-auto-publishing (改编自 wechat-auto-publishing-complete)
metadata:
  hermes:
    tags: [wechat, publishing, automation, hitl, draft]
    related_skills: [md2wechat-lite, wechat-auto-publishing]
---

# hitl-wechat-pub — Human-in-the-Loop 微信公众号发布

在 16Miku/wechat-auto-publishing 基础上，增加了 3 个人工确认节点的 HITL 工作流。

## 核心产出

可复用的 HITL 工作流（三步人工确认）：

1. 准备环境
2. 收集当日源素材
3. **⏸️ 人工确认选题**（HITL #1）
4. 按目标风格撰写文章
5. 准备 cover.png + 内图
6. 组装可发布的包
7. **⏸️ 选择排版格式**（HITL #2）
8. **⏸️ 选择发布公众号**（HITL #3）
9. 生成 HTML → 推送到公众号草稿箱
10. 后台手动发布
11. 归档输出和执行结果
12. 可选添加定时和告警

## 安全规则

永远不要在 skill 包中存储真实私密值，包括：
- 真实的 WECHAT_APP_ID / WECHAT_APP_SECRET
- 真实的 API Key、Cookie、Token
- 用户特定的公众号标识符
- 私有文件系统路径

## Skill 结构

### References（参考文档）

- `references/writing-style-js.md` — **剑识写作风格**（基于16Miku原版改编，有态度/第一人称/无编号/无CTA）
- `references/html-formatting-pitfalls.md` — HTML排版坑点与修复方案（已内置）
- `references/autumn-warm-design-prompt.md` — autumn-warm 🟠 设计规范

### Templates

- 风格文件: `~/styles/剑闻.md`, `剑识.md`, `洞剑.md`, `智剑.md`, `远剑.md`
- 账号配置: `~/.md2wechat/styles.yaml`
- 公众号凭据: `~/.md2wechat/.env.{jdqx|daishou|bushao|jianshi}`

## 标准执行流程

### Step 1: 准备环境
验证运行时依赖、发布脚本依赖链、非敏感配置占位符。

### Step 2: 收集素材
收集原始素材，过滤并压缩。

**⚠️ 坑点**：必须输出**原始池**（8-15条）再压缩，不能跳过。写文章时依赖素材池文件中的数据，不依赖 Agent 上下文中的 web_search 记忆。

**关键产出文件格式：**

文件路径：`~/wechat-publish-{abbr}/step2-source-gathering-YYYY-MM-DD.md`

```
# Step 2: 素材池
# 采集日期: YYYY-MM-DD

## 一、原始素材池

### 🤖 AI · 人工智能

1. **[标题]** — [核心数字/事实]
   - [原文描述，2-3句，含具体数字]
   🔗 [来源名](URL)

(按维度分节，8-15条。每条必须含：标题 / 原文描述 / 来源URL)

## 二、市场角度压缩

| # | 核心观察 | 市场判断 | 情绪钩子 |
...

## 三、精选选题（3-5条）

| # | 选题 | 一句话摘要 | 优先级 | 情绪钩子 |
...

## 四、推荐主线

**[主线候选]**
- 理由

## 五、采集来源汇总

| 维度 | 来源 |
|------|------|
| AI | 来源名 |
...
```

### Step 2.5: 人工确认选题 ⏸️ **HITL #1 — 暂停等待用户**

**这一步必须暂停，等待用户确认后才可继续。** 不要自动跳到 Step 3。

执行方式：
1. Agent 完成 Step 2 后，向用户展示精选选题列表（3-5 条），每条包含：标题 / 一句话摘要 / 优先级 / 情绪钩子
2. 同时展示推荐的主线选题和辅助线选题
3. 使用 `clarify` 工具让用户选择：
   - 用户可从推荐选题中选一条或多条作为主线
   - 用户可提出自己的选题方向（"Other"选项）
   - 用户可要求补充采集更多资讯
4. 用户确认后，Agent 记录确认的选题到素材池文件末尾（`## 确认选题` 段），然后才进入 Step 3

**禁止行为：**
- ❌ 不等用户确认就自动开始写文章
- ❌ 自行决定主线选题跳过人工确认
- ❌ 用默认选题直接推进后续步骤

确认选题记录格式：
```markdown
## 确认选题（用户确认）

- **主线**: [用户选定的主线选题]
- **辅助线**: [用户选定的辅助选题]
- **风格**: [用户指定的写作风格，如剑闻/剑识/洞剑等]
- **确认时间**: YYYY-MM-DD HH:MM
```

### Step 3: 撰写文章

按**用户确认的选题**和**指定风格**撰写文章。

**写作流程（必须按顺序执行，不可跳过）：**

1. **LOAD `writing-styles` skill** — 确认该风格的定位、语调、结构、选材范围、禁忌
2. **READ `~/styles/<风格名>.md`** — 加载具体风格文件（剑闻/剑识/洞剑/智剑/远剑），逐条对照文章结构、标题格式、语调规范
3. **OPEN 原版 article-template.md** — 参考 `~/styles/` 下的风格文件结构
4. **WRITE** — 严格按照风格文件的结构写，不自由发挥
5. **SELF-CHECK** — 每段检查是否符合风格要求（见下方合规清单）

**风格 → 结构速查：**

| 风格 | 结构 | 开头风格 | 人称 |
|------|------|---------|------|
| **剑闻** | 🔥今日重点 → ₿加密 → 🤖AI → 🚀科技 → 🌐宏观 → 💰融资 → 📊数据速览 | 2-3句概括，中性简报 | 第三人称 |
| **剑识** | 第一句抛结论 → ## 1/2/3/4/5分论点 → 我的判断 → 对读者的影响 | 直接给判断/反常识 | 第一人称 |
| **洞剑** | 信号 → 为什么重要 → 接下来看什么 | "注意一个反常信号" | 发现者视角 |
| **智剑** | 自问自答开头 → 1-2-3步骤 → 踩坑提醒 → 友好收尾 | "最近很多人问……" | 口语化第一人称 |
| **远剑** | 主流叙事 → 历史类比 → 未来路径 → 新视角框架 | "把时间拉长到10年" | 冷静理性 |

**每条写作的通用规范：**
- 每条新闻**只给事实 + 一句为什么重要**，不要堆砌数据
- 先理解决策含义，再用目标账号的语音重写
- 每条末尾标注来源：`📎 [来源名](链接)`
- **关键数字加粗**
- 段落 2-4 句，段间空行

**⚠️ 坑点警告：**
- ❌ 不要在 Markdown 文件开头使用 YAML frontmatter（`---` 标记）。微信公众号不支持 frontmatter，会被当成正文渲染。
- ❌ 不要直接粘贴抓取的新闻标题或摘要到正文。先理解事件含义，再用目标账号的语音重写。
- ❌ 不要留"报道："这样的媒体前缀或"..."截断片段。
- ✅ 每段读起来像完整段落，不是素材占位符。

**剑闻写作合规清单（写完后逐条检查）：**
```
□ 标题: 剑闻 | [热点1] · [热点2] · YYYY-MM-DD
□ 开头: 2-3句概括，中性简报，无情绪钩子
□ 板块顺序正确: 🔥→₿→🤖→🚀→🌐→💰→📊
□ 每条 ### 标题
□ 每条 2-4句 + 为什么重要
□ 每条 📎 来源
□ 关键数字加粗
□ 板块间 --- 分隔
□ 📊 市场数据表格收尾
□ 每日22:00前推送 · 戴剑整理
□ 无个人判断/预测
□ 无Slogan/坐右铭
□ 无代码块
```

**剑识写作合规清单：**
```
□ 标题: 剑识 | [核心观点 ≤20字]（无"此刻"）
□ 第一句直接抛判断/反常识观点
□ 无数字编号（不用 ## 1、）
□ 每个数据配一个判断
□ ## 剑识 段收束（非"我的判断"）
□ 结尾为观点判断（非点赞转发CTA）
□ 无教程/代码块/工具推荐
□ 无Slogan/坐右铭
```

### Step 4: 准备图片
准备 cover.png + image1.jpg + image2.jpg（如需）

### Step 5: 组装包
确保包是自包含的：article.md + cover.png + images（如有）

### Step 5.5: 选择排版格式 ⏸️ **HITL #2 — 暂停等待用户**

**这一步必须暂停，让用户选择排版格式。**

执行方式：
1. Agent 向用户展示可选的 AI mode 主题（使用 `clarify` 工具）
2. 用户选择后，Agent 记录选择结果并加载对应设计规范

**可选排版格式（md2wechat-lite AI mode 仅 4 主题，无需 API Key）：**

| 选项 | 色调 | 适用场景 |
|------|------|---------|
| **AI mode - autumn-warm** | 🟠 秋日暖光 | 温暖文艺、个人随笔 |
| **AI mode - spring-fresh** | 🟢 春日清新 | 轻松话题、生活分享 |
| **AI mode - ocean-calm** | 🔵 深蓝静谧 | 科技/金融/数据分析 |
| **AI mode - custom** | 🎨 自定义 | 自由风格 |

**工作方式：**
1. AI mode 返回设计提示词（JSON 格式的 `action_required` 响应）
2. 用 LLM 按提示词生成微信公众号兼容 HTML（纯内联样式）
3. 通过 WeChat API 或 npm CLI `sync-html` 推送草稿

### Step 5.6: 选择发布公众号 ⏸️ **HITL #3 — 暂停等待用户**

**这一步必须暂停，让用户选择目标公众号。**

执行方式：
1. Agent 向用户展示公众号选项（使用 `clarify` 工具）
2. 用户选择后，Agent 加载对应的 `.env` 文件和写作风格

**公众号配置示例（凭据在 skill 包外自行配置）：**

| 公众号 | 缩写 | .env 文件路径 | 适用风格 |
|--------|------|---------------|---------|
| 公众号A | aaa | `~/.md2wechat/.env.aaa` | 剑闻 |
| 公众号B | bbb | `~/.md2wechat/.env.bbb` | 智剑 |
| 公众号C | ccc | `~/.md2wechat/.env.ccc` | 洞剑 |
| 公众号D | ddd | `~/.md2wechat/.env.ddd` | 剑识 |

**选择后执行：**
```bash
source ~/.md2wechat/.env.{abbr}
```

### Step 6: 生成 HTML 并发布到草稿

**⚠️ 关键步骤**：Markdown 不能直接上传到微信公众号，必须转换为 HTML。

**唯一推荐路径（AI mode）：**

```bash
# 1. 获取 AI 设计提示词
source ~/.md2wechat/.env.{abbr}
md2wechat convert article_clean.md --mode ai --theme [用户选择的主题] --title "文章标题(≤32字)"
```

```python
# 2. LLM 按设计提示词生成纯内联样式 HTML（先去 frontmatter）
#    每个 <p> 必须带 color: #4a413d（防微信重置）
#    用 <div> 主容器包裹所有内容（不在 <body> 设样式）
```

```python
# 3. WeChat API 直调上传草稿
import urllib.request, json

token_url = f'https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid={appid}&secret={secret}'
access_token = json.loads(urllib.request.urlopen(token_url).read())['access_token']

# 上传封面 → 创建草稿
articles = [{
    "title": title,
    "author": author,
    "digest": digest,
    "content": html_content,   # 必须是 HTML，非 Markdown
    "thumb_media_id": cover_media_id,
    "need_open_comment": 0,
}]
draft = json.loads(urllib.request.urlopen(
    urllib.request.Request(
        f'https://api.weixin.qq.com/cgi-bin/draft/add?access_token={access_token}',
        data=json.dumps({"articles": articles}, ensure_ascii=False).encode('utf-8'),
        method='POST'
    )
).read())
```

**或 npm CLI 简版：**
```bash
WECHAT_APP_ID="$WECHAT_APPID" WECHAT_APP_SECRET=*** \
  /home/admin/.hermes/node/bin/md2wechat sync-html article_final.html \
    --title "短标题" --author "戴剑" --cover cover.png
```

### Step 7: 正式发布（可选）
个人订阅号限制：API 不支持 freepublish，必须后台手动发布。

### Step 8: 归档结果
保存结果工件、日志、标识符。

### Step 9: 添加定时和告警

## 重要边界

**freepublish 不等于手动发布** — API 发布在首页可见性上可能不等同于后台手动发布。

推荐生产模式：
- **draft_only（推荐）**：自动生成内容 → 自动准备图片 → 自动提交草稿 → **人工在后台发布**
- **full_publish（测试用）**：完全自动化，但需接受 API 发布可能不等同于手动发布

## 多账号部署原则

```
一个账号 = 一个工作目录 = 一个 .env = 一个标题历史 = 一条 cron
```

## 图片验证原则

不要仅信任文件扩展名。发布前验证：
- 文件存在、可读、非空
- 真实格式为 PNG/JPEG/WebP

## 复现检查清单

- [ ] 运行时工具已安装
- [ ] 发布依赖链已安装
- [ ] 安全配置占位符已存在
- [ ] 操作员知道在 skill 包外放置凭据
- [ ] 素材收集路径已定义
- [ ] 文章模板可用
- [ ] 图片策略已定义
- [ ] Markdown → HTML 转换流程已确定（AI mode）
- [ ] 发布检查已定义
- [ ] 结果归档格式已定义
- [ ] 定时示例已存在（如需自动化）

## References（本 skill 内置）

- `references/html-formatting-pitfalls.md` — HTML 排版 5 大坑点及修复方案
- `references/autumn-warm-design-prompt.md` — autumn-warm 🟠 设计规范