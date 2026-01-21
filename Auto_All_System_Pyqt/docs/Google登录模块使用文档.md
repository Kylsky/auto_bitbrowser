# Google登录模块使用文档

## 模块概览

Google登录模块位于 `src/google/backend/`，提供完整的Google账号登录和状态检测功能。

## 核心模块

### 1. `google_auth.py` - 登录状态检测

提供底层的登录状态检测和认证功能。

#### GoogleLoginStatus 枚举

```python
from google.backend import GoogleLoginStatus

# 可能的状态值
GoogleLoginStatus.LOGGED_IN       # 已登录
GoogleLoginStatus.NOT_LOGGED_IN   # 未登录
GoogleLoginStatus.NEED_PASSWORD   # 需要密码
GoogleLoginStatus.NEED_2FA        # 需要2FA
GoogleLoginStatus.NEED_RECOVERY   # 需要辅助邮箱
GoogleLoginStatus.SESSION_EXPIRED # 会话过期
GoogleLoginStatus.SECURITY_CHECK  # 安全检查
GoogleLoginStatus.UNKNOWN         # 未知状态
```

#### 检测登录状态

```python
from google.backend import check_google_login_status, is_logged_in
from playwright.async_api import Page

async def example(page: Page):
    # 方法1: 详细检测
    status, info = await check_google_login_status(page)
    if status == GoogleLoginStatus.LOGGED_IN:
        print(f"已登录: {info.get('email')}")
    elif status == GoogleLoginStatus.NEED_2FA:
        print("需要2FA验证")
    
    # 方法2: 快速检测
    if await is_logged_in(page):
        print("已登录")
```

#### 执行登录

```python
from google.backend import google_login

async def login_example(page: Page):
    account_info = {
        'email': 'user@gmail.com',
        'password': 'password123',
        'secret': '2FA_SECRET_KEY',        # 可选
        'backup': 'recovery@email.com'     # 可选
    }
    
    success, message = await google_login(page, account_info)
    if success:
        print(f"登录成功: {message}")
    else:
        print(f"登录失败: {message}")
```

### 2. `google_login_service.py` - 登录服务

提供高级的登录服务，集成比特浏览器。

#### GoogleLoginService 类

```python
from google.backend import GoogleLoginService

# 创建服务实例
service = GoogleLoginService(log_callback=print)

# 同步登录比特浏览器窗口
success, status, message = service.login_browser_sync(
    browser_id="abc123",
    account_info=None,  # None表示从数据库获取
    target_url="https://one.google.com",  # 登录后跳转
    close_after=True    # 完成后关闭浏览器
)

if success:
    print(f"登录成功: {message}")
```

#### 便捷函数

```python
from google.backend import (
    login_google_account,
    check_browser_login_status
)

# 快速登录（最简单的方式）
success, status, msg = login_google_account("browser_id_123")
if success:
    print("登录成功！")

# 检查浏览器登录状态
status, info = check_browser_login_status("browser_id_123")
print(f"状态: {status}, 邮箱: {info.get('email')}")
```

#### 批量检查登录状态

```python
import asyncio
from google.backend import GoogleLoginService

async def batch_check():
    service = GoogleLoginService()
    
    browser_ids = ["id1", "id2", "id3"]
    
    def callback(browser_id, status, info):
        print(f"{browser_id}: {status}")
    
    results = await service.batch_check_login_status(
        browser_ids,
        callback=callback
    )
    
    for bid, (status, info) in results.items():
        print(f"{bid}: {status} - {info.get('email', 'N/A')}")

asyncio.run(batch_check())
```

## 使用场景

### 场景1: 单个账号登录

```python
from google.backend import login_google_account

def login_single_account():
    # 方式1: 从数据库自动获取账号信息
    success, status, msg = login_google_account("browser_id")
    
    # 方式2: 手动提供账号信息
    account_info = {
        'email': 'test@gmail.com',
        'password': 'pwd123',
        'secret': 'ABCD1234EFGH5678'
    }
    success, status, msg = login_google_account(
        "browser_id",
        account_info=account_info
    )
    
    return success
```

### 场景2: 检查是否需要重新登录

```python
from google.backend import check_browser_login_status, GoogleLoginStatus

def check_and_relogin_if_needed(browser_id):
    status, info = check_browser_login_status(browser_id)
    
    if status == GoogleLoginStatus.LOGGED_IN:
        print(f"已登录: {info.get('email')}")
        return True
    
    elif status == GoogleLoginStatus.SESSION_EXPIRED:
        print("会话过期，重新登录...")
        from google.backend import login_google_account
        success, _, _ = login_google_account(browser_id)
        return success
    
    else:
        print(f"状态异常: {status}")
        return False
```

### 场景3: 带日志的登录

```python
from google.backend import GoogleLoginService

def login_with_logging(browser_id):
    # 自定义日志函数
    def my_logger(message):
        with open('login.log', 'a') as f:
            f.write(f"{message}\n")
        print(message)
    
    service = GoogleLoginService(log_callback=my_logger)
    
    success, status, message = service.login_browser_sync(
        browser_id,
        close_after=False  # 保持浏览器打开
    )
    
    return success
```

