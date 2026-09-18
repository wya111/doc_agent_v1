# 跨境物流单证智能转换与合规校验系统

> PDF / 图片 (OCR) / EDI (TXT) → 统一 JSON → 40+ 规则校验 → 目的国合规附加风险

本系统接收 **报关单 / 提单 / 装箱单** 三份跨境物流单证，自动完成格式解析、字段归一化抽取、三单交叉比对与业务合规校验，最终以 Streamlit 网页形式可视化展示结果。所有解析在本地完成，数据不上云。

---

## 目录结构

```
agent_v2/
├── doc_validator/            # 主项目
│   ├── main.py               # Streamlit 前端 & 流水线编排入口
│   ├── pipeline.py           # 解析 → 归一化 → 校验 核心流水线（与前端解耦）
│   ├── extract.py            # PDF / 图片 / EDI 分派与文本提取
│   ├── llm_client.py         # 本地 Ollama 客户端（模型枚举、抽取、JSON 修复）
│   ├── validate.py           # 规则库、合规知识库、价格参考、校验引擎
│   ├── core.py               # 数据模型（pydantic）、字段清洗 & 规范化工具
│   ├── notify.py             # 错误邮件通知（预留接口）
│   ├── rule.json             # 业务规则配置
│   ├── compliance_kb.json    # 目的国合规附加风险知识库
│   ├── price_ref.json        # 价格参考数据
│   ├── requirements.txt      # Python 依赖
│   ├── webui.bat             # Windows 一键启动脚本
│   └── .streamlit/config.toml
└── test/                     # 测试数据
    ├── clean/                # 正常示例（clean/1、clean/3 …）
    └── error/                # 故意含错示例（group_017、group_018 …）
```

---

## 前置环境

### 1. Conda / Python 环境

