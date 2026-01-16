# CONSENSUS_钉钉直播回放下载软件开发指南

## 需求描述

编写一份专业、系统且全面的《钉钉直播回放下载软件开发指南》技术文档，该文档存储于项目的docs目录下。本指南旨在详细阐述如何使用Python语言实现钉钉直播回放视频的完整下载功能，为开发人员提供清晰、可操作的技术指导。

## 技术实现方案

### 核心技术栈

#### 编程语言
- **Python**: 3.8+（支持3.8、3.9、3.10、3.11、3.12）

#### 核心依赖库
- **PyYAML** >= 6.0: 配置文件解析
- **requests** >= 2.31.0: HTTP请求处理
- **playwright** >= 1.40.0: 自动化浏览器操作（可选）
- **selenium** >= 4.15.0: 自动化浏览器操作（可选）

#### 外部工具
- **N_m3u8DL-RE**: m3u8/HLS流媒体下载工具
- **ffmpeg**: 视频处理和合并工具

#### 开发工具
- **pytest** >= 7.4.0: 测试框架
- **pytest-cov** >= 4.1.0: 测试覆盖率
- **black** >= 23.7.0: 代码格式化
- **isort** >= 5.12.0: 导入排序
- **mypy** >= 1.5.0: 类型检查
- **flake8** >= 6.1.0: 代码风格检查
- **pylint** >= 3.0.0: 代码质量检查

### 功能模块设计

#### 1. Cookie获取机制

**实现方案**:
- 提供多种Cookie获取方式供用户选择
- 推荐手动复制方式作为默认方案
- 提供Selenium和Playwright自动化获取方案作为可选功能
- 支持从浏览器导出Cookie文件

**Cookie管理**:
- 存储格式: JSON格式，包含Cookie名称、值、域名、路径、过期时间
- 有效期管理: 自动监控Cookie过期时间，在过期前触发重新获取
- 安全注意事项: Cookie不应提交到版本控制系统，应加密存储

**代码示例**:
```python
import json
from datetime import datetime
from typing import Dict, Optional

class CookieManager:
    def __init__(self, cookie_file: str = "./cookies/cookies.json"):
        self.cookie_file = cookie_file
        self.cookies: Dict[str, Dict] = {}

    def load_cookies(self) -> bool:
        try:
            with open(self.cookie_file, 'r', encoding='utf-8') as f:
                self.cookies = json.load(f)
            return True
        except FileNotFoundError:
            return False

    def save_cookies(self) -> bool:
        try:
            with open(self.cookie_file, 'w', encoding='utf-8') as f:
                json.dump(self.cookies, f, indent=2, ensure_ascii=False)
            return True
        except Exception as e:
            print(f"保存Cookie失败: {e}")
            return False

    def is_cookie_expired(self, cookie_name: str) -> bool:
        if cookie_name not in self.cookies:
            return True
        expiry = self.cookies[cookie_name].get('expiry')
        if expiry is None:
            return False
        return datetime.now().timestamp() > expiry

    def get_cookie_string(self) -> str:
        cookie_items = []
        for name, data in self.cookies.items():
            cookie_items.append(f"{name}={data.get('value', '')}")
        return '; '.join(cookie_items)
```

#### 2. 钉钉直播回放链接解析

**实现方案**:
- 分析钉钉直播回放链接的结构
- 实现自动提取m3u8播放地址的解析算法
- 支持不同直播类型（普通直播、课堂直播等）的链接格式
- 处理认证参数（auth_key）的解析和验证

**链接结构分析**:
```
基础URL: https://dtliving-sz.dingtalk.com/live/{uuid}_normal.m3u8
认证参数: auth_key={timestamp}-{signature}-0-{hash}
```

