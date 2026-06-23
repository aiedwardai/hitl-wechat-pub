# Vision / OCR Workflow — Image to Text

当用户发送图片要求识别文字并整理成文章/报告时，使用以下工作流。

## 问题

- Hermes 工具集中**没有 `vision_analyze`** 工具
- Tesseract OCR（`pytesseract`）对中文屏幕截图/文档照片识别效果很差
- 当前模型（如 openrouter/owl-alpha）可能不直接支持图片输入（返回 404 "No endpoints found that support image input"）

## 解决方案：OpenRouter Vision API via execute_code

通过 `execute_code` 直接调用 OpenRouter 的 chat/completions API，使用支持视觉的模型读图。

### 关键步骤

1. **获取 API Key** — 从 `~/.hermes/.env` 读取 `OPENROUTER_API_KEY`（注意注释掉的行和实际值）
2. **Base64 编码图片** — 读取图片文件并 base64 编码
3. **调用 Vision 模型** — 使用 `google/gemini-3.5-flash`（支持视觉，价格便宜）
4. **解析响应** — 提取 `choices[0].message['content']`

### 代码模板

```python
import base64, json, urllib.request, re

# 1. API Key
with open('/home/admin/.hermes/.env', 'r') as f:
    content = f.read()
match = re.search(r'OPENROUTER_API_KEY\s*=\s*(sk-or-v1-\S+)', content)
api_key = match.group(1)

# 2. Encode image
with open('/path/to/image.jpg', 'rb') as f:
    img_b64 = base64.b64encode(f.read()).decode('utf-8')

# 3. Call vision API
payload = json.dumps({
    "model": "google/gemini-3.5-flash",
    "messages": [{
        "role": "user",
        "content": [
            {"type": "text", "text": "请识别图片中的所有文字，完整转录，包括中文和英文，保持原文格式输出。"},
            {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img_b64}"}}
        ]
    }],
    "max_tokens": 4000
}).encode('utf-8')

req = urllib.request.Request(
    'https://openrouter.ai/api/v1/chat/completions',
    data=payload,
    headers={'Authorization': f'Bearer {api_key}', 'Content-Type': 'application/json'},
    method='POST'
)

with urllib.request.urlopen(req, timeout=120) as resp:
    result = json.loads(resp.read())
    text = result['choices'][0]['message']['content']
    print(text)
```

### 支持视觉的 OpenRouter 模型

| 模型 | 上下文 | 价格 | 备注 |
|------|--------|------|------|
| `google/gemini-3.5-flash` | 1M | $0.0000015 | 推荐，性价比高 |
| `openai/gpt-5.5` | 1M | $0.000005 | 贵但精准 |
| `anthropic/claude-sonnet-4` | — | — | 需确认是否支持图片 |

### 坑点

- ❌ **不要用 tesseract 做中文 OCR** — 对屏幕截图/照片中的中文识别率极低，大量乱码
- ❌ **不要假设当前模型支持视觉** — owl-alpha 不支持图片输入，需换模型
- ❌ **不要尝试 `vision_analyze`** — 工具不存在
- ✅ **图片太大时先压缩** — base64 编码后超过 200KB 可能被截断
- ✅ **指定输出格式** — prompt 中明确说"完整转录""保持原文格式"
- ✅ **整理成结构化文章** — 识别出的文字通常很散，需要按报告格式重组

### 整理模板

识别出文字后，按以下结构整理：
1. **标题** — 根据内容提炼
2. **核心摘要** — 2-3 句话概括
3. **分点/分节** — 按主题分组
4. **数据表格** — 如有数字对比
5. **建议/结论** — 如适用