需要 [Miniconda](https://docs.conda.io/en/latest/miniconda.html) 或 [Anaconda](https://www.anaconda.com/download) 已安装。

- **Python 版本**：3.10+
- **建议环境名**：`doc_agent`（与 `webui.bat` 默认匹配）

### 2. Ollama

需要 [Ollama](https://ollama.com/download) 本地运行（默认监听 `http://localhost:11434`）。

> Ollama 负责所有大模型推理，本系统**不调用任何云端 API**。

### 3. 拉取模型

至少需要一个文本模型，默认使用 **qwen2.5:3b**：

```bash
ollama pull qwen2.5:3b
```

可选的增强模型（按需拉取，系统会自动识别能力并在侧边栏列出）：

| 模型 | 能力 | 适用场景 |
|------|------|----------|
| `qwen2.5:3b` | 文本 | 默认推荐，速度快，资源占用低 |
| `qwen2.5:7b` | 文本 | 提取准确度更高 |
| `qwen2.5vl:7b` | 视觉 | 图片 / 扫描件 PDF 直接送模型识别 |
| `qwen3:4b` / `deepseek-r1:7b` | 思考 | 带思考链输出，复杂单证效果更好 |

查看已安装模型：

```bash
ollama list
```

### 4. 硬件建议

| 场景 | 最低 | 推荐 |
|------|------|------|
| 仅文本模型（qwen2.5:3b） | 4 GB RAM | 8 GB RAM |
| 视觉模型（qwen2.5vl） | 8 GB RAM | 16 GB RAM + 独立显卡 |
| 思考模型（qwen3 / r1） | 8 GB RAM | 16 GB RAM + 独立显卡 |

---

## 部署步骤

### 第一步：创建 Conda 环境

```bash
conda create -n doc_agent python=3.10 -y
conda activate doc_agent
```

### 第二步：安装 Python 依赖

```bash
cd agent_v2/doc_validator
pip install -r requirements.txt
```

主要依赖包括：

- `ollama` — 本地大模型客户端
- `streamlit` — 网页前端
- `rapidocr-onnxruntime` — OCR（图片 / 扫描件 PDF）
- `pdfplumber` / `pypdfium2` — PDF 文本抽取与渲染
- `pydantic` — 数据模型校验
- `loguru` — 日志

### 第三步：确认 Ollama 正在运行

```bash
ollama serve
```

或者如果已安装为系统服务，直接访问 [http://localhost:11434](http://localhost:11434) 应返回 `Ollama is running`。

### 第四步：启动系统

**方式 A — 一键脚本（Windows）：**

双击 `doc_validator\webui.bat`，按菜单选择 `[1]` 启动。脚本会自动：

1. 检查 Ollama 是否在线
2. 清理旧日志
3. 释放 8501 端口
4. 在新 cmd 窗口启动 Streamlit
5. 等待就绪后自动打开浏览器

**方式 B — 命令行手动启动：**

```bash
conda activate doc_agent
cd agent_v2/doc_validator
python -m streamlit run main.py --server.port 8501 --server.headless true
```

浏览器访问 <http://localhost:8501>。

---

## 使用说明

1. **侧边栏选择模型**：系统自动枚举 Ollama 已安装模型，按能力标注（`[视觉]` / `[思考]`）。默认选择最轻量的文本模型。
2. **选择数据来源**：
   - **上传单证文件**：支持 `.pdf / .png / .jpg / .efi / .edi / .txt`，文件名需以 `declaration` / `billOfLading` / `packingList` 开头以便自动归类。
   - **使用 test 测试数据**：内置 `test/` 下的正常 / 异常示例，可直接复现。
3. **点击「🚀 解析并校验」**：后端执行完整流水线，处理日志实时滚动展示。
4. **查看结果**：五个标签页
   - 📋 **单据信息** — 三份单证的关键字段卡片
   - 📦 **货物清单** — 商品明细行（HS、品名、数量、单价、重量…）
   - 🗺 **航线** — 起运港 → 目的港 示意图
   - ✅ **校验结果** — 错误清单 / 风险提醒 / 目的国附加风险
   - 🖥 **处理日志** — 完整运行日志（含 LLM 思考记录）

---

## 处理流水线

```
上传 3 份单证
    │
    ▼
┌───────────────────────────────────────┐
│ 1. 分派 dispatch()                    │
│   .pdf → processPdf (pdfplumber /     │
│                      pypdfium2+OCR)   │
│   .png / .jpg → processImg (RapidOCR) │
│   .efi / .edi / .txt → processEfi     │
└───────────────────────────────────────┘
    │ 原始文本 / 图片
    ▼
┌───────────────────────────────────────┐
│ 2. 本地 Ollama 归一化抽取             │
│   文本模型 → 传文本                    │
│   视觉模型 → 传图片（PDF 渲染后）      │
│   EDI → 确定性骨架 + LLM 翻译合并     │
│   → 统一 JSON → pydantic DocumentData │
└───────────────────────────────────────┘
    │ DocumentData
    ▼
┌───────────────────────────────────────┐
│ 3. 规则校验 run_all()                 │
│   - 单单据内部校验                     │
│   - 三单交叉比对                       │
│   - 业务合规风险（rule.json）          │
│   - 目的国附加风险（compliance_kb.json）│
└───────────────────────────────────────┘
    │ errors / warnings / advisories
    ▼
┌───────────────────────────────────────┐
│ 4. 通知（预留）                       │
│    识别到错误 → 邮件通知接口           │
└───────────────────────────────────────┘
```

**设计原则**：AI（Ollama）只做「文本提取 + 归一化翻译」，所有判定走**确定性规则**；LLM 输出失败时自动回退正则兜底。

---

## 常见问题

### Q1：启动时提示「未检测到 Ollama 服务」
```
[错误] 未检测到 Ollama 服务 11434 端口，请先启动 Ollama
```
先运行 `ollama serve`，或检查是否被防火墙拦截。

### Q2：侧边栏模型列表为空
Ollama 未拉取任何模型，执行 `ollama pull qwen2.5:3b` 后重启 Streamlit。

### Q3：视觉模型拉取失败 / 内存不足
视觉模型体积较大（qwen2.5vl:7b 约 4.7 GB），建议在有独立显卡的机器运行；纯文本场景使用 `qwen2.5:3b` 即可。

### Q4：端口 8501 被占用
`webui.bat` 会自动释放；手动启动时使用 `--server.port 自定义端口`。

### Q5：日志文件位置
每次启动自动清空历史日志，写入 `doc_validator/output/app.log`。

### Q6：conda 路径不匹配
`webui.bat` 中的 conda 路径为默认 `C:\Users\wangy\miniconda3\condabin\conda.bat`，如安装位置不同请编辑脚本中的该行。

---

## 技术栈

| 层 | 技术 |
|----|------|
| 前端 | Streamlit + 自定义 CSS |
| 后端 | Python 3.10+ |
| LLM | Ollama（本地，qwen2.5 系列为主） |
| OCR | RapidOCR ONNX Runtime |
| PDF | pdfplumber + pypdfium2 |
| 数据模型 | pydantic v2 |
| 日志 | loguru |
| 通知 | SMTP（预留接口） |
