# Hermes Learning Agent for llama.cpp

这个目录是给外部 Hermes 风格学习代理使用的仓库本地学习工作区。

它的目标不是替用户生成 llama.cpp 贡献代码，而是把源码阅读过程变成可复盘的学习路径：先观察仓库，再提出小问题，逐步记录证据、概念债务、代码地图和阶段检查点。

## 目录结构

- `learner-profile.md`：记录当前学习起点、偏好和可编辑假设。
- `learning-goals.md`：记录学习终点、阶段目标和面试导向问题地图。
- `exploration-protocol.md`：规定代理如何侦察、选择问题、记录证据和处理概念债务。
- `agent-prompt.md`：交给 Hermes 或其他外部 agent 的启动提示。
- `templates/`：探索日志、概念债务、阶段检查点模板。
- `outputs/`：后续所有探索产物。请把日志、图谱、债务表和检查点都写到这里。

## 使用方式

1. 按需修改 `learner-profile.md` 和 `learning-goals.md`。
2. 启动外部 agent 时，把 `agent-prompt.md` 作为提示词，并指定当前仓库根目录为工作目录。
3. 让 agent 每轮只处理一个小问题，把所有过程记录写入 `outputs/`。
4. 每轮结束后阅读产物，再决定下一轮是否继续、改目标或补课。

## llama.cpp 特别边界

本仓库的 `AGENTS.md` 明确允许 AI 用于学习和探索，但对 AI 代写贡献代码有严格限制。因此本工作区默认只做源码学习、证据整理、调试思路和面试准备。

如果未来学习目标变成实际贡献，agent 必须先帮助用户理解相关代码和设计取舍，再由用户主导方案与实现。