**代码示例**:
```python
import re
from urllib.parse import urlparse, parse_qs
from typing import Dict, Optional

class DingTalkUrlParser:
    def __init__(self):
        self.url_pattern = re.compile(
            r'https://dtliving-sz\.dingtalk\.com/live/([a-f0-9-]+)_([a-z]+)\.m3u8'
        )
        self.auth_key_pattern = re.compile(
            r'auth_key=(\d+)-([a-f0-9]+)-(\d+)-([a-f0-9]+)'
        )

    def parse_url(self, url: str) -> Optional[Dict]:
        match = self.url_pattern.search(url)
        if not match:
            return None

        uuid, quality = match.groups()
        parsed_url = urlparse(url)
        auth_key = parse_qs(parsed_url.query).get('auth_key', [None])[0]

        result = {
            'uuid': uuid,
            'quality': quality,
            'base_url': f"{parsed_url.scheme}://{parsed_url.netloc}",
            'path': parsed_url.path,
            'auth_key': auth_key
        }

        if auth_key:
            auth_match = self.auth_key_pattern.search(f"auth_key={auth_key}")
            if auth_match:
                timestamp, signature, zero, hash_value = auth_match.groups()
                result.update({
                    'auth_timestamp': int(timestamp),
                    'auth_signature': signature,
                    'auth_hash': hash_value
                })

        return result

    def get_m3u8_url(self, uuid: str, quality: str = 'normal') -> str:
        return f"https://dtliving-sz.dingtalk.com/live/{uuid}_{quality}.m3u8"

    def is_auth_key_valid(self, auth_key: str, tolerance: int = 300) -> bool:
        match = self.auth_key_pattern.search(f"auth_key={auth_key}")
        if not match:
            return False

        timestamp = int(match.group(1))
        current_time = datetime.now().timestamp()
        return abs(current_time - timestamp) <= tolerance
```

#### 3. 视频分片下载

**实现方案**:
- 基于N_m3u8DL-RE工具实现视频分片下载
- 使用Python的subprocess模块调用N_m3u8DL-RE
- 配置下载参数（线程数、重试次数、超时时间等）
- 实现避免限流的请求间隔控制
- 支持断点续传功能
- 自动刷新认证信息

**N_m3u8DL-RE集成**:
```python
import subprocess
import os
from typing import Optional, List

class M3u8Downloader:
    def __init__(self, binary_path: str, ffmpeg_path: str):
        self.binary_path = binary_path
        self.ffmpeg_path = ffmpeg_path

    def download(
        self,
        m3u8_url: str,
        save_dir: str,
        save_name: str,
        headers: Optional[Dict[str, str]] = None,
        thread_count: int = 3,
        retry_count: int = 5,
        timeout: int = 100
    ) -> bool:
        cmd = [
            self.binary_path,
            m3u8_url,
            '--save-dir', save_dir,
            '--save-name', save_name,
            '--thread-count', str(thread_count),
            '--download-retry-count', str(retry_count),
            '--http-request-timeout', str(timeout),
            '--ffmpeg-binary-path', self.ffmpeg_path,
            '-M', 'format=mp4',
            '--del-after-done'
        ]

        if headers:
            for key, value in headers.items():
                cmd.extend(['-H', f'{key}: {value}'])

        try:
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                encoding='utf-8',
                errors='ignore'
            )

            if result.returncode == 0:
                return True
            else:
                print(f"下载失败: {result.stderr}")
                return False

        except subprocess.TimeoutExpired:
            print("下载超时")
            return False
        except Exception as e:
            print(f"下载异常: {e}")
            return False

    def download_with_retry(
        self,
        m3u8_url: str,
        save_dir: str,
        save_name: str,
        headers: Optional[Dict[str, str]] = None,
        max_retries: int = 3,
        retry_delay: int = 5
    ) -> bool:
        for attempt in range(max_retries):
            if self.download(m3u8_url, save_dir, save_name, headers):
                return True

            if attempt < max_retries - 1:
                print(f"重试 {attempt + 1}/{max_retries}，等待 {retry_delay} 秒...")
                import time
                time.sleep(retry_delay)
                retry_delay *= 2

        return False
```

#### 4. 视频合并技术

**实现方案**:
- 使用ffmpeg进行视频分片合并
- 配置合并参数（码率、格式、质量参数）
- 实现合并流程控制
- 提供质量保证措施
- 支持常见视频格式（MP4、MKV等）输出

