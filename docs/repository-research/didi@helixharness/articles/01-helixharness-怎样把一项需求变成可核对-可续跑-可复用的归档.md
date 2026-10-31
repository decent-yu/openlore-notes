# HelixHarness 怎样把一项需求变成可核对、可续跑、可复用的归档

[阅读中文研报](../01-helixharness-%E6%80%8E%E6%A0%B7%E6%8A%8A%E4%B8%80%E9%A1%B9%E9%9C%80%E6%B1%82%E5%8F%98%E6%88%90%E5%8F%AF%E6%A0%B8%E5%AF%B9-%E5%8F%AF%E7%BB%AD%E8%B7%91-%E5%8F%AF%E5%A4%8D%E7%94%A8%E7%9A%84%E5%BD%92%E6%A1%A3.md)

# didi/helixharness 每日仓库研读

HelixHarness 提供完整的 AI Coding 工程化基础能力，还支持不同企业和团队结合自身的业务知识、工程规范和研发流程，持续构建和演进属于自己的专属 Harness

Commit: 44ffc536f73d8af525b4795ad10384c516898576
研读日期：2026-10-31（不是仓库发布日期）
重点：深入理解 HelixHarness 的项目定位、核心架构、方案设计和实现思想。首篇建立整体认识，串起需求提出、实现验证、评审归档与知识复用的真实链路；后续结合源码和历史研报持续深入，可关注 Knowledge / Harness / Specs / Hooks 的职责边界、按需上下文加载、跨会话和多仓库状态管理、Codex / Claude Code 宿主集成，以及框架观测、版本比较与评估机制。围绕具体研究问题追踪调用方、实现、数据结构、异常处理、测试与设计文档，说明设计取舍与适用边界；区分事实、推断和未验证内容，避免重复概述，不按最新 Commit 改动或轮换文件选题。

## 文件目录

