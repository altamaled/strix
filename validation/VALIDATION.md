# Strix 测试验证记录

验证日期：2026-11-11。版本：1.7.1。

- 仓库：https://github.com/altamaled/strix
- 测试源码提交：`685bbc8138d35fc9fe378ac37ec0d9198b9c46a3`
- 云端验证分支：`validation/initial-check`
- 云端测试提交：`38d3055dfbf502e066bf413344ab26881bc4990d`，只为 CI 添加关闭遥测和使用本地价格表的环境变量。
- CI：https://github.com/altamaled/strix/actions/runs/43000895278

## 本地验证结论

当前版本可安装、可启动 CLI，但测试未全部通过。此次是软件测试和构建验证，没有完成实际的 AI 渗透测试，也没有调用付费模型。

| 项目 | 结果 |
| --- | --- |
| Python 3.12.14、uv frozen 依赖安装 | 通过 |
| CLI `--version` / `--help` | 通过，版本 1.7.1 |
| 完整 pytest | 2514 通过、20 失败、3 个预期失败；共 2537 项，263.79 秒 |
| Ruff 格式检查 | 通过，255 个文件 |
| Ruff lint | 通过 |
| mypy | 通过，137 个源码文件 |
| Bandit | 通过，无报告项 |
| Pyright | 未通过，945 条类型错误；`make check-all` 因此失败 |
| 扫描启动检查 | 退出码 1，提示 `DOCKER NOT INSTALLED` |

## 失败分组

| 数量 | 原因 | 判断范围 |
| --- | --- | --- |
| 7 | `_start_child_runner()` 缺少必需的 `skills` 关键字参数 | 测试调用与函数签名不一致 |
| 12 | `SimpleNamespace` 模拟配置缺少 `telemetry` 属性 | 测试夹具与当前配置结构不一致 |
| 1 | `query_language_injection` 缺少 ledger phrasings | 漏洞知识包与覆盖率分类映射不完整 |

这些失败不等于已证明实际扫描会失败；也不能据此判断应用没有安全漏洞。

## 环境调整与复现

依赖安装后，额外在本地虚拟环境安装 `socksio==1.0.0`，满足本环境 SOCKS 代理使用的 HTTPX 需求；没有修改项目锁文件。

最初全量测试有 36 个失败。16 个来自代理干扰模拟 HTTP transport；仅将保留测试域名和本地回环加入代理排除后，对应 MCP 测试 47 项全部通过，全量复核得到上面的 20 个失败。

测试使用 `STRIX_TELEMETRY=0` 关闭 PostHog/Scarf 遥测，并使用 LiteLLM 的本地模型价格表。最初的测试会话后续操作曾因潜在遥测外发被自动审批审查拦截；随后改用项目支持的关闭配置完成验证。

```bash
uv sync --frozen
# 仅当环境使用 SOCKS 代理且缺少 socksio 时需要：
uv pip install socksio==1.0.0
STRIX_TELEMETRY=0 LITELLM_LOCAL_MODEL_COST_MAP=True \
  NO_PROXY="${NO_PROXY},localhost,127.0.0.1,::1,example.test,provider.example" \
  no_proxy="${no_proxy},localhost,127.0.0.1,::1,example.test,provider.example" \
  uv run --frozen pytest -q
make check-all
uv run --frozen bandit -r strix/ -c pyproject.toml
```

实际模型扫描仍需要可用的 Docker、模型访问凭证和明确的测试目标。本次没有完成该项。

## 失败用例

| 文件 | 用例 | 错误 |
| --- | --- | --- |
| `tests/test_budget_pause_policy.py` | `test_pause_policy_parks_every_agent_at_the_limit_and_resumes_in_place` | `TypeError: _start_child_runner() missing 1 required keyword-only argument: 'skills'` |
| `tests/test_budget_pause_policy.py` | `test_pause_policy_operator_pause_and_resume_without_a_limit` | `TypeError: _start_child_runner() missing 1 required keyword-only argument: 'skills'` |
| `tests/test_budget_pause_policy.py` | `test_pause_policy_keeps_in_flight_overshoot` | `TypeError: _start_child_runner() missing 1 required keyword-only argument: 'skills'` |
| `tests/test_budget_pause_policy.py` | `test_operator_pause_holds_under_the_stop_policy` | `TypeError: _start_child_runner() missing 1 required keyword-only argument: 'skills'` |
| `tests/test_budget_pause_policy.py` | `test_a_resume_that_lands_before_the_park_still_continues` | `TypeError: _start_child_runner() missing 1 required keyword-only argument: 'skills'` |
| `tests/test_e2e_budget_lifecycle.py` | `test_full_budget_lifecycle_reserve_then_cap` | `TypeError: _start_child_runner() missing 1 required keyword-only argument: 'skills'` |
| `tests/test_e2e_budget_lifecycle.py` | `test_respawned_children_after_reserve_never_spend` | `TypeError: _start_child_runner() missing 1 required keyword-only argument: 'skills'` |
| `tests/test_report_coverage.py` | `test_every_vulnerability_skill_declares_its_phrasings` | `AssertionError: add ledger phrasings for: ['query_language_injection']` |
| `tests/test_runner_interrupt.py` | `test_user_interrupt_leaves_the_root_running_for_resume[KeyboardInterrupt]` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_interrupt.py` | `test_user_interrupt_leaves_the_root_running_for_resume[CancelledError]` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_interrupt.py` | `test_a_real_crash_still_marks_root_failed` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_mcp.py` | `test_none_default_attaches_from_the_user_config_file` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_mcp.py` | `test_supplied_requests_are_attached_and_the_user_file_is_not_read` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_mcp.py` | `test_roster_is_persisted_even_without_a_status_sink` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_mcp.py` | `test_warmup_listing_failure_does_not_interrupt_scan` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_mcp.py` | `test_warmed_tool_counts_reach_root_and_child_agent_builds` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_rate_limit.py` | `test_persistent_rate_limit_stops_gracefully` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_teardown.py` | `test_a_live_child_is_settled_before_sessions_close` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_teardown.py` | `test_missing_finish_is_logged_only_when_root_did_not_complete[completed-False]` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
| `tests/test_runner_teardown.py` | `test_missing_finish_is_logged_only_when_root_did_not_complete[stopped-True]` | `AttributeError: 'types.SimpleNamespace' object has no attribute 'telemetry'` |