**ffmpeg集成**:
```python
import subprocess
from typing import Optional

class VideoMerger:
    def __init__(self, binary_path: str):
        self.binary_path = binary_path

    def merge_ts_to_mp4(
        self,
        input_file: str,
        output_file: str,
        video_codec: str = 'libx264',
        audio_codec: str = 'aac',
        video_bitrate: str = '2000k',
        audio_bitrate: str = '128k'
    ) -> bool:
        cmd = [
            self.binary_path,
            '-i', input_file,
            '-c:v', video_codec,
            '-c:a', audio_codec,
            '-b:v', video_bitrate,
            '-b:a', audio_bitrate,
            '-y',
            output_file
        ]

        try:
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                encoding='utf-8',
                errors='ignore'
            )

            if result.returncode == 0:
                return True
            else:
                print(f"合并失败: {result.stderr}")
                return False

        except Exception as e:
            print(f"合并异常: {e}")
            return False

    def get_video_info(self, file_path: str) -> Optional[Dict]:
        cmd = [
            self.binary_path,
            '-i', file_path,
            '-f', 'null',
            '-'
        ]

        try:
            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                encoding='utf-8',
                errors='ignore'
            )

            return {
                'stderr': result.stderr,
                'returncode': result.returncode
            }

        except Exception as e:
            print(f"获取视频信息失败: {e}")
            return None
```

#### 5. 文件管理策略

**实现方案**:
- 制定标准化的文件命名规范
- 实现存储路径管理方案
- 按日期/用户/课程分类存储
- 确保下载文件的有序组织和快速检索

**文件命名规范**:
```
格式: {标题}_{日期}_{时长}.{扩展名}
示例: Python课程_20260116_013045.mp4
```

**存储路径结构**:
```
{下载目录}/{年份}/{月份}/{用户}/{课程}/
示例: ./downloads/2026/01/张三/Python课程/
```

**代码示例**:
```python
import os
import re
from datetime import datetime
from typing import Optional

class FileManager:
    def __init__(self, base_dir: str = "./downloads"):
        self.base_dir = base_dir

    def sanitize_filename(self, filename: str) -> str:
        return re.sub(r'[<>:"/\\|?*]', '_', filename)

    def generate_filename(
        self,
        title: str,
        date: Optional[datetime] = None,
        duration: Optional[int] = None,
        extension: str = 'mp4'
    ) -> str:
        sanitized_title = self.sanitize_filename(title)

        if date is None:
            date = datetime.now()

        date_str = date.strftime('%Y%m%d')

        if duration is not None:
            hours = duration // 3600
            minutes = (duration % 3600) // 60
            seconds = duration % 60
            duration_str = f"{hours:02d}{minutes:02d}{seconds:02d}"
        else:
            duration_str = "000000"

        return f"{sanitized_title}_{date_str}_{duration_str}.{extension}"

    def generate_save_path(
        self,
        user: str,
        course: str,
        date: Optional[datetime] = None
    ) -> str:
        if date is None:
            date = datetime.now()

        year = date.strftime('%Y')
        month = date.strftime('%m')

        sanitized_user = self.sanitize_filename(user)
        sanitized_course = self.sanitize_filename(course)

        path = os.path.join(
            self.base_dir,
            year,
            month,
            sanitized_user,
            sanitized_course
        )

        os.makedirs(path, exist_ok=True)
        return path

    def get_file_size(self, file_path: str) -> int:
        try:
            return os.path.getsize(file_path)
        except OSError:
            return 0

    def format_file_size(self, size: int) -> str:
        for unit in ['B', 'KB', 'MB', 'GB']:
            if size < 1024:
                return f"{size:.2f} {unit}"
            size /= 1024
        return f"{size:.2f} TB"
```

#### 6. 配置管理

**实现方案**:
- 使用YAML格式的配置文件
- 支持灵活配置应用参数
- 提供默认配置与自定义配置的合并机制
- 支持配置热加载

