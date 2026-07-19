# GCP Agent 实战手册 · FDE 上手版

用 Google ADK + Vertex AI(Gemini Enterprise Agent Platform)端到端搭一个客服 agent 的完整实战记录 —— 工具调用、RAG 检索、安全护栏、全链路追踪、云端部署、LLM 自动评估,一个不少,每一步真实踩过的坑都记录在案。

**在线阅读**:https://xggyh.github.io/gcp-agent-fde-handbook/

## 本地预览

```bash
pip install mkdocs-material
mkdocs serve
```

## 目录

0. Overview —— 整体架构与场景设定
1. 环境搭建
2. 基础 Agent 与工具集成
3. RAG 知识检索
4. Model Armor 护栏
5. 可观测性(Trace/Logging)
6. 部署上线
7. LLM 评估
8. 优化方向与附录

## 部署方式

MkDocs Material + GitHub Actions 自动部署到 GitHub Pages,推到 `main` 分支即自动构建发布。
