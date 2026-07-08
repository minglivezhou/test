# COBOL Knowledge Graph QA System — Architecture & Pipeline

## 系统架构总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                    COBOL KG QA System                                │
│                                                                      │
│  [1] Ingestion  →  [2] Parse  →  [3] Entity/Relation Extraction      │
│       ↓                                                              │
│  [4] Normalization  →  [5] Business Semantics Layer  (★新增)         │
│       ↓                                                              │
│  [6] Validation  →  [7] Export (CSV/JSONL)                           │
│       ↓                                                              │
│  [8] Neo4j Import  →  [9] Retrieval  →  [10] QA                     │
│       ↓                                                              │
│  [11] MCP Server  ←→  [12] Incremental Update                        │
└──────────────────────────────────────────────────────────────────────┘
```

> **执行模式分类说明**
> - 🔵 **Rule-based**：纯规则/语法驱动，确定性强，速度快
> - 🟡 **AI-assisted**：规则为主，LLM/向量模型辅助消歧或补全
> - 🔴 **Fully-AI**：主要依赖 LLM，结构化程度较低的步骤

---

## Step 1 — Ingestion（代码库摄入）

**执行模式：** 🔵 Rule-based

**职责：** 扫描源码目录，收集 COBOL 源文件、Copybook、JCL、数据库 DDL 等原始文件。

**数据样本（输入）：**
```
/src/cobol/
  ├── ORDMAIN.cbl        # 主程序
  ├── ORDSVC.cbl         # 订单服务子程序
  ├── ORDCOPY.cpy        # Copybook（数据定义共享）
  └── ORDJCL.jcl         # JCL 批处理脚本
```

**数据样本（输出）：** 文件清单 JSONL
```jsonl
{"path": "src/cobol/ORDMAIN.cbl",  "type": "COBOL_PROGRAM", "size_bytes": 12480, "encoding": "EBCDIC", "hash": "a3f9c1..."}
{"path": "src/cobol/ORDCOPY.cpy",  "type": "COPYBOOK",       "size_bytes": 3240,  "encoding": "EBCDIC", "hash": "b2e4d7..."}
{"path": "src/cobol/ORDJCL.jcl",   "type": "JCL",            "size_bytes": 980,   "encoding": "ASCII",  "hash": "c1f8a2..."}
```

---

## Step 2 — Parse（语法解析）

**执行模式：** 🔵 Rule-based

**职责：** 使用 COBOL Parser（推荐 ProLeap COBOL Parser / ANTLR grammars-v4）将每个源文件转成 AST 或 ASG。

**关键工具：**
- [ProLeap COBOL Parser](https://github.com/uwol/proleap-cobol-parser)（含 symbol resolution、call resolution）
- ANTLR4 `grammars-v4/cobol` grammar

**数据样本（输入）：** 原始 COBOL 源码片段
```cobol
IDENTIFICATION DIVISION.
PROGRAM-ID. ORDMAIN.

DATA DIVISION.
WORKING-STORAGE SECTION.
  01 WS-ORDER-ID    PIC 9(10).
  01 WS-STATUS      PIC X(2)  VALUE 'OK'.

PROCEDURE DIVISION.
  MAIN-PARA.
    CALL 'ORDSVC' USING WS-ORDER-ID WS-STATUS.
    STOP RUN.