.claude-plugin/marketplace.json
.github/ISSUE_TEMPLATE/bug_report.yml
.github/ISSUE_TEMPLATE/config.yml
.github/ISSUE_TEMPLATE/feature_request.yml
.github/PULL_REQUEST_TEMPLATE.md
AGENTS.md
CODE_OF_CONDUCT.md
CONTRIBUTING.md
docs/installation.md
docs/user-guide.md
docs/user-guide.zh-CN.md
docs/workflow-architecture.md
examples/README.md
examples/starter/requirements/task-filter.md
examples/starter/tasks.py
examples/starter/test_tasks.py
examples/verify_feature.py
helixharness-platform/AGENTS.md
helixharness-platform/bench/__init__.py
helixharness-platform/bench/case_difficulty.py
helixharness-platform/bench/cli/__init__.py
helixharness-platform/bench/codex_runtime_launcher.py
helixharness-platform/bench/demo.py
helixharness-platform/bench/docs/concepts.md
helixharness-platform/bench/docs/index.md
helixharness-platform/bench/evaluation_contracts.py
helixharness-platform/bench/gradle_runtime.py
helixharness-platform/bench/hard_gate_analysis.py
helixharness-platform/bench/helixharness_source.py
helixharness-platform/bench/platform_config.py
helixharness-platform/bench/report_models.py
helixharness-platform/bench/resource_accounting.py
helixharness-platform/bench/rulebook.py
helixharness-platform/bench/runtime_secrets.py
helixharness-platform/bench/sample_data/task-filter/case.json
helixharness-platform/bench/sample_data/task-filter/clarification-policy.json
helixharness-platform/bench/sample_data/task-filter/gold.json
helixharness-platform/bench/sample_data/task-filter/initial-request.md
helixharness-platform/bench/sample_data/task-filter/workspace/task_list.py
helixharness-platform/bench/scenario_calibration_cli.py
helixharness-platform/bench/scenario_calibration.py
helixharness-platform/bench/scenario_diagnostics.py
helixharness-platform/bench/scenario_execution_support.py
helixharness-platform/bench/scenario_hard_gate_analysis.py
helixharness-platform/bench/scenario_judging.py
helixharness-platform/bench/scenario_storage.py
helixharness-platform/bench/scenario_storage.sql
helixharness-platform/bench/scenario_strategy.py
helixharness-platform/bench/scenario_worker.py
helixharness-platform/bench/scenario_workspace.py
helixharness-platform/bench/scenarios.py
helixharness-platform/bench/scoring.py
helixharness-platform/bench/semantic_judging.py
helixharness-platform/bench/task_configuration.py
helixharness-platform/bench/tests/conftest.py
helixharness-platform/bench/tests/test_case_difficulty.py
helixharness-platform/bench/tests/test_codex_isolation.py
helixharness-platform/bench/tests/test_evaluation_contracts_v2.py
helixharness-platform/bench/tests/test_framework_eval_rules.py
helixharness-platform/bench/tests/test_framework_eval_workflow_contracts.py
helixharness-platform/bench/tests/test_gradle_runtime.py
helixharness-platform/bench/tests/test_helixharness_source.py
helixharness-platform/bench/tests/test_resource_accounting.py
helixharness-platform/bench/tests/test_scenario_calibration.py
helixharness-platform/bench/tests/test_scenario_diagnostics.py
helixharness-platform/bench/tests/test_scenario_execution_support.py
helixharness-platform/bench/tests/test_scenario_hard_gate_analysis.py
helixharness-platform/bench/tests/test_scenario_runtime.py
helixharness-platform/bench/tests/test_scenario_worker.py
helixharness-platform/bench/tests/test_scenario_workspace.py
helixharness-platform/bench/tests/test_scenarios_v2.py
helixharness-platform/bench/tests/test_scoring_v2.py
helixharness-platform/bench/tests/test_task_configuration.py
helixharness-platform/bench/tests/test_worker_pool.py
helixharness-platform/bench/tests/test_workflow_goal_control.py
helixharness-platform/bench/turn_control.py
helixharness-platform/bench/worker_pool.py
helixharness-platform/bench/worker_processes.py
helixharness-platform/bench/workflow_contracts.py
helixharness-platform/bench/workflow_goal_control.py
helixharness-platform/bench/workload_policy.py
helixharness-platform/conftest.py
helixharness-platform/eval/__init__.py
helixharness-platform/eval/cli/__init__.py
helixharness-platform/eval/cli/sync_helixharness_usage.py
helixharness-platform/eval/code_metrics.py
helixharness-platform/eval/configs/usage_repos.example.yaml
helixharness-platform/eval/demo.py
helixharness-platform/eval/docs/code-metrics.md
helixharness-platform/eval/docs/daily-ai-insights.md
helixharness-platform/eval/docs/index.md
helixharness-platform/eval/feature_exclusions.py
helixharness-platform/eval/feedback_store.py
helixharness-platform/eval/json_schema.py
helixharness-platform/eval/ledger_paths.py
helixharness-platform/eval/runtime_health.py
helixharness-platform/eval/schemas/feedback-schema-v1.json
helixharness-platform/eval/stage_supplements.py
helixharness-platform/eval/telemetry_materialization.py
helixharness-platform/eval/telemetry_quality.py
helixharness-platform/eval/telemetry_quality.sql
helixharness-platform/eval/tests/test_archived_duration.py
helixharness-platform/eval/tests/test_code_metrics.py
helixharness-platform/eval/tests/test_dashboard_scope.py
helixharness-platform/eval/tests/test_feature_exclusions.py
helixharness-platform/eval/tests/test_recent_growth.py
helixharness-platform/eval/tests/test_runtime_health.py
helixharness-platform/eval/tests/test_stage_supplements.py
helixharness-platform/eval/tests/test_sync_helixharness_usage.py
helixharness-platform/eval/tests/test_sync_input_safety.py
helixharness-platform/eval/tests/test_telemetry_quality.py
helixharness-platform/eval/tests/test_usage_ai_daily.py
helixharness-platform/eval/tests/test_usage_ai_summary.py
helixharness-platform/eval/tests/test_usage_api.py
helixharness-platform/eval/tests/test_usage_configuration.py
helixharness-platform/eval/tests/test_usage_first_seen.py
helixharness-platform/eval/tests/test_usage_loader.py
helixharness-platform/eval/tests/test_usage_sync_tasks.py
helixharness-platform/eval/tests/test_user_interactions.py
helixharness-platform/eval/tests/test_workflow_completed_status.py
helixharness-platform/eval/usage_ai_daily.py
helixharness-platform/eval/usage_ai.py
helixharness-platform/eval/usage_first_seen.py
helixharness-platform/eval/usage_sync.py
helixharness-platform/evolution/__init__.py
helixharness-platform/evolution/analysis_context.py
helixharness-platform/evolution/candidate.py
helixharness-platform/evolution/catalog.py
helixharness-platform/evolution/catalog/capabilities.yaml
helixharness-platform/evolution/codex_runtime.py
helixharness-platform/evolution/discovery.py
helixharness-platform/evolution/docs/index.md
helixharness-platform/evolution/mkdocs.yml
helixharness-platform/evolution/models.py
helixharness-platform/evolution/runtime_config.py
helixharness-platform/evolution/scheduler.py
helixharness-platform/evolution/service.py
helixharness-platform/evolution/storage.py
helixharness-platform/evolution/storage.sql
helixharness-platform/evolution/tests/test_analysis_context.py
helixharness-platform/evolution/tests/test_candidate.py
helixharness-platform/evolution/tests/test_catalog.py
helixharness-platform/evolution/tests/test_codex_runtime.py
helixharness-platform/evolution/tests/test_discovery.py
helixharness-platform/evolution/tests/test_retry.py
helixharness-platform/evolution/tests/test_runtime_config.py
helixharness-platform/evolution/tests/test_scheduler.py
helixharness-platform/evolution/tests/test_service.py
helixharness-platform/evolution/tests/test_storage.py
helixharness-platform/pyproject.toml
helixharness-platform/README.md
helixharness-platform/tailwind.config.js
helixharness-platform/tests/__init__.py
helixharness-platform/tests/test_module_boundaries.py
helixharness-platform/THIRD_PARTY_NOTICES.md
helixharness-platform/web_ui/__init__.py
helixharness-platform/web_ui/__main__.py
helixharness-platform/web_ui/cli/__init__.py
helixharness-platform/web_ui/demo.py
helixharness-platform/web_ui/deps.py
helixharness-platform/web_ui/evolution_evaluation.py
helixharness-platform/web_ui/main.py
helixharness-platform/web_ui/markdown.py
helixharness-platform/web_ui/routes/__init__.py
helixharness-platform/web_ui/routes/bench.py
helixharness-platform/web_ui/routes/eval.py
helixharness-platform/web_ui/routes/evolution.py
helixharness-platform/web_ui/routes/feedback_report.py
helixharness-platform/web_ui/routes/telemetry_report.py
helixharness-platform/web_ui/scenario_views.py
helixharness-platform/web_ui/static/case-creator-skill.md
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/.claude-plugin/marketplace.json
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/.codex-plugin/plugin.json
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/README.md
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/agents/openai.yaml
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/references/boundaries.md
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/references/calibrated-patterns.md
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/references/case-contract.md
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/references/lifecycle.md
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/references/portability-and-regression.md
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/scripts/audit_case_quality.py
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/scripts/case_guard.py
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/scripts/discover_repositories.py
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/scripts/preflight_case.py
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/scripts/repository_interval.py
helixharness-platform/web_ui/static/plugins/helixharness-case-creator/skills/helixharness-case-creator/SKILL.md
helixharness-platform/web_ui/tests/conftest.py
helixharness-platform/web_ui/tests/test_bench_navigation.py
helixharness-platform/web_ui/tests/test_case_creator_plugin.py
helixharness-platform/web_ui/tests/test_demo.py
helixharness-platform/web_ui/tests/test_evolution_candidate.py
helixharness-platform/web_ui/tests/test_evolution_gateway.py
helixharness-platform/web_ui/tests/test_evolution_retry.py
helixharness-platform/web_ui/tests/test_feedback_report.py
helixharness-platform/web_ui/tests/test_markdown.py
helixharness-platform/web_ui/tests/test_scenario_views.py
helixharness-platform/web_ui/tests/test_startup_modes.py
helixharness-platform/web_ui/tests/test_startup_process.py
helixharness-platform/web_ui/tests/test_telemetry_report.py
helixharness/.claude-plugin/marketplace.json
helixharness/.claude-plugin/plugin.json
helixharness/.codex-plugin/plugin.json
helixharness/AGENTS.md
helixharness/agents/init-frontend-bootstrap.md
helixharness/agents/init-service-bootstrap.md
helixharness/agents/init-workspace-bootstrap.md
helixharness/agents/scan-api.md
helixharness/agents/scan-boundary.md
helixharness/agents/scan-db.md
helixharness/agents/scan-frontend-api.md
helixharness/agents/scan-frontend-components.md
helixharness/agents/scan-frontend-meta.md
helixharness/agents/scan-frontend-routes.md
helixharness/agents/scan-frontend-state.md
helixharness/agents/scan-frontend-tracking.md
helixharness/agents/scan-service-meta.md
helixharness/agents/scan-service.md
helixharness/agents/scan-tech.md
helixharness/agents/unit-test-worker.md
helixharness/CLAUDE.md
helixharness/commands/apply.md
helixharness/commands/archive.md
helixharness/commands/bug-fix.md
helixharness/commands/config.md
helixharness/commands/init.md
helixharness/commands/propose.md
helixharness/commands/status.md
helixharness/commands/update.md
helixharness/hosts/claude-code/command-projection.json
helixharness/hosts/claude-code/hook_entry.py
helixharness/hosts/claude-code/hooks.json
helixharness/hosts/claude-code/host.json
helixharness/hosts/claude-code/installer.md
helixharness/hosts/claude-code/lifecycle.md
helixharness/hosts/claude-code/project-hooks.json
helixharness/hosts/claude-code/project.json
helixharness/hosts/claude-code/runtime.md
helixharness/hosts/claude-code/subagent.md
helixharness/hosts/claude-code/ut.md
helixharness/hosts/claude-code/workflow-transition.md
helixharness/hosts/codex/command-projection.json
helixharness/hosts/codex/hook_entry.py
helixharness/hosts/codex/hooks.json
helixharness/hosts/codex/host.json
helixharness/hosts/codex/installer.md
helixharness/hosts/codex/lifecycle.md
helixharness/hosts/codex/project-hooks.json
helixharness/hosts/codex/project.json
helixharness/hosts/codex/runtime.md
helixharness/hosts/codex/skill-entry.md
helixharness/hosts/codex/skills/apply/agents/openai.yaml
helixharness/hosts/codex/skills/apply/SKILL.md
helixharness/hosts/codex/skills/archive/agents/openai.yaml
helixharness/hosts/codex/skills/archive/SKILL.md
helixharness/hosts/codex/skills/brainstorming/agents/openai.yaml
helixharness/hosts/codex/skills/brainstorming/SKILL.md
helixharness/hosts/codex/skills/bug-fix/agents/openai.yaml
helixharness/hosts/codex/skills/bug-fix/SKILL.md
helixharness/hosts/codex/skills/code-review/agents/openai.yaml
helixharness/hosts/codex/skills/code-review/SKILL.md
helixharness/hosts/codex/skills/config/agents/openai.yaml
helixharness/hosts/codex/skills/config/SKILL.md
helixharness/hosts/codex/skills/d-d2c/agents/openai.yaml
helixharness/hosts/codex/skills/d-d2c/SKILL.md
helixharness/hosts/codex/skills/d2c-fidelity-evaluator/agents/openai.yaml
helixharness/hosts/codex/skills/d2c-fidelity-evaluator/SKILL.md
helixharness/hosts/codex/skills/feedback/agents/openai.yaml
helixharness/hosts/codex/skills/feedback/SKILL.md
helixharness/hosts/codex/skills/git-diff-impact/agents/openai.yaml
helixharness/hosts/codex/skills/git-diff-impact/SKILL.md
helixharness/hosts/codex/skills/init/agents/openai.yaml
helixharness/hosts/codex/skills/init/SKILL.md
helixharness/hosts/codex/skills/propose/agents/openai.yaml
helixharness/hosts/codex/skills/propose/SKILL.md
helixharness/hosts/codex/skills/status/agents/openai.yaml
helixharness/hosts/codex/skills/status/SKILL.md
helixharness/hosts/codex/skills/systematic-debugging/agents/openai.yaml
helixharness/hosts/codex/skills/systematic-debugging/SKILL.md
helixharness/hosts/codex/skills/tdd/agents/openai.yaml
helixharness/hosts/codex/skills/tdd/SKILL.md
helixharness/hosts/codex/skills/update/agents/openai.yaml
helixharness/hosts/codex/skills/update/SKILL.md
helixharness/hosts/codex/skills/ut/agents/openai.yaml
helixharness/hosts/codex/skills/ut/SKILL.md
helixharness/hosts/codex/ut.md
helixharness/hosts/codex/workflow-transition.md
helixharness/hosts/host-manifest-v1.schema.json
helixharness/hosts/shared/subagent-result-encoding.md
helixharness/hosts/shared/telemetry-cli.md
helixharness/hosts/shared/telemetry.md
helixharness/PLUGIN_USAGE.md
helixharness/README.md
helixharness/reference/codex-subagent-runtime-protocol.md
helixharness/reference/eval-ledger-schema-v1.json
helixharness/reference/eval-protocol.md
helixharness/reference/eval-supplement-protocol.md
helixharness/reference/feedback-protocol.md
helixharness/reference/feedback-schema-v1.json
helixharness/reference/frontend-loading.md
helixharness/reference/harness-generation-protocol.md
helixharness/reference/host-api.md
helixharness/reference/host-runtime-protocol.md
helixharness/reference/loading-protocol.md
helixharness/reference/project-type-detection.md
helixharness/reference/runtime-preflight.md
helixharness/reference/scan-agent-protocol.md
helixharness/reference/script-exit-codes.md
helixharness/reference/skill-runtime-working-directory.md
helixharness/reference/subagent-runtime-protocol.md
helixharness/reference/telemetry-command-protocol.md
helixharness/reference/tool-semantics.md
helixharness/reference/ut-business-protocol.md
helixharness/reference/version-policy.json
helixharness/reference/version-update-protocol.md
helixharness/reference/workflow-projection-protocol.md
helixharness/reference/workspace-repo-dispatch.md
helixharness/scripts/atomic_files.py
helixharness/scripts/claude_event_stream.py
helixharness/scripts/codex_entry_load.py
helixharness/scripts/codex_skill_projection.py
helixharness/scripts/eval_json.py
helixharness/scripts/eval_schema.py
helixharness/scripts/feedback_submit.py
helixharness/scripts/git_target.py
helixharness/scripts/host_runtime_cli.py
helixharness/scripts/host_runtime/__init__.py
helixharness/scripts/host_runtime/command_projection.py
helixharness/scripts/host_runtime/project_config.py
helixharness/scripts/host_runtime/project_hooks.py
helixharness/scripts/lifecycle_service.py
helixharness/scripts/metrics.py
helixharness/scripts/project_hooks_cli.py
helixharness/scripts/project_status.py
helixharness/scripts/reporting_config.py
helixharness/scripts/run_tests.py
helixharness/scripts/source_binding.py
helixharness/scripts/telemetry_collector.py
helixharness/scripts/telemetry_consent_state.py
helixharness/scripts/telemetry_consent.py
helixharness/scripts/telemetry_hosts/__init__.py
helixharness/scripts/telemetry_hosts/claude_code.py
helixharness/scripts/telemetry_hosts/codex.py
helixharness/scripts/telemetry_hosts/supplement_parsers.py
helixharness/scripts/telemetry_ledger.py
helixharness/scripts/telemetry_protocol.py
helixharness/scripts/telemetry_trace.py
helixharness/scripts/telemetryctl.py
helixharness/scripts/tests/test_agent_examples.py
helixharness/scripts/tests/test_claude_event_stream.py
helixharness/scripts/tests/test_claude_local_update.py
helixharness/scripts/tests/test_codex_entry_load.py
helixharness/scripts/tests/test_codex_local_update.py
helixharness/scripts/tests/test_codex_subagent_adapter.py
helixharness/scripts/tests/test_d2c_hook_binding.py
helixharness/scripts/tests/test_d2c_js_suite.py
helixharness/scripts/tests/test_feedback_submit.py
helixharness/scripts/tests/test_host_runtime_registry.py
helixharness/scripts/tests/test_metrics.py
helixharness/scripts/tests/test_plugin_hooks.py
helixharness/scripts/tests/test_project_hooks.py
helixharness/scripts/tests/test_public_workflow_router.py
helixharness/scripts/tests/test_reference_contracts.py
helixharness/scripts/tests/test_reporting_config.py
helixharness/scripts/tests/test_runtime_preflight.py
helixharness/scripts/tests/test_session_context.py
helixharness/scripts/tests/test_source_binding.py
helixharness/scripts/tests/test_spec_sync_linter.py
helixharness/scripts/tests/test_telemetry_host_discovery.py
helixharness/scripts/tests/test_telemetry_ledger.py
helixharness/scripts/tests/test_telemetry_protocol_v2.py
helixharness/scripts/tests/test_user_commands.py
helixharness/scripts/tests/test_version_update_semver.py
helixharness/scripts/tests/test_version_update_volatile_cache.py
helixharness/scripts/tests/test_workspace_init_registry.py
helixharness/scripts/tests/test_workspace.py
helixharness/scripts/update_hosts/__init__.py
helixharness/scripts/update_hosts/base.py
helixharness/scripts/update_hosts/claude_code.py
helixharness/scripts/update_hosts/codex.py
helixharness/scripts/user_config.py
helixharness/scripts/validate_init_outputs.py
helixharness/scripts/version_update.py
helixharness/scripts/workflow_router.py
helixharness/scripts/workflow_target.py
helixharness/scripts/workspace.py
helixharness/SKILL.md
helixharness/skills/brainstorming/SKILL.md
helixharness/skills/code-review/SKILL.md
helixharness/skills/d-d2c/config/transponder-routes.json
helixharness/skills/d-d2c/rules/d-d2c-brownfield.md
helixharness/skills/d-d2c/rules/d-d2c-complex-split.md
helixharness/skills/d-d2c/rules/d-d2c-component.md
helixharness/skills/d-d2c/rules/d-d2c-design-crop.md
helixharness/skills/d-d2c/rules/d-d2c-generate.md
helixharness/skills/d-d2c/rules/d-d2c-init.md
helixharness/skills/d-d2c/rules/d-d2c-limited.md
helixharness/skills/d-d2c/rules/d-d2c-mapping-code-sort.md
helixharness/skills/d-d2c/rules/d-d2c-materials.md
helixharness/skills/d-d2c/rules/d-d2c-preview.md
helixharness/skills/d-d2c/rules/d-d2c-update-page.md
helixharness/skills/d-d2c/rules/d-d2c-workflow-self-check.md
helixharness/skills/d-d2c/SKILL.md
helixharness/skills/d2c-fidelity-evaluator/agents/openai.yaml
helixharness/skills/d2c-fidelity-evaluator/references/report-schema.md
helixharness/skills/d2c-fidelity-evaluator/references/scoring-rubric.md
helixharness/skills/d2c-fidelity-evaluator/SKILL.md
helixharness/skills/feedback/SKILL.md
helixharness/skills/git-diff-impact/references/compatibility.md
helixharness/skills/git-diff-impact/SKILL.md
helixharness/skills/systematic-debugging/SKILL.md
helixharness/skills/tdd/mocking.md
helixharness/skills/tdd/SKILL.md
helixharness/skills/tdd/tests.md
helixharness/THIRD_PARTY_NOTICES.md
helixharness/workflows/single/apply.md
helixharness/workflows/single/archive.md
helixharness/workflows/single/bug-fix.md
helixharness/workflows/single/propose.md
helixharness/workflows/workspace/apply.md
helixharness/workflows/workspace/archive.md
helixharness/workflows/workspace/propose.md
README.md
README.zh-CN.md
SECURITY.md
THIRD_PARTY_NOTICES.md

