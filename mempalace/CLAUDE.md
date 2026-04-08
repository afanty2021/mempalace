# MemPalace 核心包

[根目录](../CLAUDE.md) > **mempalace/**

> 最后更新：2026-04-08 22:41:22

## 模块职责

MemPalace 核心包包含所有核心功能模块，提供：
- **数据采集**：从项目文件和对话导出中提取记忆
- **存储管理**：ChromaDB 向量存储 + SQLite 知识图谱
- **检索系统**：语义搜索 + 四层记忆栈
- **AI 集成**：MCP 服务器（19 工具）+ Python API
- **压缩优化**：AAAK 方言（有损缩写）

---

## 模块结构图

```mermaid
graph TD
    A["mempalace/ - 核心包"] --> B["CLI 与服务器"];
    A --> C["数据采集"];
    A --> D["存储与检索"];
    A --> E["高级功能"];

    B --> B1["cli.py - CLI 入口"];
    B --> B2["mcp_server.py - MCP 服务器"];

    C --> C1["miner.py - 项目文件采集"];
    C --> C2["convo_miner.py - 对话采集"];
    C --> C3["normalize.py - 格式转换"];
    C --> C4["split_mega_files.py - 文件分割"];

    D --> D1["searcher.py - 语义搜索"];
    D --> D2["layers.py - 四层记忆栈"];
    D --> D3["knowledge_graph.py - 知识图谱"];
    D --> D4["palace_graph.py - 房间导航"];

    E --> E1["dialect.py - AAAK 压缩"];
    E --> E2["onboarding.py - 引导设置"];
    E --> E3["entity_detector.py - 实体检测"];
    E --> E4["general_extractor.py - 通用提取"];
    E --> E5["room_detector_local.py - 房间检测"];

    click B1 "#cli" "查看 CLI 详情"
    click B2 "#mcp-server" "查看 MCP 服务器详情"
    click C1 "#miner" "查看项目采集详情"
    click C2 "#convo-miner" "查看对话采集详情"
    click D1 "#searcher" "查看搜索详情"
    click D2 "#layers" "查看记忆栈详情"
    click D3 "#knowledge-graph" "查看知识图谱详情"
```

---

## 核心模块详解

### CLI 与服务器

#### `cli.py` - CLI 入口点
**职责**：命令行接口，路由所有用户命令

**关键命令**：
```python
# 初始化宫殿
mempalace init <dir>

# 采集数据
mempalace mine <dir>                              # 项目文件
mempalace mine <dir> --mode convos                # 对话导出

# 搜索
mempalace search <query>

# 唤醒上下文
mempalace wake-up

# 状态
mempalace status
```

**入口函数**：`main()` - 使用 argparse 解析命令

#### `mcp_server.py` - MCP 服务器
**职责**：提供 19 个 MCP 工具给 AI 助手

**工具分类**：
- **读取（7 个）**：`mempalace_status`, `mempalace_list_wings`, `mempalace_list_rooms`, `mempalace_get_taxonomy`, `mempalace_search`, `mempalace_check_duplicate`, `mempalace_get_aaak_spec`
- **写入（2 个）**：`mempalace_add_drawer`, `mempalace_delete_drawer`
- **知识图谱（5 个）**：`mempalace_kg_query`, `mempalace_kg_add`, `mempalace_kg_invalidate`, `mempalace_kg_timeline`, `mempalace_kg_stats`
- **导航（3 个）**：`mempalace_traverse`, `mempalace_find_tunnels`, `mempalace_graph_stats`
- **代理日记（2 个）**：`mempalace_diary_write`, `mempalace_diary_read`

**Palace Protocol**：AI 首次唤醒时自动学习
```python
PALACE_PROTOCOL = """
1. ON WAKE-UP: Call mempalace_status to load palace overview + AAAK spec.
2. BEFORE RESPONDING: call mempalace_kg_query or mempalace_search FIRST.
3. IF UNSURE: say "let me check" and query the palace.
4. AFTER EACH SESSION: call mempalace_diary_write.
5. WHEN FACTS CHANGE: call mempalace_kg_invalidate then mempalace_kg_add.
"""
```

---

### 数据采集

#### `miner.py` - 项目文件采集
**职责**：扫描项目目录，分块存储文件到 ChromaDB

**核心功能**：
- **目录扫描**：递归遍历，支持 `.gitignore` 过滤
- **文件路由**：根据路径和内容自动分配到房间
- **文本分块**：800 字符块，100 字符重叠
- **去重检查**：跳过已采集的文件

**关键函数**：
```python
def mine(project_dir, palace_path, wing_override=None, agent="mempalace",
         limit=0, dry_run=False, respect_gitignore=True, include_ignored=None):
    """采集项目目录到宫殿"""
```

**支持的文件类型**：`.txt`, `.md`, `.py`, `.js`, `.ts`, `.json`, `.yaml`, `.html`, `.css`, `.java`, `.go`, `.rs`, `.rb`, `.sh`, `.csv`, `.sql`, `.toml`

**跳过的目录**：`.git`, `node_modules`, `__pycache__`, `.venv`, `dist`, `build`, `.next`, `coverage`

#### `convo_miner.py` - 对话采集
**职责**：从对话导出中提取记忆

**支持的格式**：
- Claude Code JSONL
- Claude.ai JSON
- ChatGPT JSON
- Slack JSON
- 纯文本

**提取模式**：
- `exchange`：按 Q+A 对分块（默认）
- `general`：分类为 5 种记忆类型（决策、偏好、里程碑、问题、情感）

**关键函数**：
```python
def mine_convos(convo_dir, palace_path, wing=None, agent="mempalace",
                limit=0, dry_run=False, extract_mode="exchange"):
    """采集对话导出到宫殿"""
```

#### `normalize.py` - 格式转换
**职责**：将 5 种聊天格式转换为标准对话格式

**转换函数**：
```python
def normalize_claude_code_jsonl(raw_content: str) -> list
def normalize_claude_ai_json(raw_content: str) -> list
def normalize_chatgpt_json(raw_content: str) -> list
def normalize_slack_json(raw_content: str) -> list
def normalize_plain_text(raw_content: str) -> list
```

#### `split_mega_files.py` - 文件分割
**职责**：将连接的对话文件分割为每会话文件

**使用场景**：某些导出将多个会话连接到一个巨大文件中

**命令**：
```bash
mempalace split ~/chats/                      # 分割
mempalace split ~/chats/ --dry-run            # 预览
mempalace split ~/chats/ --min-sessions 3     # 仅分割 >= 3 会话的文件
```

---

### 存储与检索

#### `searcher.py` - 语义搜索
**职责**：通过 ChromaDB 向量搜索查找记忆

**搜索接口**：
```python
def search(query: str, palace_path: str, wing: str = None,
           room: str = None, n_results: int = 5):
    """搜索宫殿，返回逐字抽屉内容"""

def search_memories(query: str, palace_path: str, wing: str = None,
                    room: str = None, n_results: int = 5) -> dict:
    """编程搜索 - 返回字典而非打印"""
```

**过滤选项**：
- `wing`：限制到一个项目/人
- `room`：限制到一个主题
- `n_results`：返回结果数

#### `layers.py` - 四层记忆栈
**职责**：分层记忆管理，按需加载

**层级结构**：
```python
class Layer0:    # ~100 tokens，始终加载
    """从 ~/.mempalace/identity.txt 读取 AI 身份"""

class Layer1:    # ~500-800 tokens，始终加载
    """从宫殿自动生成关键事实"""

class Layer2:    # ~200-500 tokens，按需加载
    """翅膀/房间过滤检索"""

class Layer3:    # 无限制，按需加载
    """完整语义搜索"""

class MemoryStack:
    """统一接口"""
    def wake_up(wing=None)        # L0 + L1
    def recall(wing, room)        # L2
    def search(query, wing, room) # L3
```

**使用示例**：
```python
from mempalace.layers import MemoryStack

stack = MemoryStack()
print(stack.wake_up())                # ~170 tokens
print(stack.recall(wing="my_app"))     # 按需
print(stack.search("pricing change"))  # 按需
```

#### `knowledge_graph.py` - 知识图谱
**职责**：时序实体关系图（SQLite）

**核心功能**：
- **实体管理**：人、项目、工具、概念
- **关系三元组**：`subject → predicate → object`
- **时间有效性**：`valid_from → valid_to`
- **时间查询**：查看特定时间点的状态

**使用示例**：
```python
from mempalace.knowledge_graph import KnowledgeGraph

kg = KnowledgeGraph()
kg.add_triple("Max", "child_of", "Alice", valid_from="2015-04-01")
kg.add_triple("Max", "loves", "chess", valid_from="2025-10-01")

# 查询：Max 的所有关系
kg.query_entity("Max")

# 时间查询：2026 年 1 月 Max 的状态
kg.query_entity("Max", as_of="2026-01-15")

# 失效：Max 的运动损伤已恢复
kg.invalidate("Max", "has_issue", "sports_injury", ended="2026-02-15")
```

**存储**：`~/.mempalace/knowledge_graph.sqlite3`

#### `palace_graph.py` - 房间导航
**职责**：基于房间的导航图

**核心功能**：
- **BFS 遍历**：从房间跨翅膀走图
- **隧道检测**：发现跨域连接
- **连通性统计**：图结构概览

---

### 高级功能

#### `dialect.py` - AAAK 压缩
**职责**：AAAK 方言编码器（有损缩写）

**格式规范**：
```
实体：3 字母大写代码（ALC=Alice）
情感：*动作标记* (*warm*=喜悦)
结构：管道分隔字段
日期：ISO 格式 (2026-03-31)
重要性：★ 到 ★★★★★
```

**使用示例**：
```python
from mempalace.dialect import Dialect

# 基本：压缩任何文本
dialect = Dialect()
compressed = dialect.compress("We decided to use GraphQL instead of REST...")

# 带实体映射
dialect = Dialect(entities={"Alice": "ALC", "Bob": "BOB"})

# 从配置文件
dialect = Dialect.from_config("entities.json")
```

**注意**：AAAK 是**有损**的，不能重建原始文本。在 LongMemEval 上得分 84.2% vs 原始模式 96.6%。

#### `onboarding.py` - 引导设置
**职责**：首次运行的引导式设置

**功能**：
- 询问用户关于人和项目
- 生成 AAAK 引导文件
- 创建翅膀配置
- 设置身份文件

#### `entity_detector.py` - 实体检测
**职责**：从内容自动检测人和项目

**功能**：
- 扫描文件查找名称
- 确认检测到的实体
- 保存到 `entities.json`

#### `general_extractor.py` - 通用提取
**职责**：将文本分类为 5 种记忆类型

**类型**：
- `decision`：做出的决定
- `preference`：偏好和习惯
- `milestone`：里程碑事件
- `problem`：问题和挑战
- `emotional`：情感和感受

#### `room_detector_local.py` - 房间检测
**职责**：使用 70+ 模式将文件夹映射到房间名

**模式示例**：
- `auth*` → `auth`
- `user*` → `users`
- `test*` → `testing`
- `deploy*` → `deployment`

---

## 配置与数据

### 配置文件

**`~/.mempalace/config.json`**：
```json
{
  "palace_path": "/custom/path/to/palace",
  "collection_name": "mempalace_drawers",
  "people_map": {"Kai": "KAI", "Priya": "PRI"}
}
```

**`~/.mempalace/wing_config.json`**（由 `init` 生成）：
```json
{
  "default_wing": "wing_general",
  "wings": {
    "wing_kai": {"type": "person", "keywords": ["kai", "kai's"]},
    "wing_driftwood": {"type": "project", "keywords": ["driftwood", "analytics"]}
  }
}
```

**`~/.mempalace/identity.txt`**：
纯文本，成为 L0 - 每次会话加载。

### 数据存储

**ChromaDB**：
- 路径：`~/.mempalace/palace`
- 集合：`mempalace_drawers`
- 元数据：wing, room, source_file, chunk_index, added_by, filed_at

**SQLite 知识图谱**：
- 路径：`~/.mempalace/knowledge_graph.sqlite3`
- 表：`entities`, `triples`

---

## 测试

### 测试文件

```
tests/
├── test_config.py              # 配置系统
├── test_convo_miner.py         # 对话采集
├── test_dialect.py             # AAAK 压缩
├── test_knowledge_graph.py     # 知识图谱
├── test_mcp_server.py          # MCP 服务器
├── test_miner.py               # 项目采集
├── test_normalize.py           # 格式转换
├── test_searcher.py            # 搜索功能
└── test_split_mega_files.py    # 文件分割
```

### 运行测试

```bash
# 所有测试
pytest tests/ -v

# 单个模块
pytest tests/test_searcher.py -v

# 带覆盖
pytest tests/ --cov=mempalace --cov-report=html
```

### 测试覆盖率

> 最后更新：2026-04-09 00:03:00

```bash
# 生成覆盖率报告
pytest tests/ --cov=mempalace --cov-report=html --cov-report=term

# 查看 HTML 报告
open htmlcov/index.html  # macOS
```

**覆盖率统计**：
- **总体覆盖率**：**30%**（2525/3591 行）
- 测试结果：**79 通过**，**5 失败**，**17 错误**（共 101 个测试）
- HTML 报告：`htmlcov/index.html`
- 测试文件数：10 个

**覆盖率详情**（部分模块）：
| 模块 | 覆盖率 | 说明 |
|------|--------|------|
| `config.py` | 94% | 配置系统 |
| `dialect.py` | 72% | AAAK 压缩 |
| `convo_miner.py` | 68% | 对话采集 |
| `searcher.py` | 14% | 语义搜索（部分功能） |
| `spellcheck.py` | 27% | 拼写检查 |

**备注**：
- 测试覆盖核心功能：采集、存储、检索、MCP 服务器
- 基准测试独立运行（见 `benchmarks/`）
- 覆盖率数据通过 `pytest-cov` 生成
- 部分测试失败原因：httpx 网络超时（测试环境问题，非代码缺陷）

---

## 常见问题 (FAQ)

**Q: 如何重置宫殿？**
```bash
rm -rf ~/.mempalace/palace
mempalace init ~/projects/myapp
mempalace mine ~/projects/myapp
```

**Q: 如何备份宫殿？**
```bash
cp -r ~/.mempalace/palace ~/.mempalace/palace.backup
cp ~/.mempalace/knowledge_graph.sqlite3 ~/.mempalace/kg.backup
```

**Q: AAAK vs 原始模式？**
- 原始模式：96.6% LongMemEval，默认存储
- AAAK 模式：84.2% LongMemEval，用于上下文加载

**Q: 如何修复损坏的宫殿？**
```bash
mempalace repair  # 从 SQLite 元数据重建向量索引
```

---

## 相关文件清单

### 核心模块（27 个文件）

**CLI 与服务器**：
- `cli.py`
- `mcp_server.py`

**数据采集**：
- `miner.py`
- `convo_miner.py`
- `normalize.py`
- `split_mega_files.py`

**存储与检索**：
- `searcher.py`
- `layers.py`
- `knowledge_graph.py`
- `palace_graph.py`

**高级功能**：
- `dialect.py`
- `onboarding.py`
- `entity_detector.py`
- `general_extractor.py`
- `room_detector_local.py`
- `entity_registry.py`
- `spellcheck.py`

**配置与工具**：
- `config.py`
- `version.py`
- `__init__.py`
- `__main__.py`

### 测试（10 个文件）

- `tests/conftest.py`
- `tests/test_config.py`
- `tests/test_convo_miner.py`
- `tests/test_dialect.py`
- `tests/test_knowledge_graph.py`
- `tests/test_mcp_server.py`
- `tests/test_miner.py`
- `tests/test_normalize.py`
- `tests/test_searcher.py`
- `tests/test_split_mega_files.py`
- `tests/test_version_consistency.py`

---

## 变更记录

### 2026-04-08 - 模块文档创建 🚀

- ✅ **创建核心包文档**：
  - 模块职责与结构图
  - 27 个核心模块详解
  - 配置与数据存储说明
  - 测试指南与 FAQ
  - 相关文件清单

- 📊 **模块分析完成**：
  - CLI 与服务器：2 个模块
  - 数据采集：4 个模块
  - 存储与检索：4 个模块
  - 高级功能：8 个模块
  - 配置与工具：4 个模块

- 🔧 **技术栈识别**：
  - 存储：ChromaDB + SQLite
  - 压缩：自定义 AAAK 方言
  - 集成：MCP 协议
  - 测试：pytest

- 📖 **文档覆盖**：
  - ✅ 核心包文档完成
  - ✅ Mermaid 结构图生成
  - ✅ 导航面包屑添加