```

**数据样本（输出）：** AST/ASG 摘要（JSON）
```json
{
  "program": "ORDMAIN",
  "divisions": ["IDENTIFICATION", "DATA", "PROCEDURE"],
  "working_storage": [
    {"level": "01", "name": "WS-ORDER-ID", "pic": "9(10)"},
    {"level": "01", "name": "WS-STATUS",   "pic": "X(2)", "value": "OK"}
  ],
  "paragraphs": [
    {
      "name": "MAIN-PARA",
      "statements": [
        {"type": "CALL", "target": "ORDSVC", "using": ["WS-ORDER-ID", "WS-STATUS"]},
        {"type": "STOP RUN"}
      ]
    }
  ]
}
```

---

## Step 3 — Entity / Relation Extraction（实体关系抽取）

**执行模式：** 🔵 Rule-based（结构实体）+ 🟡 AI-assisted（复杂调用消歧）

**职责：** 从 AST/ASG 抽取标准化实体与关系。

### 实体类型

| Entity Type    | 说明                   |
|----------------|------------------------|
| `Program`      | COBOL 主程序           |
| `Paragraph`    | PROCEDURE DIVISION 段  |
| `Section`      | SECTION 块             |
| `Copybook`     | 共享数据定义           |
| `DataItem`     | 数据项（变量）         |
| `FileDesc`     | 文件描述（FD）         |
| `DB2Table`     | SQL 访问的 DB2 表      |
| `CICSCommand`  | CICS 事务命令          |

### 关系类型

| Relation Type   | 说明                           |
|-----------------|--------------------------------|
| `CALLS`         | 程序调用关系                   |
| `CONTAINS`      | 结构包含（Program→Paragraph）  |
| `COPIES`        | Copybook 引用                  |
| `DEFINES`       | 定义数据项                     |
| `READS_TABLE`   | 读取 DB2 表                    |
| `WRITES_TABLE`  | 写入 DB2 表                    |
| `USES_FILE`     | 使用文件描述符                 |

**数据样本（输出）：** 实体 JSONL
```jsonl
{"id": "prog:ORDMAIN",       "type": "Program",   "name": "ORDMAIN",      "path": "src/cobol/ORDMAIN.cbl"}
{"id": "prog:ORDSVC",        "type": "Program",   "name": "ORDSVC",       "path": "src/cobol/ORDSVC.cbl"}
{"id": "copy:ORDCOPY",       "type": "Copybook",  "name": "ORDCOPY",      "path": "src/cobol/ORDCOPY.cpy"}
{"id": "data:ORDMAIN|WS-ORDER-ID", "type": "DataItem", "name": "WS-ORDER-ID", "pic": "9(10)", "owner": "prog:ORDMAIN"}
```

**数据样本（输出）：** 关系 JSONL
```jsonl
{"from": "prog:ORDMAIN", "to": "prog:ORDSVC",  "type": "CALLS",  "paragraph": "MAIN-PARA", "confidence": 1.0,  "source": "ast"}
{"from": "prog:ORDMAIN", "to": "copy:ORDCOPY", "type": "COPIES", "paragraph": null,         "confidence": 1.0,  "source": "ast"}
```

> **关系存储要点：**  
> Neo4j 中每条关系独立一行记录。  
> - `ORDMAIN → ORDSVC`（CALLS）：一条记录  
> - `ORDMAIN → ORDCOPY`（COPIES）：另一条独立记录  
> 无论节点 A 与多少个节点有关系，每对关系都单独一行，互不干扰。

---

## Step 4 — Normalization（归一化）

**执行模式：** 🔵 Rule-based

**职责：** 统一实体 ID 格式、关系类型枚举、属性类型，确保后续导入无歧义。

**规则：**
- 实体 ID 格式：`{type_prefix}:{qualified_name}`，如 `prog:ORDMAIN`、`data:ORDMAIN|WS-ORDER-ID`
- 关系类型全大写下划线风格：`CALLS`、`READS_TABLE`、`WRITES_TABLE`
- 数值字段转 `int/float`，布尔字段转 `true/false`
- 空值统一为 `null`

**数据样本（前后对比）：**

| 字段          | 归一化前               | 归一化后                           |
|---------------|------------------------|------------------------------------|
| 实体 ID       | `ordmain`              | `prog:ORDMAIN`                     |
| 关系类型      | `call_to`              | `CALLS`                            |
| start_line    | `"42"`（字符串）       | `42`（整数）                       |
| confidence    | `"0.93"`（字符串）     | `0.93`（浮点）                     |
| is_async      | `"false"`（字符串）    | `false`（布尔）                    |

---

## Step 5 — Business Semantics Layer（业务语义层）★ 新增

**执行模式：** 🟡 AI-assisted / 🔴 Fully-AI

**职责：** 在结构实体基础上，叠加**业务含义标签**，将技术节点映射到业务概念，使 QA 系统能回答"哪个程序处理订单域"、"哪些 Copybook 属于客户数据"等业务级问题。

### 5.1 业务域标签（Rule-based）

通过命名规则、目录约定、程序注释自动推断：

```jsonl
{"id": "prog:ORDMAIN",  "business_domain": "ORDER",   "source": "naming_rule"}
{"id": "prog:PMTMAIN",  "business_domain": "PAYMENT",  "source": "naming_rule"}
{"id": "copy:CUSTCOPY", "business_domain": "CUSTOMER", "source": "dir_convention"}
```

命名规则示例：
```python
DOMAIN_PREFIXES = {
    "ORD": "ORDER",
    "PMT": "PAYMENT",
    "CUST": "CUSTOMER",
    "INV": "INVENTORY",
}
```

### 5.2 业务功能摘要（AI-assisted）

对每个 Paragraph / Program 用 LLM 生成自然语言摘要：

```jsonl
{
  "id": "prog:ORDMAIN|para:MAIN-PARA",
  "summary": "Entry point of order processing: calls ORDSVC to validate and persist the order, then terminates the batch.",
  "source": "llm",
  "model": "gpt-4o",
  "generated_at": "2026-07-08T10:00:00Z"
}
```

### 5.3 业务实体关系扩展

新增业务级关系：

| Relation Type       | 说明                             |
|---------------------|----------------------------------|
| `BELONGS_TO_DOMAIN` | 程序/Copybook 属于某业务域       |
| `HANDLES_ENTITY`    | 程序处理某业务实体（如 ORDER）   |
| `TRIGGERS`          | 触发下游程序或批处理作业         |

```jsonl
{"from": "prog:ORDMAIN", "to": "domain:ORDER",    "type": "BELONGS_TO_DOMAIN", "source": "naming_rule"}
{"from": "prog:ORDMAIN", "to": "entity:Order",    "type": "HANDLES_ENTITY",    "source": "llm"}
{"from": "prog:ORDMAIN", "to": "prog:PMTMAIN",    "type": "TRIGGERS",          "source": "jcl_analysis"}
```

---

## Step 6 — Validation（验证）

**执行模式：** 🔵 Rule-based

**职责：** 在导出前校验数据完整性，防止脏数据进入 Neo4j。

**校验规则：**

| 校验项                        | 说明                                       |
|-------------------------------|--------------------------------------------|
| 实体 ID 非空且唯一            | 防止重复节点                               |
| 关系 `from`/`to` 存在于实体集 | 悬空关系检测                               |
| 关系类型在枚举白名单内        | 防止类型拼写错误                           |
| `confidence` 在 [0,1] 区间    | 数值范围校验                               |
| Copybook 引用可解析           | 引用的 Copybook 文件已被摄入               |

**数据样本（验证报告）：**
```json
{
  "total_entities": 1240,
  "total_relations": 3870,
  "errors": [
    {
      "type": "DANGLING_RELATION",
      "from": "prog:LEGACYX",
      "to":   "prog:MISSING",
      "relation": "CALLS",
      "action": "skip"
    }
  ],
  "warnings": 2,
  "valid_entities": 1240,
  "valid_relations": 3869
}
```

---

## Step 7 — Export（导出 CSV/JSONL）

**执行模式：** 🔵 Rule-based

**职责：** 将验证后的实体/关系导出为 Neo4j 可导入格式。

### 节点表 `nodes.csv`

```csv
id,type,name,path,qualified_name,business_domain,summary,language,start_line,end_line
prog:ORDMAIN,Program,ORDMAIN,src/cobol/ORDMAIN.cbl,ORDMAIN,ORDER,"Entry point of order processing",COBOL,1,120
prog:ORDSVC,Program,ORDSVC,src/cobol/ORDSVC.cbl,ORDSVC,ORDER,,COBOL,1,340
copy:ORDCOPY,Copybook,ORDCOPY,src/cobol/ORDCOPY.cpy,ORDCOPY,ORDER,,COBOL,,
data:ORDMAIN|WS-ORDER-ID,DataItem,WS-ORDER-ID,,ORDMAIN.WS-ORDER-ID,,,COBOL,,
```

**节点表核心字段（最小必需）：**

| 字段               | 是否必须 | 说明                             |
|--------------------|----------|----------------------------------|
| `id`               | ✅ 必须  | 唯一标识，关系表靠它连接         |
| `type`             | ✅ 必须  | 映射为 Neo4j label               |
| `name`             | ✅ 必须  | 可读名称                         |
| `path`             | 推荐     | 源文件路径                       |
| `business_domain`  | 推荐     | 业务语义层叠加                   |
| `summary`          | 可选     | LLM 生成摘要（供向量检索）       |

### 关系表 `edges.csv`

```csv
from,to,type,paragraph,confidence,source,line
prog:ORDMAIN,prog:ORDSVC,CALLS,MAIN-PARA,1.0,ast,42
prog:ORDMAIN,copy:ORDCOPY,COPIES,,1.0,ast,
prog:ORDMAIN,domain:ORDER,BELONGS_TO_DOMAIN,,1.0,naming_rule,
prog:ORDMAIN,entity:Order,HANDLES_ENTITY,,0.85,llm,
```

**关系表核心字段（最小必需）：**

| 字段         | 是否必须 | 说明                          |
|--------------|----------|-------------------------------|
| `from`       | ✅ 必须  | 起点节点 ID                   |
| `to`         | ✅ 必须  | 终点节点 ID                   |
| `type`       | ✅ 必须  | 关系类型（全大写）            |
| `confidence` | 推荐     | 置信度 [0,1]                  |
| `source`     | 推荐     | 来源（ast/lsp/llm/rule）      |

> **关键说明：Neo4j 关系是"一条记录一个关系"**  
> - `ORDMAIN → ORDSVC`（CALLS）= 第 1 条记录  
> - `ORDMAIN → ORDCOPY`（COPIES）= 第 2 条记录（独立，与上面无关）  
> - `ORDMAIN → domain:ORDER`（BELONGS_TO_DOMAIN）= 第 3 条记录  
> 每个节点对之间有多少种关系，就有多少行记录。每行只描述"A 和 B 的某一种关系"。

---

## Step 8 — Neo4j Import（图数据库导入）

**执行模式：** 🔵 Rule-based

**职责：** 将 CSV 导入 Neo4j，建立图结构。

### 导入节点（LOAD CSV）

```cypher
LOAD CSV WITH HEADERS FROM 'file:///nodes.csv' AS row
MERGE (n:Entity {id: row.id})
SET n.type            = row.type,
    n.name            = row.name,
    n.path            = row.path,
    n.business_domain = row.business_domain,
    n.summary         = row.summary,
    n.start_line      = CASE WHEN row.start_line = '' THEN null
                             ELSE toInteger(row.start_line) END,
    n.end_line        = CASE WHEN row.end_line   = '' THEN null
                             ELSE toInteger(row.end_line)   END