**配置文件示例**:
```yaml
app:
  name: "DingTalk Downloader"
  version: "0.1.0"
  debug: false

download:
  thread_count: 3
  retry_count: 5
  timeout: 100
  temp_dir: "./temp"
  output_dir: "./downloads"

n_m3u8dl_re:
  binary_path: "./assets/bin/N_m3u8DL-RE.exe"
  del_after_done: true
  use_system_proxy: false

ffmpeg:
  binary_path: "./assets/bin/ffmpeg.exe"
  output_format: "mp4"
  video_codec: "libx264"
  audio_codec: "aac"
  video_bitrate: "2000k"
  audio_bitrate: "128k"

logging:
  level: "INFO"
  file: "./logs/app_%Y-%m-%d.log"
  console: true
  format: "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
  date_format: "%Y-%m-%d %H:%M:%S"
  max_size: 10485760
  backup_count: 7
```

**配置管理代码**:
```python
import yaml
from typing import Dict, Any
from pathlib import Path

class ConfigManager:
    def __init__(self, config_file: str = "./config/settings.yaml"):
        self.config_file = config_file
        self.config: Dict[str, Any] = {}
        self.load_config()

    def load_config(self) -> bool:
        try:
            with open(self.config_file, 'r', encoding='utf-8') as f:
                self.config = yaml.safe_load(f)
            return True
        except FileNotFoundError:
            print(f"配置文件不存在: {self.config_file}")
            return False
        except yaml.YAMLError as e:
            print(f"配置文件解析失败: {e}")
            return False

    def save_config(self) -> bool:
        try:
            with open(self.config_file, 'w', encoding='utf-8') as f:
                yaml.dump(self.config, f, default_flow_style=False, allow_unicode=True)
            return True
        except Exception as e:
            print(f"保存配置文件失败: {e}")
            return False

    def get(self, key: str, default: Any = None) -> Any:
        keys = key.split('.')
        value = self.config
        for k in keys:
            if isinstance(value, dict) and k in value:
                value = value[k]
            else:
                return default
        return value

    def set(self, key: str, value: Any) -> None:
        keys = key.split('.')
        config = self.config
        for k in keys[:-1]:
            if k not in config:
                config[k] = {}
            config = config[k]
        config[keys[-1]] = value

    def merge_config(self, user_config: Dict[str, Any]) -> None:
        def merge_dict(base: Dict, update: Dict) -> Dict:
            for key, value in update.items():
                if key in base and isinstance(base[key], dict) and isinstance(value, dict):
                    base[key] = merge_dict(base[key], value)
                else:
                    base[key] = value
            return base

        self.config = merge_dict(self.config, user_config)
```

#### 7. 鉴权管理

**实现方案**:
- 实现认证信息的自动刷新机制
- 监控Cookie有效期
- 在过期前触发重新获取流程
- 确保下载过程不中断

**鉴权管理代码**:
```python
from datetime import datetime, timedelta
from typing import Optional, Callable

class AuthManager:
    def __init__(self, cookie_manager: CookieManager):
        self.cookie_manager = cookie_manager
        self.refresh_callback: Optional[Callable] = None
        self.refresh_threshold = 300  # 提前5分钟刷新

    def set_refresh_callback(self, callback: Callable) -> None:
        self.refresh_callback = callback

    def check_and_refresh(self) -> bool:
        if not self.cookie_manager.is_cookie_expired('lv_token'):
            return True

        if self.refresh_callback:
            print("Cookie即将过期，尝试刷新...")
            return self.refresh_callback()

        print("Cookie已过期，请手动刷新")
        return False

    def get_auth_headers(self) -> Dict[str, str]:
        cookie_string = self.cookie_manager.get_cookie_string()
        return {
            'Cookie': cookie_string,
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36',
            'Referer': 'https://n.dingtalk.com/',
            'Accept': '*/*',
            'Accept-Language': 'zh-CN,zh;q=0.9'
        }

    def is_auth_valid(self) -> bool:
        return not self.cookie_manager.is_cookie_expired('lv_token')
```

### 错误处理方案

#### 错误码体系

**错误码格式**: `模块标识_错误类型_错误编号`

