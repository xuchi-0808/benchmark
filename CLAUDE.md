# xuchi_bench — Fork 定制分支

## 仓库配置

- Fork 自: [AISBench/benchmark](https://github.com/AISBench/benchmark)
- 上游 remote: `upstream` (`https://github.com/AISBench/benchmark.git`，只读)
- Fork remote: `origin` (`https://github.com/xuchi-0808/benchmark.git`，推送目标)

## 分支策略

| 分支 | 用途 |
|------|------|
| `master` | 只跟踪 upstream master，从不直接提交 |
| `xuchi_bench` | **主干开发分支**，所有定制都在此分支上开发 |

## 同步上游

```bash
git checkout master
git pull upstream master
git checkout xuchi_bench
git merge master
git push origin xuchi_bench
```

## 本分支已添加的定制

### 1. `response_id` 字段
- **文件**: `output.py`, `vllm_custom_api_chat.py`, `vllm_custom_api.py`, `gen_inferencer_output_handler.py`
- **功能**: 从 API 响应中提取 `id`（如 `chatcmpl-xxx`），存入 `Output.response_id`，写入预测结果 JSONL

### 2. `finish_reason` 字段
- **文件**: `output.py`, `vllm_custom_api_chat.py`, `vllm_custom_api.py`
- **功能**: 从 API 响应中提取 `choices[0].finish_reason`（`"stop"`/`"length"`/`"content_filter"`），存入 `Output.finish_reason`
- `"length"` 表示因 `max_tokens` 限制被截断

### 3. 错误信息增强（`_format_error`）
- **文件**: `base_api.py`
- **功能**: 当 API 返回非 200 状态码时，从 response body 中提取详细错误消息（OpenAI `error.message`、vLLM `detail` 等），替代原先只显示 `response.reason`（"Bad Request"）的垃圾逻辑
- **覆盖路径**: `stream_infer`、`text_infer`、`get_ppl`

### 4. `live_infer.jsonl`
- **文件**: `icl_gen_inferencer.py`
- **功能**: 工作目录下实时逐条追加原始推理结果，`tail -f {work_dir}/live_infer.jsonl` 可直接查看
- **记录所有请求**（成功 + 失败），失败时带 `error_info` 字段
- **格式**: 格式化输出（`indent=2`），每记录间空行分隔
  - 成功: `{"data_abbr", "id", "input", "success": true, "finish_reason", "response_id", "prediction", "input_tokens", "output_tokens", "uuid"}`
  - 失败: `{"data_abbr", "id", "input", "success": false, "response_id", "prediction": "", "error_info", "uuid"}`

### 5. 移除实时打屏
- 终端打印已移除，`live_infer.jsonl` 和 `predictions/*.jsonl` 不受影响

## 已知问题与调试记录

### "Session is closed" + `AttributeError: reasoning_content`

**现象**: 长序列性能测试时大量 `RuntimeError: Session is closed`，同时伴随 `RequestOutput` 没有 `reasoning_content` 属性的 AttributeError。

**根因分析**:
- `reasoning_content` 在 `Output.__init__` 中初始化（行: `self.reasoning_content = ""`），存在于 AISBench 上游代码中
- **AttributeError 是次生错误**：由于代码版本不同步（服务器跑的不是 `xuchi_bench`），旧版本确实没有该字段
- **真正元凶是 `Session is closed`**：aiohttp 的 `ClientSession` 在长序列流式请求中超时关闭，后续请求报 session 关闭，导致最终请求堆积后触发异常

**排查订单**:
1. 先确认代码版本：`git log --oneline -3` + `git diff HEAD -- ais_bench/benchmark/models/output.py | head -30`
2. 如果是版本问题，切到 `xuchi_bench` 分支
3. 如果版本对了但仍有 `Session is closed`，检查 aiohttp timeout 设置是否够大

**相关代码**: `ais_bench/benchmark/models/api_models/base_api.py:line 35` — `AIOHTTP_TIMEOUT = aiohttp.ClientTimeout(total=REQUEST_TIME_OUT)`
