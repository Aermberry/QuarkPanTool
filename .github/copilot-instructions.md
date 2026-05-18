# Copilot Instructions for QuarkPanTool

## Quick Start

**Setup:**
```bash
uv sync                                    # Install dependencies to .venv
uv run playwright install firefox          # Install Firefox browser for Playwright
```

**Run the application:**
```bash
uv run python quark.py                     # Start the main CLI application
```

**Run individual scripts:**
```bash
uv run python api.py                       # Test API calls directly
uv run python download.py                  # Test download functions
uv run python share.py                     # Test file sharing functionality
```

## Project Overview

QuarkPanTool is a Quark Cloud Drive (夸克网盘) batch automation tool written in Python 3.10+. It provides CLI-based operations for:

- **File transfer**: Batch save files from shared links to your cloud drive
- **Share management**: Batch generate share links for cloud drive files
- **Local downloads**: Batch download cloud drive files using httpx or aria2c

The project uses **uv** as the dependency manager (defined in `pyproject.toml`) and **Playwright** for browser-based authentication.

## Architecture

### Core Components

1. **`quark.py`** - Main entry point and file manager class (`QuarkPanFileManager`)
   - Initializes browser automation with Playwright
   - Manages session state (folder IDs, cookies, user info)
   - Coordinates API calls and download operations
   - Supports two download modes: httpx (default) and aria2c (if available)

2. **`api.py`** - API client for Quark Cloud Drive
   - `get_pwd_id()` - Extract password ID from share URL
   - `get_stoken()` - Authenticate share link access with optional password
   - `get_detail()` - Retrieve file/folder structure from share link
   - `get_sorted_file_list()` - Get paginated file listings
   - `create_dir()` - Create directories in cloud drive
   - `get_share_save_task_id()` - Initiate file save operation
   - `submit_task()` - Complete file save transaction
   - `get_user_info()` - Retrieve current user account info

3. **`download.py`** - File download engine
   - `check_aria2c()` - Detect if aria2c is installed
   - `download_file_httpx()` - Download using httpx with progress bar
   - `download_file_aria2c()` - Download using aria2c (multi-threaded, resume support)
   - `quark_file_download()` - Main download orchestration
   - `get_download_urls()` - Request download URLs from API

4. **`share.py`** - File sharing operations
   - `share_run()` - Generate share links for files in a directory
   - `share_run_retry()` - Retry wrapper for share generation with backoff

5. **`quark_login.py`** - Authentication via Playwright
   - `QuarkLogin.login()` - Browser-based interactive login (stores cookies)
   - `QuarkLogin.transfer_cookies()` - Convert Playwright cookie format to HTTP headers
   - Saves cookies to `config/cookies.txt` for reuse

6. **`ui.py`** - Interactive CLI menu system
   - `print_menu()` - Display main menu with download mode indicator
   - `run_menu()` - Event loop for user interactions (8 menu options)
   - Validates user input and delegates to manager methods

7. **`utils.py`** - Shared utilities
   - `custom_print()` - Timestamped logging with color support (red for errors)
   - `read_config()` / `save_config()` - JSON/text config I/O
   - `get_datetime()` - Timestamp formatting
   - `get_timestamp()` - Unix timestamp (10s or 13s precision)

### Data Flow

```
User Input (ui.py)
    ↓
QuarkPanFileManager (quark.py) coordinates:
    ├─ API calls (api.py) - via async httpx
    ├─ Authentication (quark_login.py) - Playwright-based
    ├─ Downloads (download.py) - httpx or aria2c
    └─ Sharing (share.py) - batch link generation
    ↓
Config/State (config/ directory):
    ├─ config.json - download settings, menu state
    ├─ cookies.txt - Playwright-exported cookies
    └─ config.txt - user preferences
```

## Key Conventions

### Async/Await Pattern
- API functions are **async** (prefixed with `async def`), called via `asyncio.run()` or `await`
- Download operations use `async with httpx.AsyncClient()` for concurrent requests
- Main UI loop in `run_menu()` is async but called from sync context via `asyncio.run()`

### Headers and Authentication
- Custom User-Agent mimics Chrome/QQBrowser for API compatibility
- Cookies stored as plain text in `config/cookies.txt` (format: Playwright JSON list)
- All API calls include full headers dict with cookies, referer, and origin

