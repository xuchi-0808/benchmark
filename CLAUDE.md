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

### 3. 实时打屏
- **文件**: `icl_gen_inferencer.py`
- **功能**: 每次推理成功，终端实时打印 `[dataset:id] [response_id] => prediction`。超过 400 字符时截断为「前 200...后 200」

### 4. `live_infer.jsonl`
- **文件**: `icl_gen_inferencer.py`
- **功能**: 工作目录下实时逐条追加原始推理结果，`tail -f {work_dir}/live_infer.jsonl` 可直接查看
- **记录所有请求**（成功 + 失败），失败时带 `error_info` 字段
- **格式**:
  - 成功: `{"data_abbr", "id", "success": true, "finish_reason", "response_id", "prediction", "input_tokens", "output_tokens", "uuid"}`
  - 失败: `{"data_abbr", "id", "success": false, "response_id", "prediction": "", "error_info", "uuid"}`
