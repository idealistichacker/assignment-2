**整份作业最省 Modal 额度的策略:**

先提醒一点：ASSIGNMENT.md 提到课程会提供 Modal credits（同时 Modal 免费版也有月度额度），建议先确认课程发的 credits 是否已经领到你当前的 workspace。

结合代码库里各模块的 GPU 规格，按性价比从高到低给你总结 5 个最省钱的做法：

① 集中批处理，减少“冷启动 + 5 分钟空转”次数

每次从 0 唤醒 ValidatorServer（L40S:1），serving.py 都要拉起 vLLM、加载 30B 模型并跑 warmup，再加上结束后 5 分钟的 scaledown_window，每零散唤醒一次都要多付好几分钟的固定 GPU 成本。

做法：尽量把本地代码写好、离线分析做好，凑齐一批集中连续跑；跑完确定短时间不测了，直接去 Dashboard 关掉空闲容器。
② 调试期只跑小子集，别每次全量跑 110 个 seed runs

artifacts/ 里一共有 110 个 runs，baseline.py 对每个正常 run 会串行调 4 次 VLM（全量大约 350–440 次推理）。

做法：
Part 1 先跑完 1 次 完整 baseline，妥善保留 output/baseline_predictions.json（后面分析错误只读本地文件和 seed_labels.json，完全不花钱）。
迭代 validator/solution.py 时，先抽 10–15 个代表性 runs 调通并验证效果，确认靠谱后再跑全量 110 个。
跑全量中途万一断了，务必加 --resume（runner.py 支持断点续跑），只补跑漏掉的 run。
③ 优化 validator/solution.py 本身就能省一大半验证费用
本地代码前置过滤：例如缺失 figure.png、图片损坏/空白等 execution_failure，本地 Python 几毫秒就能判完，不需要调 VLM（既省钱又能避免刚才 004 那种误判）。
减少无效请求与重试：baseline.py 是先直接发超长上下文，等报 BadRequestError 溢出后再多次调 /tokenize 并重发。如果自己提前按字符/长度做合理截断，能省掉大量失败请求和 /tokenize 往返；同时把几个独立维度合并或精简输出 token 数，耗时能降一半以上。
轻度并发吃满单卡：validator_model.py 配置了 --max-num-seqs 2，而 runner.py 是串行跑每个 run 的。如果在 validate(run) 内部把彼此独立的 2 次 VLM 调用并发发出去（并发度 2 即可，过高只是排队），总 GPU 挂钟时间更短，直接按比例省钱。
④ Part 3 的 GLM-4.7（H100）是最烧钱的一环，晚部署、一次过

看 generation_models.py 的 GPU 配置：

QwenServer：L4:1（最便宜）
MinistralServer：L40S:1（中等）
GlmServer：H100:1（最贵），且 infrastructure/README.md 提到首次空缓存下载 ~58 GiB 模型本身就要耗约 12 分钟！

省钱做法：

现在先不要部署 Part 3 的任何服务，等 Part 1 & Part 2 全部搞定、5 个自创 task 写完并通过 uv run python -m workflow validate-tasks 校验后，再部署 generation_models.py 和 agent_runner.py。
验证自创 task 是否有歧义/bug 时，先只跑最便宜的 --models qwen（L4）；全部确认无误后，再一次性运行 uv run python -m workflow generate 补齐 ministral 和 glm（workflow generate 自动跳过已跑完的 qwen）。
这样 H100 只需要冷启动 1 次、连续跑完 5 个 task 就收工；跑完立刻去 Dashboard 停掉 GlmServer 容器，别让 H100 白白空转 5 分钟。
⑤ 哪些工作完全不花 Modal 额度？
阅读轨迹、分析 baseline 错误、对 15 个自创 run 做人工标注（labels.json）、写报告。
整个 Harbor 打包与测试部分（uv run harbor run ...、check-submission）全部跑在本地 Docker，不消耗 Modal 额度，可以放心慢慢调。
等整份作业提交完毕、确认不再使用后，再统一把 Modal 上部署的几个 App stop 掉并清理 Volume，防止后续产生极少量的存储费。

```
Engine 000: Avg prompt throughput: 2801.4 tokens/s, Avg generation throughput: 16.7 tokens/s,
Running: 1 reqs, Waiting: 0 reqs, GPU KV cache usage: 34.3%,
Prefix cache hit rate: 0.4%, MM cache hit rate: 66.7%
```

---

* **`Avg prompt throughput`（2000–4000 tokens/s）** ：每秒读入输入文本（Prefill）的速度，吞吐很高。
* **`Avg generation throughput`（15–37 tokens/s）** ：每秒生成回复 token（Decode）的速度；因为 `baseline` 一次只串行发 1 个请求，这个速度完全正常。
* **`Running: 1 reqs, Waiting: 0 reqs`** ：GPU 上正在算 1 个请求、0 个排队。请求刚结束的瞬间会显示 `Running: 0 reqs`。
* **`GPU KV cache usage: 34.3%`** ：显存里用来存上下文缓存（KV Cache）的占用率，34% 很充裕，不会爆显存。
* **`MM cache hit rate: 66.7%`** ： **多模态图片缓存命中率** 。因为 `baseline` 对同一个 run 会连续问 3–4 个问题，每次都附带同一张 `figure.png`，所以后面几次直接复用了图片特征缓存，省了重复看图的时间。
* **`Prefix cache hit rate: 0.4%`** （顺便送你一个  **Part 2 省钱提速小彩蛋** ！）：
* 文本前缀缓存命中率几乎为 0，是因为 baseline.py 把 **每次都不同的 `question` 写在了 prompt 最开头** ，而把 4 次完全相同的长文本（`Task + Trajectory + Input files`）放在了后面，导致 vLLM 没法复用前缀缓存，每次都要重算整段轨迹！
* 等 Part 2 写 `validator/solution.py` 时，只要把公共的 `Task + Trajectory + Input files` 放前面、把不同问题放最后，就能命中 Prefix Cache，推理更快也更省 Modal 额度。
