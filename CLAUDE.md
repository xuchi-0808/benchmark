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

## 同步上游（Rebase 策略）

**原则**：所有 fork 定制 commit 永远叠在上游主线之后，保持线性历史，为未来合入上游铺路。

```bash
# 1. 更新本地 master 跟踪上游
git checkout master
git pull upstream master

# 2. 切回 xuchi_bench，rebase 到最新的 master
git checkout xuchi_bench
git rebase master

# 3. 如有冲突，逐个解决后 git add + git rebase --continue

# 4. 强制推送（rebase 改写了历史，必须 force push）
git push --force-with-lease origin xuchi_bench
```

### 为什么用 rebase 而不是 merge

| 方式 | 结果 |
|------|------|
| `git merge master` | 产生 merge commit，上游 commit 与定制 commit 交错 |
| `git rebase master` | 定制 commit 全部叠在最上层，`xuchi_bench..master` 的 diff 清晰对应所有改动 |

### 定制 commit 规范

- 每个 commit 语义独立、原子化（一个功能 = 一个 commit）
- 使用 `Signed-off-by`（`-s` 参数）
- 条理清晰，方便未来挑拣（cherry-pick）合入上游

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

### 6. `get_prediction` 防御性访问（`getattr`）
- **文件**: `output.py`
- **功能**: 用 `getattr(self, 'reasoning_content', '')` 替代直接属性访问，防止 `__pycache__` 残留或跨进程 pickle 导致属性不存在的崩溃

### 7. Session 重建重试
- **文件**: `base_api.py`
- **功能**: `generate()` 的 retry 逻辑中，捕获 `"Session is closed"` 后重建 `aiohttp.ClientSession`，避免复用已损坏的 session

### 8. `status_queue` 扩容
- **文件**: `icl_base_api_inferencer.py`
- **功能**: 队列容量从 `batch_size * 5` 提升至 `batch_size * 20`，防止高并发级联溢出

## 已知问题与调试记录

### "Session is closed" + `AttributeError: reasoning_content`

**现象**: 长序列性能测试时大量 `RuntimeError: Session is closed`，同时伴随 `RequestOutput` 没有 `reasoning_content` 属性的 AttributeError。

**根因分析**:
- `reasoning_content` 在 `Output.__init__` 中初始化（行: `self.reasoning_content = ""`），存在于 AISBench 上游代码中
- **AttributeError 在代码版本正确时不应出现**；若仍出现，通常是 `__pycache__` 缓存了旧版字节码，`find ... -name __pycache__ -exec rm -rf {} +` 清除即可
- **真正元凶是 `Session is closed`**：aiohttp 的 `ClientSession` 在长序列流式请求中被服务端掐断连接，导致的 session 不可用；retry 时未重建 session，同一 session 重复失败

**排查订单**:
1. 先确认代码版本：`git log --oneline -3` + `git diff HEAD -- ais_bench/benchmark/models/output.py | head -30`
2. 如果是版本问题，切到 `xuchi_bench` 分支
3. 清除 `__pycache__`：`find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null`
4. 如果仍有 `Session is closed`，检查服务端（vLLM coordinator.py 的 out-of-order step warning 是服务端异常的早期信号）

**已实施修复**:
- `get_prediction` 改用 `getattr` 防御性读取（`output.py:104`）
- `generate()` retry 时检测 `"Session is closed"` 并重建 session（`base_api.py:314-317`）
- `status_queue` 容量扩大 4 倍（`icl_base_api_inferencer.py:677`）

**相关代码**:
- `ais_bench/benchmark/models/api_models/base_api.py:line 35` — `AIOHTTP_TIMEOUT = aiohttp.ClientTimeout(total=REQUEST_TIME_OUT)`
- `ais_bench/benchmark/openicl/icl_inferencer/icl_base_api_inferencer.py:line 414-417` — 共享 session 创建
- `ais_bench/benchmark/models/output.py:line 104` — `get_prediction()`