**模块标识**:
- `AUTH`: 认证模块
- `PARSE`: 解析模块
- `DOWNLOAD`: 下载模块
- `MERGE`: 合并模块
- `CONFIG`: 配置模块
- `FILE`: 文件模块
- `UNKNOWN`: 未知错误

**错误类型**:
- `NET`: 网络错误
- `AUTH`: 认证错误
- `FILE`: 文件错误
- `PARAM`: 参数错误
- `TIMEOUT`: 超时错误
- `UNKNOWN`: 未知错误

**错误码示例**:
```python
class ErrorCode:
    AUTH_NET_001 = "AUTH_NET_001"
    AUTH_AUTH_001 = "AUTH_AUTH_001"
    PARSE_PARAM_001 = "PARSE_PARAM_001"
    DOWNLOAD_NET_001 = "DOWNLOAD_NET_001"
    DOWNLOAD_FILE_001 = "DOWNLOAD_FILE_001"
    MERGE_FILE_001 = "MERGE_FILE_001"
    CONFIG_FILE_001 = "CONFIG_FILE_001"
```

**错误码与解决方案映射**:
```python
ERROR_SOLUTIONS = {
    ErrorCode.AUTH_NET_001: {
        'message': '网络连接失败',
        'solution': '检查网络连接，重试或更换网络环境'
    },
    ErrorCode.AUTH_AUTH_001: {
        'message': '认证失败',
        'solution': '检查Cookie是否有效，重新获取Cookie'
    },
    ErrorCode.PARSE_PARAM_001: {
        'message': 'URL解析失败',
        'solution': '检查URL格式是否正确'
    },
    ErrorCode.DOWNLOAD_NET_001: {
        'message': '下载网络错误',
        'solution': '检查网络连接，增加重试次数'
    },
    ErrorCode.DOWNLOAD_FILE_001: {
        'message': '分片文件下载失败',
        'solution': '检查磁盘空间，增加重试次数'
    },
    ErrorCode.MERGE_FILE_001: {
        'message': '视频合并失败',
        'solution': '检查ffmpeg路径，检查输入文件是否完整'
    },
    ErrorCode.CONFIG_FILE_001: {
        'message': '配置文件读取失败',
        'solution': '检查配置文件路径和格式'
    }
}
```

#### 重试机制

**重试策略**:
- 网络错误: 重试5次，间隔1s, 2s, 4s, 8s, 16s（指数退避）
- 服务器错误(5xx): 重试3次，间隔2s, 4s, 8s
- 认证错误(401/403): 不重试，提示用户刷新Cookie
- 其他错误: 重试3次，间隔1s, 2s, 4s

**重试代码**:
```python
import time
from typing import Callable, Optional
from functools import wraps

class RetryStrategy:
    def __init__(self, max_retries: int = 3, base_delay: int = 1, exponential: bool = True):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.exponential = exponential

    def retry(self, func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(self.max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    if attempt < self.max_retries - 1:
                        delay = self.base_delay * (2 ** attempt if self.exponential else 1)
                        print(f"重试 {attempt + 1}/{self.max_retries}，等待 {delay} 秒...")
                        time.sleep(delay)
            raise last_exception
        return wrapper

    @staticmethod
    def should_retry(error_code: str) -> bool:
        if 'AUTH_AUTH' in error_code:
            return False
        return True
```

### 日志系统方案

#### 日志分级策略

**日志级别**:
- `DEBUG`: 调试信息，用于开发调试
- `INFO`: 一般信息，记录关键操作
- `WARNING`: 警告信息，记录潜在问题
- `ERROR`: 错误信息，记录错误事件
- `CRITICAL`: 严重错误，记录系统级错误