### Error Handling
- `@retry` decorator (from `retrying` library) used for Playwright login and API resilience
- HTTP errors handled implicitly (httpx raises on 4xx/5xx if not caught)
- Download fallback: aria2c → httpx if aria2c mode fails

### Configuration Management
- `CONFIG_DIR = './config'` - Single config directory (created at module import)
- Settings loaded from JSON on startup, saved after menu changes
- Download mode (httpx vs aria2c) persists in `config.json`

### Share Link Format with Password
- Share URLs with passwords use query parameter: `https://pan.quark.cn/s/{id}?pwd={code}`
- Password extracted and passed separately to `get_stoken()` API call
- User must manually format URLs in `url.txt` before batch operations

### File Organization
- Downloaded files saved to `downloads/` directory by default (configurable via menu option 7)
- Bulk share/transfer lists read from `url.txt` (one URL per line)
- Playwright browser profile stored in `./web_browser_data/` (persistent login)

## CLI Menu System

The interactive menu system in `ui.py` is the primary user interface. All 8 options run inside an async event loop (`run_menu()`).

### Menu Options Reference

| # | Option | Input Source | Primary Action | State Persistence |
|---|--------|------|------|---|
| 1 | 分享地址转存文件 | `url.txt` or stdin | Batch/single save files from share URLs | None |
| 2 | 批量生成分享链接 | stdin or `share/retry.txt` | Generate share links for cloud files | `share/share_url.txt` |
| 3 | 切换网盘保存目录 | Interactive picker | Change cloud drive folder target | `config.json` (pdir_id) |
| 4 | 创建网盘文件夹 | stdin | Create new folder in cloud drive | None (immediate) |
| 5 | 下载到本地 | `url.txt`, stdin, or direct URLs | Download files locally | None |
| 6 | 登录 | Browser/interactive | Re-authenticate (refresh cookies) | `config/cookies.txt` |
| 7 | 设置下载路径 | stdin | Change local download directory | `config.json` (save_folder) |
| 8 | 下载模式设置 | stdin (submenu) | Switch httpx/aria2c, configure threads | `config.json` |

### Option 1: File Transfer (分享地址转存文件)

```
Submenu: Batch or Single?
├─ Batch (1):
│  ├─ Reads url.txt (one URL per line)
│  ├─ Confirms count and user approval (must enter "2" to confirm)
│  └─ Calls quark_file_manager.run() for each URL in to_dir_id
│
└─ Single (other):
   └─ Prompts for share URL via stdin
```

**Key behaviors:**
- URL validation: must be >20 characters (prevents accidental entry)
- Batch requires two confirmations (batch mode + count confirmation with "2")
- Creates `url.txt` if missing and exits with code -1
- Supports password-protected URLs: `https://pan.quark.cn/s/{id}?pwd={code}`

### Option 2: Batch Share Generation (批量生成分享链接)

```
Submenu: Share or Retry?
├─ Share (1):
│  └─ Takes folder webpage URL from stdin
│
└─ Retry (2):
   └─ Reads previous failed URLs from share/retry.txt
```

**Sub-options available:**
1. **Share Duration:** 1=1day, 2=7days, 3=30days, 4=permanent
2. **Encryption:** 1=no password, 2=encrypted with user-provided or random passcode
3. **Traverse Depth:** 0=root only, 1=1-level subdirs, 2=2-level subdirs

**Output:** Saves to `share/share_url.txt` (also backs up to `share/share_url_backup.txt`)

### Option 3: Switch Cloud Directory (切换网盘保存目录)

```
Calls: quark_file_manager.load_folder_id(renew=True)
├─ Displays available folders (from QuarkPan account)
├─ User selects target folder
└─ Updates to_dir_id and to_dir_name (used for subsequent operations)
```

**State:** Saved to `config.json` as `pdir_id` and `dir_name`
**Reset:** Changes when switching users (detected in `init_config()`)

### Option 4: Create Folder (创建网盘文件夹)

```
Input: Folder name (stdin)
├─ Validation: Empty names rejected
└─ Calls: await quark_file_manager.create_dir(name)
```

**API call:** `create_dir()` in `api.py` → immediate cloud drive folder creation

### Option 5: Download Files (下载到本地)

```
Input: Single URL, comma-separated URLs, or file path (e.g., url.txt)
├─ If file path given: calls load_url_file() to extract URLs via regex
├─ For each URL:
│  ├─ Check if direct download URL (pds.quark.cn) via is_direct_download_url()
│  ├─ If direct: await quark_file_manager.download_direct_url(url)
│  └─ Otherwise: await quark_file_manager.run(url, to_dir_id, download=True)
└─ Shows progress: "正在下载第 X/Y 个"
```

