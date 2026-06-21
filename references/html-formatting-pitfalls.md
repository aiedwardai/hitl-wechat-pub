# HTML 排版坑点与修复方案

实战经验总结（2026-06-21），适用于 HITL 公众号发布流程的 Step 6。

## 坑点 1：Markdown frontmatter 被渲染成正文

YAML frontmatter（`---` 标记）会被 md2wechat CLI 当成正文处理。
解决：生成 HTML 前先去掉 frontmatter。

## 坑点 2：md2wechat 默认主题自动加章节编号

`--theme default` 会添加 "1. "、"2.1 " 等编号，与 emoji 风格冲突。
解决：使用 AI mode 获取设计提示词，手动生成干净 HTML。

## 坑点 3：微信公众号不支持 Markdown

content 字段必须是 HTML，上传 Markdown 会显示原文。
解决：用 LLM 将文章转为纯内联样式 HTML。

## 坑点 4：微信编辑器重置 p 标签颜色

每个 `<p>` 必须显式带 `color` 样式，不能依赖父容器继承。

## 坑点 5：微信不支持 body 标签样式

用 `<div>` 主容器包裹所有内容，不在 `<body>` 上设置样式。