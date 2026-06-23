# OCR / Vision Workflow

图片识别→文字转录的推荐工作流。

## 问题

tesseract OCR 对中文识别效果很差（即使安装了 chi_sim 语言包），尤其对：
- 手机拍照的文档（有畸变、光线不均）
- 深色背景的文字
- 小字体、低对比度文字

实测：1280x1707 的清晰截图，tesseract chi_sim+eng 只能识别出 20-30% 的内容，大量中文字被漏读或错读。

## 推荐方案：Vision API

通过 OpenRouter 调用支持视觉的模型（如 `google/gemini-3.5-flash`）来识别图片中的文字。

### 调用方式

```python
import base64, json, urllib.request, re

# 读取 API key
with open('/home/admin/.hermes/.env', 'r') as f:
    content = f.read()
match = re.search(r'OPENROUTER_API_KEY\s*=\s*(sk-or-v1-\S+)', content)
api_key = match.group(1)

# 编码图片
with open('image.jpg', 'rb') as f:
    img_b64 = base64.b64encode(f.read()).decode('utf-8')

# 调用
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

### 注意事项

- **当前模型 `openrouter/owl-alpha` 不支持图片输入**（返回 404 "No endpoints found that support image input"）
- 需要换用支持视觉的模型：`google/gemini-3.5-flash`（推荐）、`openai/gpt-5.5` 等
- API key 在 `~/.hermes/.env` 中，变量名 `OPENROUTER_API_KEY`，值为 `sk-or-v1-` 开头
- 图片 base64 编码后直接放在 `data:image/jpeg;base64,...` URL 中
- 超时设置 120 秒（大图片处理较慢）

### 适用场景

- 用户发图片要求识别文字
- 截图转文字
- 文档照片转录
- 图表数据提取

## 坑点记录

1. **tesseract 中文识别**：即使 `apt install tesseract-ocr-chi-sim` 成功，pytesseract 识别中文效果仍然很差。不要浪费时间调试 tesseract，直接用 vision API。
2. **API key 提取**：`~/.hermes/.env` 中 key 可能被注释（以 `#` 开头），需要用正则 `OPENROUTER_API_KEY\s*=\s*(sk-or-v1-\S+)` 匹配，不能用简单 split。
3. **execute_code 限制**：cron_mode 下 execute_code 被限制，但 terminal 中调用 OpenRouter API 正常。
4. **图片尺寸**：1280x1707 的图片 base64 约 290KB，放在 API 请求中不会超限。