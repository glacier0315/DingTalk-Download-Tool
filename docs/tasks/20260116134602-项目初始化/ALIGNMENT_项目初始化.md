# ALIGNMENT\_项目初始化

## 项目上下文分析

### 现有项目结构

```
DingTalk-Download-Tool/
├── .trae/
│   └── rules/
│       └── project_rules.md
└── docs/
    └── foundation/
        ├── N_m3u8DL-RE.md
        ├── ffmpeg.md
        └── 钉钉视频下载记录.md
```

### 技术背景

- **目标**: 开发钉钉直播回放视频下载软件
- **核心技术**: Python
- **视频格式**: m3u8 (HLS 流媒体格式)
- **下载工具**: N_m3u8DL-RE (第三方工具)
- **视频处理**: ffmpeg (视频合并和转码)
- **操作系统**: Windows 11
- **环境管理**: miniconda3 (Python 环境管理)

### 业务域理解

根据现有文档分析:

1. 钉钉直播回放视频使用 m3u8 格式存储
2. m3u8 文件包含多个 ts 分片，每个分片都有独立的 URL 和认证参数
3. 需要使用 N_m3u8DL-RE 工具下载 m3u8 视频流
4. 下载后使用 ffmpeg 合并分片为完整的视频文件

## 原始需求

用户要求:

1. 初始化一个符合 Python 最佳实践的项目结构
2. 创建主程序目录、配置文件目录、工具函数目录、测试目录及必要的初始化文件
3. 初始化 Git 仓库
4. 创建并配置.gitignore 文件以排除 Python 项目常见的不需要版本控制的文件和目录

## 边界确认

### 包含范围

- 创建符合 Python 最佳实践的项目目录结构
- 初始化 Git 仓库
- 创建.gitignore 文件
- 创建必要的**init**.py 文件

### 不包含范围

- 实现视频下载功能
- 实现视频合并功能
- 实现配置文件解析功能
- 实现日志系统
- 实现命令行界面
- 编写单元测试

## 需求理解

### Python 项目最佳实践标准结构

```
project_name/
├── src/                    # 源代码目录
│   ├── project_name/       # 主包目录
│   │   ├── __init__.py
│   │   ├── main.py         # 主程序入口
│   │   ├── config/         # 配置模块
│   │   │   └── __init__.py
│   │   ├── utils/          # 工具函数模块
│   │   │   └── __init__.py
│   │   └── downloader/     # 下载器模块
│   │       └── __init__.py
├── tests/                  # 测试目录
│   ├── __init__.py
│   ├── test_config.py
│   └── test_utils.py
├── config/                 # 配置文件目录
│   └── settings.yaml
├── .env                    # 环境变量文件
├── .gitignore             # Git忽略文件
├── pyproject.toml         # 项目配置文件
├── requirements.txt       # 依赖列表
├── README.md              # 项目说明
└── LICENSE                # 许可证
```

### .gitignore 标准配置

Python 项目需要忽略的文件和目录:

- 虚拟环境目录 (venv/, env/, .venv/)
- Python 缓存目录 (**pycache**/, _.pyc, _.pyo, \*.pyd)
- IDE 配置 (.vscode/, .idea/, _.swp, _.swo)
- 日志文件 (\*.log)
- 环境变量文件 (.env)
- 测试覆盖率报告 (.coverage, htmlcov/)
- 构建产物 (build/, dist/, \*.egg-info/)
- 操作系统文件 (.DS_Store, Thumbs.db)

## 疑问澄清

### 已明确的决策

1. **项目名称**: DingTalk-Download-Tool
2. **包名称**: dingtalk_downloader (符合 Python 包命名规范，使用下划线分隔)
3. **Python 版本**: 基于用户环境，建议使用 Python 3.8+
4. **依赖管理工具**: 使用 requirements.txt 和 pyproject.toml
5. **配置文件格式**: 使用 YAML 格式（易于阅读和编辑）

### 需要确认的问题

**无** - 所有需求都已明确，可以直接基于 Python 最佳实践进行实现。

## 技术约束

### 环境约束

- 操作系统: Windows 11
- Python 环境: miniconda3
- 终端: PowerShell 5.1

### 技术约束

- 必须遵循 Python PEP 8 编码规范
- 必须遵循 Python 项目结构最佳实践
- 必须配置完整的.gitignore 文件
- 必须初始化 Git 仓库

## 项目特性规范

### 命名规范

- 项目名称: DingTalk-Download-Tool
- 包名称: dingtalk_downloader
- 模块名称: 使用小写字母和下划线
- 类名称: 使用大驼峰命名法
- 函数和变量: 使用小写字母和下划线

### 目录结构规范

- src/: 源代码根目录
- tests/: 测试代码目录
- config/: 配置文件目录
- docs/: 文档目录（已存在）

### 文件规范

- **init**.py: 每个 Python 包目录都必须包含
- .env: 存储敏感信息（API 密钥等），不提交到 Git
- .gitignore: 排除不需要版本控制的文件
- pyproject.toml: 现代 Python 项目配置文件
- requirements.txt: 依赖列表