```

按 type 追加二级 label（推荐）：
```cypher
MATCH (n:Entity) WHERE n.type = 'Program'
SET n:Program;

MATCH (n:Entity) WHERE n.type = 'Copybook'
SET n:Copybook;
```

### 导入关系（LOAD CSV，以 CALLS 为例）

```cypher
LOAD CSV WITH HEADERS FROM 'file:///edges.csv' AS row
WITH row WHERE row.type = 'CALLS'
MATCH (a:Entity {id: row.from})
MATCH (b:Entity {id: row.to})
MERGE (a)-[r:CALLS]->(b)
SET r.paragraph  = row.paragraph,
    r.confidence = CASE WHEN row.confidence = '' THEN null
                        ELSE toFloat(row.confidence) END,
    r.source     = row.source,
    r.line       = CASE WHEN row.line = '' THEN null
                        ELSE toInteger(row.line) END
```

> 如果关系类型是动态的，需要 APOC 或按类型分表导入。

### 推荐索引

```cypher
CREATE INDEX entity_id IF NOT EXISTS FOR (n:Entity) ON (n.id);
CREATE INDEX entity_type IF NOT EXISTS FOR (n:Entity) ON (n.type);
CREATE INDEX entity_domain IF NOT EXISTS FOR (n:Entity) ON (n.business_domain);
```

---

## Step 9 — Retrieval（检索）

**执行模式：** 🟡 AI-assisted（向量检索 + 图检索混合）

**职责：** 给定用户问题，混合图检索（Cypher）和向量检索，找到相关实体/代码片段。

### 检索策略

| 场景                        | 策略                                  |
|-----------------------------|---------------------------------------|
| "ORDMAIN 调用了哪些程序"    | Cypher 图遍历（精确，Rule-based）     |
| "哪些程序处理订单"          | 业务域过滤 + 图遍历（Rule+向量）      |
| "找处理库存扣减的逻辑"      | 向量相似度检索（Fully-AI）            |
| "ORDMAIN 的数据流"          | 多跳图遍历（Rule-based）              |

**Cypher 检索示例：**
```cypher
// 查 ORDMAIN 的所有直接调用
MATCH (p:Program {name: 'ORDMAIN'})-[:CALLS]->(callee:Program)
RETURN callee.name, callee.path, callee.summary