### 场景4: 使用Playwright直接操作

```python
from playwright.async_api import async_playwright
from google.backend import GoogleLoginService
import asyncio

async def custom_login_flow():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False)
        page = await browser.new_page()
        
        service = GoogleLoginService()
        
        account_info = {
            'email': 'user@gmail.com',
            'password': 'pass123'
        }
        
        success, status, msg = await service.login_with_playwright(
            page, 
            account_info,
            force_login=True  # 强制重新登录
        )
        
        if success:
            # 登录成功后继续其他操作
            await page.goto('https://drive.google.com')
            await page.screenshot(path='drive.png')
        
        await browser.close()

asyncio.run(custom_login_flow())
```

## 账号信息格式

账号信息字典支持以下字段：

```python
account_info = {
    # 必填
    'email': 'user@gmail.com',
    'password': 'password123',
    
    # 可选 - 辅助邮箱（多个别名都支持）
    'backup': 'recovery@email.com',
    'backup_email': 'recovery@email.com',
    'recovery_email': 'recovery@email.com',
    
    # 可选 - 2FA密钥（多个别名都支持）
    'secret': 'ABCD1234EFGH5678',
    '2fa_secret': 'ABCD1234EFGH5678',
    'secret_key': 'ABCD1234EFGH5678',
}
```

## Google One 状态检测

```python
from google.backend import check_google_one_status
from playwright.async_api import Page

async def check_student_offer(page: Page):
    # 先导航到Google One页面
    await page.goto("https://one.google.com/ai-student")
    
    # 检测状态
    status, link = await check_google_one_status(page, timeout=10)
    
    if status == "subscribed":
        print("已订阅")
    elif status == "verified":
        print("已验证但未绑卡")
    elif status == "link_ready":
        print(f"有SheerID链接: {link}")
    elif status == "ineligible":
        print("无资格")
    else:
        print(f"超时或错误: {status}")
```

## 错误处理

```python
from google.backend import login_google_account, GoogleLoginStatus

def safe_login(browser_id):
    try:
        success, status, message = login_google_account(browser_id)
        
        if not success:
            # 根据状态处理错误
            if status == "error":
                print(f"系统错误: {message}")
            elif status == "login_failed":
                print(f"登录失败: {message}")
                # 可能需要更新账号信息
            else:
                print(f"未知状态: {status} - {message}")
        
        return success
        
    except Exception as e:
        print(f"异常: {e}")
        return False
```

## 最佳实践

### 1. 总是检查登录状态后再操作

```python
from google.backend import check_browser_login_status, login_google_account

def ensure_logged_in(browser_id):
    status, _ = check_browser_login_status(browser_id)
    
    if status != "logged_in":
        success, _, _ = login_google_account(browser_id)
        return success
    
    return True
```

### 2. 使用日志回调跟踪进度

```python
def progress_logger(message):
    print(f"[{time.strftime('%H:%M:%S')}] {message}")

service = GoogleLoginService(log_callback=progress_logger)
```

### 3. 批量操作时添加延迟

```python
import time

browser_ids = ["id1", "id2", "id3"]

for bid in browser_ids:
    login_google_account(bid)
    time.sleep(2)  # 避免API过载
```

## 注意事项

1. **2FA密钥格式**: 必须是Base32编码的字符串（如 "ABCD1234EFGH5678"）
2. **比特浏览器API**: 确保比特浏览器正在运行（默认端口54345）
3. **数据库依赖**: 使用`None`作为account_info时，需要数据库中有对应的browser_id记录
4. **超时设置**: 默认超时为30秒，网络差时可能需要增加
5. **并发限制**: 批量操作时建议控制并发数量，避免API限流

## 与旧代码的迁移

如果你正在使用旧的`bit_playwright.py`或`run_playwright_google.py`：

```python
# 旧代码
from bit_playwright import google_login
await google_login(page, account_info)

# 新代码（推荐）
from google.backend import GoogleLoginService
service = GoogleLoginService()
success, status, msg = await service.login_with_playwright(page, account_info)
```

## 调试技巧

### 启用详细日志

```python
def verbose_logger(message):
    print(f"[DEBUG] {message}")

service = GoogleLoginService(log_callback=verbose_logger)
```

### 检查页面状态

```python
from playwright.async_api import Page

async def debug_page(page: Page):
    print(f"URL: {page.url}")
    print(f"Title: {await page.title()}")
    
    from google.backend import check_google_login_status
    status, info = await check_google_login_status(page)
    print(f"Status: {status}")
    print(f"Info: {info}")
```

---

## 总结

Google登录模块提供了从底层状态检测到高级服务的完整解决方案：

- **底层**: `google_auth.py` - 精确的状态检测
- **中层**: `google_login()` - 基础登录流程
- **高层**: `GoogleLoginService` - 完整的服务封装
- **便捷**: `login_google_account()` - 一行代码登录

根据你的需求选择合适的API层级使用。