**日志配置**:
```python
import logging
import logging.handlers
from pathlib import Path
from typing import Optional

class LoggerManager:
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.logger = self._setup_logger()

    def _setup_logger(self) -> logging.Logger:
        logger = logging.getLogger('dingtalk_downloader')
        logger.setLevel(getattr(logging, self.config.get('level', 'INFO')))

        formatter = logging.Formatter(
            self.config.get('format', '%(asctime)s - %(name)s - %(levelname)s - %(message)s'),
            datefmt=self.config.get('date_format', '%Y-%m-%d %H:%M:%S')
        )

        if self.config.get('console', True):
            console_handler = logging.StreamHandler()
            console_handler.setFormatter(formatter)
            logger.addHandler(console_handler)

        log_file = self.config.get('file')
        if log_file:
            log_path = Path(log_file)
            log_path.parent.mkdir(parents=True, exist_ok=True)

            file_handler = logging.handlers.RotatingFileHandler(
                log_file,
                maxBytes=self.config.get('max_size', 10485760),
                backupCount=self.config.get('backup_count', 7),
                encoding='utf-8'
            )
            file_handler.setFormatter(formatter)
            logger.addHandler(file_handler)

        return logger

    def get_logger(self, name: str) -> logging.Logger:
        return logging.getLogger(f'dingtalk_downloader.{name}')
```

#### 日志记录规范

**关键操作节点**:
- 认证操作: 记录Cookie获取、刷新、验证
- 链接解析: 记录URL解析、认证参数提取
- 文件下载: 记录下载开始、进度、完成
- 视频合并: 记录合并开始、进度、完成
- 错误处理: 记录错误信息、重试、恢复

**日志格式**:
```
时间戳 - 模块名 - 日志级别 - 操作ID - 消息内容
```

**日志示例**:
```python
logger = LoggerManager(config).get_logger('downloader')

logger.info('[AUTH_001] 开始获取Cookie')
logger.debug('[PARSE_001] 解析URL: https://...')
logger.info('[DOWNLOAD_001] 开始下载视频，分片数: 100')
logger.warning('[DOWNLOAD_002] 分片下载失败，重试中...')
logger.error('[DOWNLOAD_003] 下载失败，错误码: DOWNLOAD_NET_001')
logger.critical('[SYSTEM_001] 系统错误，程序退出')
```

### 测试与验证方案

#### 单元测试

**测试框架**: pytest

**测试覆盖率**: 不低于80%

**测试示例**:
```python
import pytest
from dingtalk_downloader.parser import DingTalkUrlParser

class TestDingTalkUrlParser:
    def setup_method(self):
        self.parser = DingTalkUrlParser()

    def test_parse_normal_url(self):
        url = "https://dtliving-sz.dingtalk.com/live/65ebb87c-aa44-4616-849c-1ff33e433997_normal.m3u8?auth_key=1768550700-93c71b307a7c4f918490104e7a7570e6-0-37f29e07c84de09781af9d69e2065b58"
        result = self.parser.parse_url(url)

        assert result is not None
        assert result['uuid'] == '65ebb87c-aa44-4616-849c-1ff33e433997'
        assert result['quality'] == 'normal'
        assert result['auth_timestamp'] == 1768550700

    def test_parse_invalid_url(self):
        url = "https://example.com/invalid.m3u8"
        result = self.parser.parse_url(url)

        assert result is None

    def test_get_m3u8_url(self):
        uuid = '65ebb87c-aa44-4616-849c-1ff33e433997'
        url = self.parser.get_m3u8_url(uuid, 'hd')

        assert url == f"https://dtliving-sz.dingtalk.com/live/{uuid}_hd.m3u8"
```

#### 集成测试

**测试流程**:
1. 测试Cookie获取和验证
2. 测试URL解析和认证参数提取
3. 测试视频下载和合并
4. 测试文件管理和存储

**测试示例**:
```python
import pytest
from dingtalk_downloader.downloader import M3u8Downloader
from dingtalk_downloader.merger import VideoMerger

class TestIntegration:
    def setup_method(self):
        self.downloader = M3u8Downloader('./assets/bin/N_m3u8DL-RE.exe', './assets/bin/ffmpeg.exe')
        self.merger = VideoMerger('./assets/bin/ffmpeg.exe')

    def test_download_and_merge(self, tmp_path):
        m3u8_url = "https://example.com/video.m3u8"
        save_dir = str(tmp_path)
        save_name = "test_video"

        result = self.downloader.download(m3u8_url, save_dir, save_name)
        assert result is True

        output_file = f"{save_dir}/{save_name}.mp4"
        assert os.path.exists(output_file)
```

