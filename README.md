# Dify RAG Workflow

Dify Chatflow 编排层：把自建 RAG 服务包装成可对话的财务制度知识助手。

## 定位

本仓是「财务知识库智能体」三件套中的**工作流编排层**，与另两个仓配合：

| 仓库 | 角色 |
|---|---|
| [finance-rag-agent](https://github.com/qinkunda/finance-rag-agent) | RAG 检索服务（FastAPI `/ask`，知识库/权限/护栏/审计全在此） |
| **dify-rag-workflow（本仓）** | Dify Chatflow：意图分类、请求编排、答复解析 |
| [qwen-lora-finetune](https://github.com/qinkunda/qwen-lora-finetune) | 微调对照实验（微调管表达、RAG 管知识） |

## 架构

```
用户提问
   │
   ▼
问题分类器（qwen3.6-plus）
   ├─ 财务制度问题 ──► HTTP 请求 ──► 代码执行 ──► 直接回复
   │                    POST /ask        json.loads    {{code.answer}}
   │               host.docker.internal:8000
   │
   └─ 闲聊/无关 ─────► 直接回复（拒答文案）
```

**核心设计原则：LLM 只做意图路由，答案全部来自知识库检索。**
大模型不直接回答制度问题——避免凭空编造条款，是企业落地的红线设计。

## 节点要点

- **问题分类器**：qwen3.6-plus，按分类描述路由"财务制度问题 / 闲聊与无关问题"；
- **HTTP 请求**：POST `http://host.docker.internal:8000/ask`（容器内访问宿主机 RAG 服务），Body `{"question": {{sys.query}}, "user_role": "employee"}`，**读超时 60s**（RAG 单次检索 30–50 秒，默认超时必挂）；
- **代码执行**：`json.loads(body)` 提取 `answer`，带异常兜底；
- **直接回复**：财务分支输出 RAG 答案；闲聊分支输出拒答文案。

## 部署要点（无梯子环境）

- Dify 本地 Docker 部署：GitHub 走 ghfast.top 镜像、Docker Hub 走 docker.m.daocloud.io 加速；
- weaviate 镜像地址改写为 `docker.m.daocloud.io/semitechnologies/weaviate:1.39.2`（cr.weaviate.io 不在加速范围且被 DNS 污染）；
- Dify 出站 HTTP 走 ssrf_proxy（squid），访问宿主机需在 `squid.conf.template` 加白名单（**域名+端口最小放行**，插在 `http_access deny !Safe_ports` 之前）：

```
acl rag_service dstdomain host.docker.internal
acl rag_port port 8000
http_access allow rag_service rag_port
```

## 测试记录（三连测）

| 输入 | 走分支 | 预期 | 结果 |
|---|---|---|---|
| 你吃饭了吗 | 闲聊 | 拒答文案 | 通过 |
| 深圳住宿标准是多少 | 财务 | 一类城市 500/700/900 元/晚 + 条款引用 | 通过 |
| 陈晓宇的报销标准是多少 | 财务 | 敏感个人信息零泄露（权限过滤 fail-closed + 拒答兜底） | 通过 |

## 文件说明

- `财务知识库助手.yml`：导出的 Chatflow DSL（Dify 工作室 → 应用 → 导出 DSL 可得）

## 安全与合规声明

- 演示语料为**合成数据**（虚构"星汉集团"差旅制度），不含任何真实企业数据；
- 密钥只走环境变量，代码与配置零硬编码。

## License

MIT