// 查订单域所有程序
MATCH (p:Program {business_domain: 'ORDER'})
RETURN p.name, p.summary ORDER BY p.name
```

**数据样本（检索结果）：**
```json
[
  {"name": "ORDSVC",  "path": "src/cobol/ORDSVC.cbl",  "summary": "Validates and persists order data"},
  {"name": "ORDVAL",  "path": "src/cobol/ORDVAL.cbl",  "summary": "Order field validation routines"}
]
```

---

## Step 10 — QA（问答）

**执行模式：** 🔴 Fully-AI（LLM 综合检索结果生成回答）

**职责：** 将检索结果拼装为 Prompt，调用 LLM 生成自然语言回答，含代码引用和置信度。

**Prompt 模板（示意）：**
```
你是 COBOL 代码知识专家。根据以下代码知识图谱信息回答用户问题。

【图谱检索结果】
Program: ORDMAIN (ORDER 域)
  - CALLS: ORDSVC (订单验证与持久化)
  - CALLS: ORDVAL (字段校验)
  - COPIES: ORDCOPY (共享数据定义)

【用户问题】
ORDMAIN 的主要职责是什么，它依赖哪些程序？

【回答要求】
- 引用具体程序名
- 说明调用链
- 如有不确定，注明置信度
```

**数据样本（输出）：**
```json
{
  "question": "ORDMAIN 的主要职责是什么，它依赖哪些程序？",
  "answer": "ORDMAIN 是订单处理的入口程序（ORDER 域）。它的主要职责是协调订单的校验与持久化流程：\n1. 调用 ORDVAL 做字段级校验\n2. 调用 ORDSVC 完成订单的业务验证和数据库写入\n3. 通过 ORDCOPY Copybook 共享订单数据结构定义",
  "sources": ["prog:ORDMAIN", "prog:ORDSVC", "prog:ORDVAL", "copy:ORDCOPY"],
  "confidence": 0.92
}
```

---

## Step 11 — MCP Server（模型上下文协议服务）

**执行模式：** 🔵 Rule-based（路由）+ 🔴 Fully-AI（LLM 调用）

**职责：** 提供标准化 MCP 接口，让外部 AI 工具（Claude、Copilot 等）以工具调用形式查询 COBOL 知识图谱。

**MCP 工具定义（示意）：**
```json
{
  "tools": [
    {
      "name": "query_program",
      "description": "查询 COBOL 程序的结构、调用关系和业务语义",
      "parameters": {
        "program_name": "string",
        "include_callees": "boolean",
        "include_data_items": "boolean"
      }
    },
    {
      "name": "search_by_domain",
      "description": "按业务域查找相关程序和 Copybook",
      "parameters": {
        "domain": "string  // ORDER | PAYMENT | CUSTOMER | INVENTORY"
      }
    },
    {
      "name": "ask_cobol_qa",
      "description": "自然语言提问，获取基于知识图谱的回答",
      "parameters": {
        "question": "string"
      }
    }
  ]
}
```

**数据样本（MCP 请求/响应）：**
```json
// 请求
{"tool": "query_program", "program_name": "ORDMAIN", "include_callees": true}

