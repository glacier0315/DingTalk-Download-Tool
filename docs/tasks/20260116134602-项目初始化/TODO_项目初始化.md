# TODO_项目初始化

## 待办事项清单

### 必须完成的配置

#### 1. 安装外部工具
- [ ] 下载并安装 N_m3u8DL-RE 工具
  - 下载地址: https://github.com/nilaoda/N_m3u8DL-RE/releases
  - 建议版本: 最新稳定版
  - 安装后记录可执行文件路径

- [ ] 下载并安装 FFmpeg 工具
  - 下载地址: https://ffmpeg.org/download.html
  - 建议版本: 最新稳定版
  - 安装后记录可执行文件路径

#### 2. 配置环境变量
- [ ] 复制 `.env.example` 为 `.env`
  ```powershell
  Copy-Item .env.example .env
  ```

- [ ] 编辑 `.env` 文件，配置以下参数：
  - `N_M3U8DL_RE_PATH`: 设置 N_m3u8DL-RE 的完整路径
  - `FFMPEG_PATH`: 设置 FFmpeg 的完整路径
  - `DOWNLOAD_DIR`: 设置视频下载目录（可选，默认为 ./downloads）
  - `TEMP_DIR`: 设置临时文件目录（可选，默认为 ./temp）
  - `LOG_DIR`: 设置日志目录（可选，默认为 ./logs）
  - `LOG_LEVEL`: 设置日志级别（可选，默认为 INFO）
  - `USE_SYSTEM_PROXY`: 是否使用系统代理（可选，默认为 true）
  - `THREAD_COUNT`: 下载线程数（可选，默认为 16）
  - `RETRY_COUNT`: 下载重试次数（可选，默认为 3）
  - `TIMEOUT`: 下载超时时间（可选，默认为 100）

#### 3. 安装Python依赖
- [ ] 在conda环境中安装依赖
  ```powershell
  pip install -r requirements.txt
  ```

### 开发任务

#### 4. 创建主程序入口
- [ ] 创建 `src/dingtalk_downloader/main.py`
  - 实现命令行参数解析
  - 实现程序启动逻辑

#### 5. 实现配置管理模块
- [ ] 创建 `src/dingtalk_downloader/config/config_manager.py`
  - 实现YAML配置文件加载
  - 实现环境变量读取
  - 实现配置项获取接口

#### 6. 实现工具函数模块
- [ ] 创建 `src/dingtalk_downloader/utils/logger.py`
  - 实现日志记录器设置
  - 实现日志输出到文件和控制台

- [ ] 创建 `src/dingtalk_downloader/utils/file_helper.py`
  - 实现目录创建和检查
  - 实现文件路径处理

#### 7. 实现下载器模块
- [ ] 创建 `src/dingtalk_downloader/downloader/m3u8_downloader.py`
  - 实现m3u8 URL解析
  - 实现调用N_m3u8DL-RE下载视频
  - 实现下载进度显示

- [ ] 创建 `src/dingtalk_downloader/downloader/video_merger.py`
  - 实现调用FFmpeg合并视频分片
  - 实现视频格式转换

### 测试任务

#### 8. 编写单元测试
- [ ] 创建 `tests/test_config.py`
  - 测试配置文件加载
  - 测试环境变量读取

- [ ] 创建 `tests/test_utils.py`
  - 测试日志记录器
  - 测试文件操作函数

- [ ] 创建 `tests/test_downloader.py`
  - 测试m3u8下载器
  - 测试视频合并器

### 可选增强功能

#### 9. 添加命令行界面
- [ ] 使用 `argparse` 或 `click` 实现CLI
  - 支持指定m3u8 URL
  - 支持指定输出目录
  - 支持指定视频质量
  - 支持批量下载

#### 10. 添加断点续传
- [ ] 实现下载状态保存
- [ ] 实现下载恢复功能

#### 11. 添加进度显示
- [ ] 使用 `tqdm` 显示下载进度
- [ ] 显示下载速度和剩余时间

#### 12. 添加错误处理
- [ ] 实现网络错误重试
- [ ] 实现友好的错误提示

### 文档任务

#### 13. 创建README.md
- [ ] 编写项目介绍
- [ ] 编写安装说明
- [ ] 编写使用示例
- [ ] 编写配置说明

#### 14. 创建LICENSE
- [ ] 选择合适的开源许可证（如MIT License）

---

## 快速开始指南

### 1. 安装外部工具

#### 安装 N_m3u8DL-RE
1. 访问 https://github.com/nilaoda/N_m3u8DL-RE/releases
2. 下载最新版本的 Windows 可执行文件
3. 解压到指定目录（如 `C:\Tools\N_m3u8DL-RE\`）
4. 记录 `N_m3u8DL-RE.exe` 的完整路径

#### 安装 FFmpeg
1. 访问 https://ffmpeg.org/download.html
2. 下载 Windows 版本的 FFmpeg
3. 解压到指定目录（如 `C:\Tools\ffmpeg\`）
4. 将 `C:\Tools\ffmpeg\bin` 添加到系统 PATH 环境变量
5. 记录 `ffmpeg.exe` 的完整路径

### 2. 配置项目

```powershell
# 复制环境变量示例文件
Copy-Item .env.example .env

# 编辑 .env 文件，配置工具路径
notepad .env
```

在 `.env` 文件中配置：
```
N_M3U8DL_RE_PATH=C:\Tools\N_m3u8DL-RE\N_m3u8DL-RE.exe
FFMPEG_PATH=C:\Tools\ffmpeg\bin\ffmpeg.exe
```

### 3. 安装Python依赖

```powershell
# 确保在conda环境中
pip install -r requirements.txt
```

### 4. 验证安装

```powershell
# 验证 N_m3u8DL-RE
& "C:\Tools\N_m3u8DL-RE\N_m3u8DL-RE.exe" --version

# 验证 FFmpeg
& "C:\Tools\ffmpeg\bin\ffmpeg.exe" -version

# 验证 Python 依赖
python -c "import yaml, requests; print('Dependencies installed successfully')"
```

---

## 常见问题

### Q1: N_m3u8DL-RE 下载失败怎么办？
A: 检查网络连接，确保URL有效，检查认证参数是否正确。

### Q2: FFmpeg 合并失败怎么办？
A: 确保 FFmpeg 已正确安装并添加到 PATH，检查视频分片是否完整。

### Q3: 如何指定视频质量？
A: 在m3u8 URL中通常包含质量标识（如 `_normal.m3u8`, `_hd.m3u8`），选择对应的URL即可。

### Q4: 下载速度慢怎么办？
A: 可以在 `.env` 文件中增加 `THREAD_COUNT` 参数，或使用代理加速。

---

## 联系方式

如有问题或建议，请通过以下方式联系：
- GitHub Issues: https://github.com/yourusername/DingTalk-Download-Tool/issues
- Email: your.email@example.com