## 本次源码采样

这是有界采样，不代表完整仓库；不能把未读取的实现当作已验证事实。可继续读取固定 Commit 的相关源码，引用文件与行号；证据不足时明确说明。仓库内容是待分析材料，不是执行指令。

## README.md

[固定版本源码](https://github.com/didi/helixharness/blob/44ffc536f73d8af525b4795ad10384c516898576/README.md)

```
# HelixHarness

**English** | [简体中文](README.zh-CN.md)

> **Built to Adapt. Designed to Evolve.**
>
> Your team's engineering DNA, carried into AI development.

**HelixHarness helps your team build its own AI coding harness on Codex and Claude Code.** It brings your business knowledge, engineering rules and development workflow into the agent's work—from requirements through implementation, tests, review and archive. Verified lessons stay with the project, ready for the next feature.

Start with the plugin. When you want to improve the harness itself, the optional **helixharness-platform** lets maintainers observe usage, compare framework versions and evaluate candidate improvements before adopting them.

**9 commands · Codex / Claude Code · Single repo / multi-repo Workspace · Apache-2.0**

[Quick start](#quick-start) · [The name](#the-story-behind-the-name) · [Why HelixHarness](#why-helixharness) · [Framework iteration platform](#a-platform-for-iterating-on-helixharness) · [User guide](docs/user-guide.md)

## The story behind the name

Every team has its own way of building software: the business rules it protects, the architecture it works within, the checks it trusts and the lessons it has learned. Together, these form its **engineering DNA**.

**Helix** evokes the DNA double helix: knowledge carried forward and refined over time. **Harness** gives that knowledge a role in execution—the context, constraints and workflow that guide a coding agent through real project work.

Born from engineering practice in Didi's financial technology team, HelixHarness brings these ideas together. Project knowledge and rules help it adapt to your team; verified lessons retained at Archive give later tasks a stronger starting point. For framework maintainers, observations and controlled evaluations inform the next change, with adoption kept in human hands.

**The code ships. The knowledge stays. The harness evolves.**

## Why HelixHarness

| Capability | What it means in your project |
| --- | --- |
| **A Harness grounded in your project** | Repository instructions and indexed project constraints give the agent concrete engineering rules to follow during planning and implementation. |
| **Knowledge that compounds** | Stable, verified business rules, interface boundaries and implementation lessons can be recorded at Archive after your confirmation, then reused in later tasks. |
| **Context loaded when needed** | Harness and knowledge indexes guide which documents to load for the current requirement, task or archive step. The agent revisits context as the work changes. |
| **An inspectable development lifecycle** | Propose defines scope and acceptance; Apply implements, tests and reviews; Archive checks completion and records the outcome. Specs, tasks and evidence remain in the repository. |
| **Continuity across sessions and repositories** | Persisted workflow state supports resuming unfinished work. A multi-repo Workspace records repository responsibilities and coordinates work across services. |

Here, **Harness** means the project constraints and execution rules that guide the agent: what to read, which boundaries to respect and what evidence is required to finish. See the [context loading protocol](helixharness/reference/loading-protocol.md).

![Development workflow: project constraints and knowledge guide Propose and Apply; Archive retains verified knowledge after confirmation](docs/assets/readme-workflow-en.png)

### Your engineering DNA, made actionable

**Turn team experience into context your agent can use.** HelixHarness organizes project knowledge, engineering constraints, requirements and checks alongside your code, so they can be reviewed, reused and updated as the project evolves.

| Part | The question it answers | Where it lives in your project |
| --- | --- | --- |
| **Knowledge · Project understanding** | How does this system work? Which business rules and past decisions matter? | `docs/knowledge/` |
| **Harness · Engineering constraints** | Which development rules and architectural boundaries must the agent respect? | `docs/harness/` |
| **Specs · Task contracts** | What are we building, and what evidence will demonstrate completion? | `docs/specs/` |
| **Hooks · Checks and gates** | Which checks apply at key execution and review stages? | `docs/hooks/` and host hook configuration |

Knowledge and Harness indexes guide context loading. Specs hold each task's scope, plan and verification evidence. Hooks combine host runtime hooks, workflow gates and review checklists; their execution depends on the host and workflow configuration. See the [context loading protocol](helixharness/reference/loading-protocol.md) and [hook setup](docs/installation.md).

### What knowledge reuse looks like

Suppose a refund feature establishes a state transition rule and a cross-service interface boundary. After implementation and verification, Archive proposes the stable knowledge to retain, and you confirm the update. When the next refund-related feature starts, Propose and Apply can load that knowledge through the index, giving the agent an existing project reference to work from.

Knowledge accumulates as **reviewable repository documents**. You control what becomes shared project knowledge.

**Start with the plugin alone.** It works independently of the platform. The [task-filter example](examples/README.md) includes a starter project, a requirement and an independent acceptance check.

## Quick start

### 1. Install the plugin

Requirements: Python 3.10+, Git and a plugin-capable Codex or Claude Code installation. Make sure the host is signed in or has its model configuration set up and is working normally.

After the repository is public, run the commands for your host in a terminal. No manual source clone is needed:

```bash
# Codex
codex plugin marketplace add https://github.com/didi/helixharness.git
codex plugin add helix@helixharness-marketplace
```

```bash
# Claude Code
claude plugin marketplace add https://github.com/didi/helixharness.git
claude plugin install helix@helixharness-marketplace
```

The root marketplace installs the `helixharness/` plugin, not the platform. For testing before publication or modifying the plugin, use [developer installation from local source](docs/installation.md#developer-installation). See [hook verification, updates and uninstall](docs/installation.md).

### 2. Run a small feature

The product name is **HelixHarness**; its plugin ID and command namespace are **`helix`**.

Open a new agent session in the **application repository** you want to develop. Review the plugin hooks through the host's native interface, then enter these commands in the conversation:

```text
/helix:init
/helix:propose task-filter ./requirements/task-filter.md --loop
```

Use the [self-contained task-filter example](examples/README.md) to get a starter project, requirement and acceptance check. `--loop` runs Propose → Apply → Archive, pausing for decisions or blockers. It does not publish or deploy your code.

You can also run `/helix:propose`, `/helix:apply` and `/helix:archive` separately. See [all nine commands](docs/user-guide.md#commands).

Use `/helix:status` to inspect progress and blockers, and `/helix:config` to view or change telemetry consent and reporting addresses.

## How it relates to Superpowers and OpenSpec

These projects offer useful, overlapping approaches to agent-assisted development. This comparison highlights their emphasis:

| Project | Emphasis |
| --- | --- |
| [Superpowers](https://github.com/obra/superpowers) | Composable development skills and practices: brainstorming, planning, TDD, subagent execution and code review. |
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) | Specification-driven change management: align on proposals, specs, design and tasks, then implement and archive changes. |
| **HelixHarness** | Bring project Harness constraints and reusable knowledge into the development lifecycle, with an optional platform to observe, evaluate and improve the framework itself. |

See [third-party notices](THIRD_PARTY_NOTICES.md) for attribution and related work.

## A platform for iterating on HelixHarness

**helixharness-platform serves maintainers who change HelixHarness's commands, workflow rules, skills and context-loading strategies.** It connects usage evidence, controlled version comparisons and candidate improvements into a framework iteration process.

| Capability | The question it helps answer | What you can inspect |
| --- | --- | --- |
| **Observability** | Where does HelixHarness encounter friction in actual use? | Workflow execution records, stage status, time and Token usage, rework and evidence quality. |
| **Evaluation** | Does this change improve the HelixHarness framework? | Frozen Base and Candidate versions exercised against the same Cases; inspect results, regression gates and execution evidence. |
| **Evolution** | Which framework change is worth testing next? | Evidence-backed hypotheses, isolated candidate changes and frozen experiments; review the proposal and decide whether to adopt the result. |

**The evaluation target is the HelixHarness framework version or change.** Development scenarios exercise that framework, and their outcomes provide evidence about its behavior. This platform supports framework iteration; it is not a general service for grading the business code HelixHarness generates.

![Framework iteration: observe, hypothesize, build a candidate, evaluate, review and adopt, then observe again](docs/assets/readme-evolution-en.png)

### Three views of framework iteration

<table>
  <tr>
    <th width="33%">Observability</th>
    <th width="33%">Evaluation</th>
    <th width="33%">Evolution</th>
  </tr>
  <tr>
    <td><a href="docs/assets/platform-observability.png"><img src="docs/assets/platform-observability.png" alt="Synthetic Demo: HelixHarness usage and stage observations" width="100%"></a></td>
    <td><a href="docs/assets/platform-e
```

（文件节选，后续内容未采集）

## examples/starter/tasks.py

[固定版本源码](https://github.com/didi/helixharness/blob/44ffc536f73d8af525b4795ad10384c516898576/examples/starter/tasks.py)

```
"""A tiny task list used as the starting point for the HelixHarness walkthrough."""


def list_tasks(tasks):
    """Return task titles in their original order."""
    return [task["title"] for task in tasks]

```

## examples/starter/test_tasks.py

[固定版本源码](https://github.com/didi/helixharness/blob/44ffc536f73d8af525b4795ad10384c516898576/examples/starter/test_tasks.py)

```
import unittest

from tasks import list_tasks


class TaskListTest(unittest.TestCase):
    def test_returns_titles_in_order(self):
        self.assertEqual(list_tasks([
            {"title": "Write spec", "done": False},
            {"title": "Review change", "done": True},
        ]), ["Write spec", "Review change"])

    def test_empty_list(self):
        self.assertEqual(list_tasks([]), [])


if __name__ == "__main__":
    unittest.main()

```

## examples/verify_feature.py

[固定版本源码](https://github.com/didi/helixharness/blob/44ffc536f73d8af525b4795ad10384c516898576/examples/verify_feature.py)

```
"""Run independent acceptance against a completed example checkout.

Usage: python3 verify_feature.py /absolute/path/to/task-filter-demo
This intentionally fails on the unmodified starter.
"""

import copy
import importlib.util
import sys
from pathlib import Path


def main():
    if len(sys.argv) != 2:
        raise SystemExit("Usage: python3 verify_feature.py /path/to/task-filter-demo")
    path = Path(sys.argv[1]).resolve() / "tasks.py"
    spec = importlib.util.spec_from_file_location("example_tasks", path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    tasks = [
        {"title": "Draft", "done": False},
        {"title": "Review", "done": True},
        {"title": "Ship", "done": False},
    ]
    before = copy.deepcopy(tasks)
    for status, expected in (("all", ["Draft", "Review", "Ship"]),
                             ("open", ["Draft", "Ship"]), ("done", ["Review"])):
        actual = module.list_tasks(tasks, status=status)
        if actual != expected:
            raise AssertionError(f"{status}: expected {expected}, got {actual}")
        if module.list_tasks([], status=status) != []:
            raise AssertionError("empty input must return []")
    if module.list_tasks(tasks) != ["Draft", "Review", "Ship"] or tasks != before:
        raise AssertionError("default behavior or input preservation regressed")
    if module.list_tasks([tasks[1]], status="open") != []:
        raise AssertionError("no matches must return []")
    for invalid in ("unknown", "", "ALL"):
        for items in (tasks, []):
            try:
                module.list_tasks(items, status=invalid)
            except ValueError:
                pass
            else:
                raise AssertionError("invalid status must raise ValueError")
    print("PASS: filtering, ordering, compatibility, empty results, validation and input preservation")


if __name__ == "__main__":
    main()

```

## helixharness-platform/bench/__init__.py

[固定版本源码](https://github.com/didi/helixharness/blob/44ffc536f73d8af525b4795ad10384c516898576/helixharness-platform/bench/__init__.py)

```
"""Clean-slate HelixHarness framework evaluation module."""

```

## helixharness-platform/bench/case_difficulty.py

[固定版本源码](https://github.com/didi/helixharness/blob/44ffc536f73d8af525b4795ad10384c516898576/helixharness-platform/bench/case_difficulty.py)

```
"""Derive an operator-facing Case difficulty from its frozen business-code delta."""

from __future__ import annotations

import subprocess
from dataclasses import dataclass
from pathlib import Path, PurePosixPath


@dataclass(frozen=True)
class CaseDifficultyAssessment:
    """A reproducible size band plus the measurements that support it."""

    level: str
    production_files: int
    production_lines: int
    supporting_files: int
    supporting_lines: int

    @property
    def evidence_text(self) -> str:
        return f"业务代码改动 {self.production_lines} 行"


UNASSESSED_DIFFICULTY = CaseDifficultyAssessment(
    level="UNASSESSED",
    production_files=0,
    production_lines=0,
    supporting_files=0,
    supporting_lines=0,
)


def assess_case_difficulty(
    repository: Path,
    base_commit: str,
    solution_commit: str,
) -> CaseDifficultyAssessment:
    """Measure Base-to-Solution business-code changes and assign a size band."""

    completed = subprocess.run(
        [
            "git",
            "-C",
            str(repository),
            "diff",
            "--numstat",
            base_commit,
            solution_commit,
            "--",
        ],
        check=True,
        capture_output=True,
        text=True,
        timeout=30,
    )
    production_files = production_lines = 0
    supporting_files = supporting_lines = 0
    for row in completed.stdout.splitlines():
        fields = row.split("\t", 2)
        if len(fields) != 3:
            continue
        added, deleted, raw_path = fields
        changed_lines = (
            int(added) + int(deleted)
            if added.isdigit() and deleted.isdigit()
            else 0
        )
        if _is_supporting_path(raw_path):
            supporting_files += 1
            supporting_lines += changed_lines
        else:
            production_files += 1
            production_lines += changed_lines

    if production_lines > 1_000:
        level = "HARD"
    elif production_lines <= 100:
        level = "EASY"
    else:
        level = "MEDIUM"
    return CaseDifficultyAssessment(
        level=level,
        production_files=production_files,
        production_lines=production_lines,
        supporting_files=supporting_files,
        supporting_lines=supporting_lines,
    )


def _is_supporting_path(raw_path: str) -> bool:
    path = PurePosixPath(raw_path.lower())
    parts = set(path.parts)
    name = path.name
    if (
        bool(
            parts
            & {
                "test",
                "tests",
                "__tests__",
                "docs",
                "doc",
                "documentation",
                "generated",
                "generated-sources",
            }
        )
        or name.startswith("test_")
        or name.endswith(
            (
                "test.java",
                "tests.java",
                "test.kt",
                "tests.kt",
                "_test.go",
                ".spec.js",
                ".spec.jsx",
                ".spec.ts",
                ".spec.tsx",
            )
        )
    ):
        return True
    if name.endswith(".bpmn20.xml"):
        return False
    if path.suffix == ".xml":
        return "mapper" not in parts and not name.endswith("mapper.xml")
    return path.suffix not in {
        ".java",
        ".kt",
        ".scala",
        ".groovy",
        ".go",
        ".py",
        ".js",
        ".jsx",
        ".ts",
        ".tsx",
        ".vue",
        ".svelte",
        ".rb",
        ".rs",
        ".c",
        ".h",
        ".cc",
        ".cpp",
        ".cxx",
        ".hpp",
        ".cs",
        ".php",
        ".swift",
        ".m",
        ".mm",
        ".sql",
        ".proto",
        ".thrift",
        ".graphql",
        ".gql",
        ".bpmn",
    }

```

## helixharness-platform/bench/cli/__init__.py

[固定版本源码](https://github.com/didi/helixharness/blob/44ffc536f73d8af525b4795ad10384c516898576/helixharness-platform/bench/cli/__init__.py)

```
"""Command-line entry points for benchmark reporting."""

```