#### 性能测试

**测试指标**:
- 下载速度: MB/s
- CPU占用率: %
- 内存占用: MB
- 并发处理能力: 同时下载数

**测试方法**:
```python
import time
import psutil

def test_download_performance():
    start_time = time.time()
    process = psutil.Process()

    start_cpu = process.cpu_percent()
    start_mem = process.memory_info().rss / 1024 / 1024

    downloader.download(m3u8_url, save_dir, save_name)

    end_time = time.time()
    end_cpu = process.cpu_percent()
    end_mem = process.memory_info().rss / 1024 / 1024

    duration = end_time - start_time
    avg_cpu = (start_cpu + end_cpu) / 2
    avg_mem = (start_mem + end_mem) / 2

    print(f"下载耗时: {duration:.2f}秒")
    print(f"平均CPU占用: {avg_cpu:.2f}%")
    print(f"平均内存占用: {avg_mem:.2f}MB")
```

#### 异常测试

**测试场景**:
- 网络中断
- 认证失效
- 文件损坏
- 磁盘空间不足

**测试示例**:
```python
import pytest
from unittest.mock import patch, MagicMock

def test_network_interrupt():
    with patch('requests.get', side_effect=ConnectionError('Network error')):
        with pytest.raises(ConnectionError):
            downloader.download(m3u8_url, save_dir, save_name)

def test_auth_expired():
    with patch('dingtalk_downloader.auth.CookieManager.is_cookie_expired', return_value=True):
        with pytest.raises(AuthenticationError):
            downloader.download(m3u8_url, save_dir, save_name)

def test_disk_full():
    with patch('os.makedirs', side_effect=OSError('No space left')):
        with pytest.raises(OSError):
            file_manager.generate_save_path(user, course)
```

### 代码规范方案

#### Python风格规范

**遵循标准**: PEP 8

**代码格式化工具**:
- black: 代码格式化
- isort: 导入排序

**代码检查工具**:
- flake8: 代码风格检查
- pylint: 代码质量检查
- mypy: 类型检查

**配置示例**:
```ini
# setup.cfg
[flake8]
max-line-length = 100
exclude = .git,__pycache__,build,dist

[pylint]
max-line-length = 100
disable = C0111,R0903
```

#### 模块化设计

**设计原则**:
- 单一职责原则
- 开闭原则
- 依赖倒置原则

**模块划分**:
```
dingtalk_downloader/
├── __init__.py
├── config/
│   ├── __init__.py
│   └── config_manager.py
├── auth/
│   ├── __init__.py
│   ├── cookie_manager.py
│   └── auth_manager.py
├── parser/
│   ├── __init__.py
│   └── url_parser.py
├── downloader/
│   ├── __init__.py
│   ├── m3u8_downloader.py
│   └── retry_strategy.py
├── merger/
│   ├── __init__.py
│   └── video_merger.py
├── file/
│   ├── __init__.py
│   └── file_manager.py
├── logger/
│   ├── __init__.py
│   └── logger_manager.py
└── utils/
    ├── __init__.py
    └── helpers.py
```

#### 命名规范

**函数命名**: 小写字母+下划线（snake_case）
```python
def download_video(url: str, save_path: str) -> bool:
    pass
```

**类命名**: 大驼峰（PascalCase）
```python
class VideoDownloader:
    pass
```

**变量命名**: 小写字母+下划线（snake_case）
```python
video_url = "https://..."
save_path = "./downloads/"
```

**常量命名**: 大写字母+下划线（UPPER_CASE）
```python
MAX_RETRY_COUNT = 5
DEFAULT_TIMEOUT = 100
```

#### 文档要求

**文档字符串风格**: Google风格

**示例**:
```python
def download_video(
    url: str,
    save_path: str,
    headers: Optional[Dict[str, str]] = None,
    retry_count: int = 3
) -> bool:
    """下载视频文件。

    Args:
        url: 视频URL地址
        save_path: 保存路径
        headers: 请求头，可选
        retry_count: 重试次数，默认为3

    Returns:
        bool: 下载成功返回True，失败返回False

    Raises:
        ConnectionError: 网络连接失败
        AuthenticationError: 认证失败

    Examples:
        >>> download_video('https://example.com/video.m3u8', './downloads/')
        True
    """
    pass
```