**Supported input formats:**
- Single URL: `https://pan.quark.cn/s/abc123`
- CSV URLs: `https://pan.quark.cn/s/abc123,https://pan.quark.cn/s/def456`
- File path: `url.txt` or any file with URLs (regex extracted)

### Option 6: Login (登录)

```
Action:
├─ Clears config/cookies.txt
├─ Calls quark_file_manager.reinit() (resets headers['cookie'])
├─ Triggers browser-based Playwright login via get_cookies()
└─ Saves new cookies to config/cookies.txt
```

**Flow:** Cookie → headers → subsequent API calls

### Option 7: Set Download Path (设置下载路径)

```
Input: New local directory path (stdin)
├─ Validation: Non-empty
├─ Updates: quark_file_manager.save_folder
└─ Persists to: config.json['save_folder']
```

**Default:** `downloads/` (created on first run if needed)

### Option 8: Download Mode Configuration (下载模式设置)

```
Main menu (aria2c available check):
├─ Shows current mode (httpx/aria2c)
├─ Shows aria2c availability
└─ Submenu:
   ├─ 1. Switch to aria2c (if available)
   ├─ 2. Switch to httpx
   ├─ 3. Set aria2c connections (1-64 range)
   └─ 4. Return to main menu

Config persistence: config.json
├─ download_mode: 'httpx' or 'aria2c'
├─ aria2c_connections: integer (default 16)
└─ aria2c_splits: integer (synced with connections)
```

**Fallback behavior:** If aria2c mode selected but not installed, automatically reverts to httpx with error message

### Input Validation Patterns

```python
# Menu choice validation
if input_text.strip() in ['q', 'Q']:  # Quit
if input_text.strip() in [str(i) for i in range(1, 9)]:  # 1-8

# URL validation
if url and len(url.strip()) > 20:  # Share URLs

# Confirmation patterns
if input_text and input_text.strip() == '2':  # Explicit "2" for batch confirmation
if share_option and share_option == '1':  # Submenu branching
```

### Adding New Menu Options

1. **Update `print_menu()`** - Add new option number and label to the display strings
2. **Add async handler** in `run_menu()` - Create elif branch for `input_text.strip() == 'X'`
3. **Implement logic** - Call `quark_file_manager` methods or new functions
4. **State persistence** - If needed, use `read_config()` / `save_config()` to `config.json`
5. **Test input/output** - Validate with edge cases (empty input, invalid types, file not found)

## Common Tasks

### Adding a new API endpoint
1. Add function to `api.py` with `async def`
2. Use `httpx.AsyncClient()` with the existing headers pattern
3. Handle `json_data['status']` for success/error detection
4. Call from `QuarkPanFileManager` method in `quark.py`

### Modifying download behavior
- Multi-threaded: Change `connections` parameter in `download_file_aria2c()` (1-64 range)
- Progress display: Adjust `tqdm` parameters in `download_file_httpx()` or `download_file_aria2c()`
- Resume support: Already enabled for aria2c; httpx doesn't support resume out-of-box

### Debugging API responses
- `custom_print()` outputs JSON responses with timestamps
- Check headers (User-Agent, referer) match Quark's expectations
- Verify stoken is valid before calling `get_detail()` or `get_sorted_file_list()`

### UI menu additions
1. Add option number and label to `print_menu()` string
2. Implement handler method in `QuarkPanFileManager`
3. Add case/conditional in `run_menu()` event loop
4. Save state to `config.json` if needed

## Dependencies

- **httpx** - Async HTTP client for API calls
- **playwright** - Browser automation for login
- **prettytable** - CLI table formatting
- **tqdm** - Progress bars for downloads
- **colorama** - Colored console output
- **retrying** - Retry decorator for resilience
- **aria2c** - Optional external tool for accelerated downloads (must be in PATH)

## Deployment Notes

- Requires Python 3.10+ (`pyproject.toml` enforces `requires-python = ">=3.10"`)
- First run initializes `.venv` and downloads Firefox binary (via Playwright)
- Interactive login required unless cookies pre-populated in `config/cookies.txt`
- Cookies expire; old cookies may need refresh via menu option 6 (login)
- For Linux environments without browser display, manually extract cookies and place in `config/cookies.txt`