// 响应
{
  "program": {"name": "ORDMAIN", "domain": "ORDER", "summary": "Entry point of order processing"},
  "callees": [
    {"name": "ORDSVC", "type": "CALLS", "paragraph": "MAIN-PARA"},
    {"name": "ORDVAL", "type": "CALLS", "paragraph": "VALIDATE-PARA"}
  ]
}
```

---

## Step 12 — Incremental Update（增量更新）

**执行模式：** 🔵 Rule-based（变更检测）+ 🟡 AI-assisted（语义变更分析）

**职责：** 当源码发生变更时，仅重新处理受影响文件，避免全量重跑。

**变更检测策略：**
```python
# 比较文件 hash，找出变更文件
changed_files = [
    f for f in current_files
    if f.hash != previous_index.get(f.path, {}).get("hash")
]
```

**增量更新流程：**
```
变更文件检测
    ↓
重新 Parse 变更文件
    ↓
删除旧实体/关系（按文件范围）
    ↓
插入新实体/关系
    ↓
重新触发 Business Semantics Layer（仅受影响节点）
    ↓
更新向量索引（仅变更节点的 summary）
```

**Cypher 增量删除示例（按文件清理旧数据）：**
```cypher
// 删除来自变更文件的旧节点和关系
MATCH (n:Entity {path: 'src/cobol/ORDMAIN.cbl'})
DETACH DELETE n
```

**数据样本（增量日志）：**
```json
{
  "run_id": "incr-20260708-001",
  "changed_files": ["src/cobol/ORDMAIN.cbl"],
  "deleted_entities": 12,
  "deleted_relations": 34,
  "inserted_entities": 14,
  "inserted_relations": 37,
  "duration_seconds": 8.3
}
```

---

## 完整执行模式一览表

| 步骤                           | 执行模式          | 关键工具/技术                            |
|--------------------------------|-------------------|------------------------------------------|
| 1. Ingestion                   | 🔵 Rule-based     | 文件扫描、hash 计算                      |
| 2. Parse                       | 🔵 Rule-based     | ProLeap / ANTLR4                         |
| 3. Entity/Relation Extraction  | 🔵/🟡 混合        | AST visitor + LLM 消歧                   |
| 4. Normalization               | 🔵 Rule-based     | 规则映射、类型转换                       |
| 5. Business Semantics Layer    | 🟡/🔴 混合        | 命名规则 + LLM 摘要 + JCL 分析           |
| 6. Validation                  | 🔵 Rule-based     | Schema 校验、悬空关系检测                |
| 7. Export (CSV/JSONL)          | 🔵 Rule-based     | Python csv 库                            |
| 8. Neo4j Import                | 🔵 Rule-based     | LOAD CSV / neo4j-admin import            |
| 9. Retrieval                   | 🟡 AI-assisted    | Cypher 图遍历 + 向量检索（Qdrant 等）    |
| 10. QA                         | 🔴 Fully-AI       | LLM (GPT-4o / Claude)                   |
| 11. MCP Server                 | 🔵/🔴 混合        | MCP SDK + LLM                            |
| 12. Incremental Update         | 🔵/🟡 混合        | Hash diff + 图更新 + 向量更新            |

---

## Neo4j 节点/关系 Schema 快速参考

### 最小节点表字段

```csv
id,type,name,path,qualified_name,business_domain,summary,language,start_line,end_line
```

| 字段              | 必须 | 类型    | 说明                    |
|-------------------|------|---------|-------------------------|
| `id`              | ✅   | string  | 全局唯一，稳定可复现    |
| `type`            | ✅   | string  | 枚举：Program/Copybook… |
| `name`            | ✅   | string  | 可读名称                |
| `path`            | 推荐 | string  | 源文件路径              |
| `business_domain` | 推荐 | string  | 业务域（ORDER 等）      |
| `summary`         | 可选 | string  | LLM 摘要（向量化用）    |
| `start_line`      | 可选 | integer | 定义起始行              |
| `end_line`        | 可选 | integer | 定义结束行              |

### 最小关系表字段

```csv
from,to,type,paragraph,confidence,source,line
```

| 字段         | 必须 | 类型   | 说明                              |
|--------------|------|--------|-----------------------------------|
| `from`       | ✅   | string | 起点节点 id                       |
| `to`         | ✅   | string | 终点节点 id                       |
| `type`       | ✅   | string | 全大写：CALLS/COPIES/DEFINES…     |
| `paragraph`  | 可选 | string | 发生调用的段名                    |
| `confidence` | 推荐 | float  | 置信度 [0,1]                      |
| `source`     | 推荐 | string | 来源：ast/lsp/llm/rule/jcl        |
| `line`       | 可选 | integer| 源码行号                         |

> **一条记录 = 一个关系（不可合并）**  
> A→B 的 CALLS 和 A→C 的 CALLS 是两条独立记录，不能合并到一行。  
> A→B 的 CALLS 和 A→B 的 COPIES 也是两条独立记录（同节点对，不同关系类型）。