## 技术约束

### 环境约束
- 操作系统: Windows 10/11
- Python版本: 3.8+
- 浏览器: Chrome 120+（用于自动化获取Cookie）

### 依赖约束
- N_m3u8DL-RE: 需要单独下载并配置路径
- ffmpeg: 需要单独下载并配置路径

### 安全约束
- Cookie不应提交到版本控制系统
- Cookie应加密存储
- 不应提供非法获取视频内容的方法
- 不应涉及钉钉的隐私数据窃取

### 性能约束
- 下载速度: 不低于1MB/s（取决于网络环境）
- 内存占用: 不超过500MB
- CPU占用: 不超过50%

## 集成方案

### 与现有系统集成
- 使用现有的配置文件格式（YAML）
- 保持与现有项目结构一致
- 复用现有的模块划分方式

### 外部工具集成
- N_m3u8DL-RE: 通过subprocess调用
- ffmpeg: 通过subprocess调用
- Selenium/Playwright: 作为可选依赖

## 任务边界限制

### 包含内容
1. 编写完整的《钉钉直播回放下载软件开发指南》技术文档
2. 文档包含5个核心章节：功能实现、错误处理、日志系统、测试与验证、代码规范
3. 提供完整的代码示例和流程图
4. 包含配置说明和参数解释
5. 确保文档符合专业技术文档规范

### 不包含内容
1. 不实现实际的下载功能代码（仅提供指南和示例）
2. 不进行实际的钉钉API调用测试
3. 不涉及钉钉的反爬虫技术规避
4. 不提供非法获取视频内容的方法
5. 不涉及钉钉的隐私数据窃取

## 验收标准

### 文档完整性
- [ ] 包含所有5个核心章节
- [ ] 每个章节内容完整，覆盖所有要求的功能点
- [ ] 包含目录、术语表、版本历史、参考文献

### 文档质量
- [ ] 所有代码示例完整可运行
- [ ] 流程图使用mermaid语法绘制
- [ ] 配置说明和参数解释清晰详细
- [ ] 文档格式符合专业技术文档规范

### 技术准确性
- [ ] 代码示例经过实际验证
- [ ] 技术方案与现有项目架构对齐
- [ ] 技术方案可行，无技术风险

### 文档可读性
- [ ] 章节结构清晰，逻辑连贯
- [ ] 语言表达准确，易于理解
- [ ] 提供清晰的导航和索引功能
- [ ] 使用Markdown格式，跨平台兼容

### 文档专业性
- [ ] 遵循专业技术文档编写规范
- [ ] 包含必要的代码注释和说明
- [ ] 提供实用的操作指引
- [ ] 包含常见问题和解决方案

## 最终共识

经过对需求的深入分析和讨论，我们达成以下共识：

1. **技术方案确认**: 使用Python 3.8+作为开发语言，集成N_m3u8DL-RE和ffmpeg作为核心工具，采用YAML格式进行配置管理。

2. **功能模块确认**: 实现7个核心功能模块（Cookie获取、链接解析、视频下载、视频合并、文件管理、配置管理、鉴权管理），以及完善的错误处理、日志系统和测试方案。

3. **文档结构确认**: 文档包含5个核心章节（功能实现、错误处理、日志系统、测试与验证、代码规范），以及目录、术语表、版本历史、参考文献等必要要素。

4. **技术约束确认**: 明确环境约束、依赖约束、安全约束和性能约束，确保技术方案的可行性和安全性。

5. **验收标准确认**: 制定明确的验收标准，包括文档完整性、文档质量、技术准确性、文档可读性和文档专业性。

6. **任务边界确认**: 明确包含和不包含的内容，确保任务范围清晰，避免过度设计。

所有关键决策点已确认，所有不确定性已解决，可以进入下一阶段的设计和实现工作。
