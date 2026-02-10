# Init-Maker

一个 AI Agent Skill，用于为开源项目自动生成 **Windows 一键初始化脚本** (`init.bat`)。

## 这是什么？

很多开源项目的部署流程散落在 README 的各个角落——装 Python、装 Node、建虚拟环境、填 API Key……对新手极不友好。

**Init-Maker** 是一份给 AI Agent 阅读的技能定义文件 (`SKILL.md`)。当你把它交给支持 Skill 的 AI Agent（如 Claude）时，Agent 会：

1. 自动扫描目标项目的依赖与配置文件
2. 生成一个**完整可运行**的 `init.bat` 脚本
3. 用户双击即可完成项目的全部初始化

## 适用项目类型

| 层 | 技术栈 |
|----|--------|
| 前端 | 纯 HTML + JS + CSS（通过 npm 管理依赖） |
| 后端 | Python |

> 其他架构的项目可以参考本 Skill 的结构进行改造。

## 生成的脚本做了什么？

脚本严格分为 **三个阶段**，按顺序执行：

### 阶段一：环境检查

逐项检测系统是否已安装必要工具（Python、Node.js、npm 等）。

- 若系统支持 `winget`（Windows 10 2004+ / Windows 11），会询问是否**自动安装**缺失工具
- 若 winget 不可用或用户选择手动安装，则提示安装后按回车重新检测
- 所有检查通过后才进入下一阶段

### 阶段二：自动安装

全自动、无需用户操作：

- 创建 Python 虚拟环境 & 安装 `requirements.txt`
- 在前端目录执行 `npm install`
- 其他项目所需的自动化步骤

已完成的步骤会自动跳过（可重复运行）。

### 阶段三：凭据配置

交互式引导用户填写 API Key、Secret 等敏感信息：

- 自动识别项目中的 `example_credentials.*` 等模板文件
- **逐个字段** 提示输入，不会一次性要求填完所有内容
- 全部填写完成后，自动生成正式的凭据文件

## 快速开始

### 前提条件

- 一个支持 Agent Skill 的 AI 工具（如 [Claude Code](https://code.claude.com)、Claude.ai、API）
- 你的目标项目是前后端分离的 Python + HTML/JS/CSS 架构

### 使用方式

**方式一：作为 Claude Code Skill**

将本项目的 `SKILL.md` 放到 Claude Code 的 Skill 目录中：

```
# 全局安装（对所有项目生效）
~/.claude/skills/init-maker/SKILL.md

# 或项目级安装
your-project/.claude/skills/init-maker/SKILL.md
```

然后在 Claude Code 中对目标项目说：

```
请为本项目生成一个 Windows init.bat 初始化脚本
```

**方式二：直接作为 Prompt**

将 `SKILL.md` 的内容复制粘贴到任意 AI 对话中，再附上你的项目信息即可。

### 输出示例

Agent 会输出一个完整的 `init.bat` 文件，用户只需双击运行：

```
============================================================
  Phase 1: Environment Check
============================================================
[OK] Python detected.
[OK] Node.js detected.
[OK] npm detected.

============================================================
  Phase 2: Automated Installation
============================================================
[INFO] Creating Python virtual environment...
[SKIP] node_modules already exists.
[INFO] Installing Python dependencies...
[OK] All dependencies installed.

============================================================
  Phase 3: Credential Configuration
============================================================
[INFO] Configuring credentials from example_credentials.py ...
  Enter your API_KEY: ********
  Enter your SECRET: ********
[OK] credentials.py created.

============================================================
  Initialization complete! You can now start the project.
============================================================
```

## 设计原则

- **幂等性** — 脚本可重复运行，不会重复创建或覆盖已有内容
- **不退出，只阻塞** — 环境缺失时等待用户安装，而非直接报错退出
- **逐项交互** — 凭据逐字段录入，降低用户出错概率
- **零外部依赖** — 纯 Windows Batch，不依赖 PowerShell、WSL 或第三方工具
- **动态适配** — Agent 会根据项目实际文件（而非硬编码）生成脚本内容

## 项目结构

```
init-maker/
├── SKILL.md      # AI Agent 技能定义文件（核心）
└── README.md     # 本文件
```

## 自定义与扩展

`SKILL.md` 中的规则可以根据需要调整：

- **增加检测项** — 在 Phase 1 的检查表中添加新工具（如 Docker、Java）
- **修改安装步骤** — 在 Phase 2 中增减自动化操作
- **适配其他凭据格式** — Phase 3 支持 `.py`、`.json`、`.env` 等任意格式
- **迁移到其他平台** — 将 `.bat` 语法替换为 `.sh` 即可适配 Linux/macOS

## 许可证

MIT
