# Calibre Windows 构建指南（纯 Windows 环境）

本文档描述如何在真实 Windows 机器上，从源码构建 calibre Windows 安装文件（MSI + 便携版）。

源码仓库：`https://github.com/delubee/calibre.git`

> **实测环境**：Windows 11 21H2 / VS 2026 (MSVC 14.51) / Python 3.14（内嵌）/ calibre 9.12.0
> 最终产物：`C:\r\sw64\dist\calibre-64bit-9.12.0.msi`（195 MB）

---

## 1. 硬件要求

| 项目 | 最低要求 | 推荐 |
|------|----------|------|
| 磁盘空间 | 120GB | 200GB+ |
| 内存 | 8GB | 16GB+ |
| CPU | 4 核 | 8 核+ |
| 系统 | Windows 10/11 64-bit | Windows 11 |

---

## 2. 软件安装

### 2.1 Visual Studio 2026 Community Edition

下载地址：https://visualstudio.microsoft.com/

安装时勾选以下组件：

- .NET SDK
- C++ ATL for latest build tools (x86 & x64)
- C++ Clang Compiler for Windows
- C++ CMake tools for Windows
- C++/CLI support for build tools
- Git for Windows
- MSBuild
- MSBuild support for LLVM (clang-cl) toolset
- MSVC vXXX - VS C++ x64/x86 build tools
- Windows 11 SDK

安装完成后，将以下目录加入系统 PATH：

```
C:\Program Files (x86)\Microsoft Visual Studio\Installer
C:\Program Files\Microsoft Visual Studio\VC\Tools\MSVC\*\bin\Hostx64\x64
C:\Program Files\Microsoft Visual Studio\VC\Tools\Llvm\bin
```

> 验证：打开命令提示符，运行 `cl.exe` 和 `link.exe` 应能正常输出。

### 2.2 Python

安装 Python 3.13+（64-bit），确保 `py.exe` 在 PATH 中。

```cmd
py.exe -m pip install certifi html5lib
```

### 2.3 其他工具

| 工具 | 说明 |
|------|------|
| Ruby | 安装 Ruby 3.4 x64（不带 devkit），默认路径 `C:\Ruby34-x64` |
| NodeJS | 安装 LTS 版本 |
| Strawberry Perl | 默认路径 `C:\Strawberry` |
| MESON + NINJA | 从 https://github.com/mesonbuild/meson/releases 下载 MSI 安装 |
| .NET SDK | 随 VS 安装，用于 WiX |

### 2.4 WiX Toolset

在 Visual Studio 命令提示符中运行：

```cmd
dotnet tool install --global wix
```

验证：

```cmd
%USERPROFILE%\.dotnet\tools\wix.exe --version
```

**重要：WiX v7 需要接受 EULA 并预装扩展**（否则构建时才会报错，浪费时间）：

```cmd
:: 接受 EULA（只需执行一次）
%USERPROFILE%\.dotnet\tools\wix.exe eula accept wix7

:: 预装构建所需扩展（需要网络）
%USERPROFILE%\.dotnet\tools\wix.exe extension add -g WixToolset.Util.wixext
%USERPROFILE%\.dotnet\tools\wix.exe extension add -g WixToolset.UI.wixext
```

> 如果不接受 EULA，构建到 MSI 打包阶段会报错：
> `error WIX7015: You must accept the Open Source Maintenance Fee (OSMF) EULA to use WiX Toolset v7`

### 2.5 Mesa OpenGL

下载 `opengl32sw.dll`：
https://download.qt.io/development_releases/prebuilt/llvmpipe/windows/

放置到：

```
C:\mesa\64\opengl32sw.dll
```

### 2.6 RapydScript-NG（JavaScript 编译器）

calibre 的 Web 组件（阅读器、编辑器）使用 RapydScript 编写，构建时需要编译为 JavaScript。
内嵌编译器依赖 Qt WebEngine，在无头环境下容易超时失败。**强烈建议安装外部编译器**：

```cmd
npm install -g rapydscript-ng
```

验证：

```cmd
rapydscript --version
```

> 安装后 `setup.py resources` 会自动检测并使用外部编译器，绕过 WebEngine。

---

## 3. 目录结构规划

```
C:\r\                     ← 构建根目录
├── src\                  ← calibre 源码（delubee/calibre）
├── sw64\
│   └── sw\              ← 预编译依赖库（SW 环境变量指向此处）
│       ├── bin\
│       ├── lib\
│       ├── include\
│       ├── qt\
│       ├── private\
│       │   └── python\  ← 内嵌 Python
│       └── ffmpeg\
└── build\               ← 构建输出
```

---

## 4. 环境搭建步骤

### 4.1 创建目录

```cmd
mkdir C:\r
mkdir C:\r\sw64\sw
mkdir C:\r\build
```

### 4.2 克隆源码

```cmd
cd C:\r
git clone https://github.com/delubee/calibre.git src
```

### 4.3 下载预编译依赖包

这是最关键的一步。calibre 官方 CI 提供了预编译好的 Windows 依赖包，
包含 Qt、ICU、OpenSSL、Python、ffmpeg 等所有 C/C++ 依赖库。

```cmd
cd C:\r\sw64\sw
curl -L -o windows-64.tar.xz https://download.calibre-ebook.com/ci/calibre7/windows-64.tar.xz
tar -xf windows-64.tar.xz
del windows-64.tar.xz
```

> 注意：该文件较大（约 274MB 压缩 / 2.4GB 解压），下载和解压需要时间。
> 解压后 `C:\r\sw64\sw` 下应包含 `bin`、`lib`、`include`、`qt`、`private` 等目录。

**❗ 重要：依赖包可能不完整，需要补充缺失的 DLL**

实测发现 `windows-64.tar.xz` 缺少以下关键文件：

| 缺失文件 | 说明 |
|-----------|------|
| `icudt78.dll`, `icuin78.dll`, `icuuc78.dll`, `icuio78.dll`, `icutu78.dll` | ICU Unicode 库 |
| `espeak-ng.dll` | TTS 语音合成 |
| `freetype.dll` | 字体渲染 |
| `jpeg8.dll` | JPEG 图像处理 |
| `lcms2-2.dll` | 色彩管理 |
| `libffi-8.dll` | 外部函数接口（已在 Python DLLs 中，不需重复添加） |
| `brotlicommon.dll`, `brotlidec.dll`, `brotlienc.dll` | Brotli 压缩 |
| `sqlite3.dll` | 数据库（已在 Python DLLs 中） |
| 39 个 SQLite 扩展 DLL | `amatch.dll`, `regexp.dll`, `uuid.dll` 等 |
| `jpegtran-calibre.exe`, `cjpeg-calibre.exe` | JPEG 优化工具 |
| `cwebp-calibre.exe`, `JXRDecApp-calibre.exe` | WebP/JXR 图像工具 |

**解决方法：从官方 calibre MSI 中提取缺失文件**

```powershell
# 1. 下载当前版本的官方 MSI
Invoke-WebRequest -Uri "https://download.calibre-ebook.com/9.12.0/calibre-64bit-9.12.0.msi" -OutFile "C:\r\calibre-official.msi"

# 2. 管理安装解压 MSI
Start-Process msiexec -ArgumentList '/a', 'C:\r\calibre-official.msi', '/qn', 'TARGETDIR=C:\r\calibre-extracted' -Wait

# 3. 复制缺失的 DLL 到依赖目录
$src = "C:\r\calibre-extracted\PFiles64\Calibre2\app\bin"
Copy-Item "$src\icu*.dll" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\espeak-ng.dll" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\freetype.dll" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\jpeg8.dll" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\lcms2-2.dll" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\brotli*.dll" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\*-calibre.exe" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\JXRDecApp-calibre.exe" "C:\r\sw64\sw\bin\" -Force

# 4. 复制 SQLite 扩展 DLL（可选，用于 FTS 搜索等功能）
$sqliteExts = @('amatch','anycollseq','appendvfs','base64','base85','btreeinfo',
  'cksumvfs','closure','completion','compress','csv','decimal','eval','fileio',
  'fuzzer','ieee754','memstat','nextchar','noop','prefixes','randomjson',
  'regexp','rot13','sha1','shathree','spellfix','sqlar','sqlite3_tool',
  'stmt','stmtrand','tmstmpvfs','uint','unionvtab','uuid','vec1',
  'vfsstat','vtablog','zipfile','zorder')
foreach ($d in $sqliteExts) { Copy-Item "$src\$d.dll" "C:\r\sw64\sw\bin\" -Force }
```

> **注意**：不要复制 `libffi-8.dll` 和 `sqlite3.dll` 到 `sw64\sw\bin`，
> 它们已存在于 `sw64\sw\private\python\DLLs\` 中，重复会导致构建时权限冲突。

### 4.4 设置环境变量

创建系统环境变量（或在构建脚本中临时设置）：

```cmd
set SW=C:\r\sw64\sw
set QMAKE=C:\r\sw64\sw\qt\bin\qmake.exe
set CALIBRE_QT_PREFIX=C:\r\sw64\sw\qt
set OPENSSL_MODULES=C:\r\sw64\sw\lib\ossl-modules
set PIPER_TTS_DIR=C:\r\sw64\sw\piper
set CALIBRE_ESPEAK_DATA_DIR=C:\r\sw64\sw\share\espeak-ng-data
set MESA=C:\mesa
```

将以下路径加入 PATH（优先级从高到低）：

```cmd
set PATH=C:\r\sw64\sw\private\python\bin;%PATH%
set PATH=C:\r\sw64\sw\private\python\Lib\site-packages\pywin32_system32;%PATH%
set PATH=C:\r\sw64\sw\bin;%PATH%
set PATH=C:\r\sw64\sw\qt\bin;%PATH%
```

---

## 5. Bootstrap（准备源码）

Bootstrap 将原始 git 源码准备为可构建状态，包括：
- 编译 C/C++ 扩展模块
- 生成 ISO 语言代码数据
- 编译翻译文件
- 构建 GUI 资源（包含 RapydScript 编译）
- 下载 CA 证书

```cmd
cd C:\r\src
py.exe setup.py bootstrap --ephemeral
```

> `--ephemeral` 参数跳过翻译仓库的完整历史下载，加快速度。

如果 bootstrap 过程中某些步骤失败，可以单独运行子命令：

```cmd
py.exe setup.py build          :: 编译 C/C++ 扩展
py.exe setup.py iso639         :: ISO 639 语言代码
py.exe setup.py iso3166        :: ISO 3166 国家代码
py.exe setup.py translations   :: 编译翻译
py.exe setup.py gui            :: GUI 资源
py.exe setup.py resources      :: 其他资源（包含 RapydScript 编译）
py.exe setup.py cacerts        :: CA 证书
```

### 5.1 已知问题：RapydScript 编译失败

如果 `setup.py resources` 报错：

```
CompileFailure: 'fs_images' module doesn't exist in import directories: __stdlib__
```

或：

```
TimeoutError: Creating RapydScript compiler took too long
```

这是因为内嵌编译器依赖 Qt WebEngine，在无头/远程环境下容易失败。

**解决方法**：安装外部 rapydscript-ng 编译器（见 2.6 节）：

```cmd
npm install -g rapydscript-ng
```

安装后重新运行 `py.exe setup.py resources` 即可。

### 5.2 已知问题：安装后只有英文，无中文语言选项

如果安装后 calibre 语言选择中没有中文，原因是翻译文件未编译。

检查以下文件是否存在：

```cmd
dir C:\r\src\resources\localization\locales.zip
dir C:\r\src\resources\localization\stats.calibre_msgpack
```

如果不存在，说明 bootstrap 时翻译仓库克隆失败（网络问题）。手动修复：

```cmd
cd C:\r\src
git clone --depth=1 https://github.com/kovidgoyal/calibre-translations.git translations
py.exe setup.py translations
```

编译成功后会生成 `locales.zip`（约 17MB）和 `stats.calibre_msgpack`，重新构建 MSI 即可。

> **注意**：`--ephemeral` 参数会浅克隆翻译仓库，但如果网络不稳定可能静默失败。
> 建议 bootstrap 后检查 `C:\r\src\translations` 目录是否存在。

### 5.3 已知问题：manual/locale/completed.json 缺失

如果报错 `FileNotFoundError: manual/locale/completed.json`，创建空文件：

```powershell
[System.IO.File]::WriteAllText('C:\r\src\manual\locale\completed.json', '{}')
```

> 注意：必须使用无 BOM 的 UTF-8 编码，不能用 `Set-Content -Encoding UTF8`（会加 BOM）。

---

## 6. 构建安装文件

### 6.1 方式一：通过 bypy 直接构建（实测可行 ✔）

克隆 bypy 仓库：

```cmd
cd C:\r
git clone https://github.com/kovidgoyal/bypy.git
```

创建构建脚本 `C:\r\run_build.ps1`：

```powershell
$env:PYTHONIOENCODING = 'utf-8'
$env:PYTHONUTF8 = '1'
$env:BUILD_ARCH = '64'
$env:BYPY_ROOT = 'C:\r'
$env:PATH = 'C:\r\sw64\sw\private\python;C:\r\sw64\sw\private\python\Lib\site-packages\pywin32_system32;C:\r\sw64\sw\bin;C:\r\sw64\sw\qt\bin;C:\Ruby34-x64\bin;C:\Program Files\nodejs;C:\Windows\System32;C:\Windows'
Set-Location C:\r\src
& 'C:\r\sw64\sw\private\python\python.exe' 'C:\r\bypy' 'BYPY_ROOT=C:\r' 'BUILD_ARCH=64' 'BYPY_ARCH=windows-64' 'PERL=perl' 'RUBY=C:\Ruby34-x64\bin\ruby.exe' 'MESA=C:\mesa' 'NODEJS=node' program --skip-tests
```

执行：

```powershell
powershell -ExecutionPolicy Bypass -File "C:\r\run_build.ps1"
```

参数说明：
- `program`：bypy 的构建子命令，执行完整的冻结 + 打包流程
- `--skip-tests`：跳过构建测试（测试需要 `pyzstd`、`winsapi` 等可选模块，缺失不影响主流程）
- `BYPY_ROOT=C:\r`：bypy 根目录
- `BUILD_ARCH=64`：构建 64 位

输出文件位于 `C:\r\sw64\dist\`：
- `calibre-64bit-<version>.msi` — MSI 安装包

### 6.2 方式二：通过 setup.py win64（需要修改）

原始命令：

```cmd
cd C:\r\src
py.exe setup.py win64 --dont-sign --dont-notarize --dont-shutdown
```

> **注意**：此方式会尝试通过 SSH 连接虚拟机（bypy 的 VM 管理层），
> 在纯 Windows 环境下会报错 `ValueError: Not a valid SSH URL`。
> 必须使用方式一直接调用 bypy 的 `program` 子命令。

### 6.3 必要的代码修改

在纯 Windows 环境下构建，需要对 bypy 做以下修改：

#### 修改 1：跳过依赖重新安装（`C:\r\bypy\bypy\deps.py`）

依赖已解压就位后，`install_packages()` 会试图清除并重新安装，导致 DLL 锁定报错。
在 `install_packages` 函数开头添加：

```python
def install_packages(which_deps, dest_dir=PREFIX):
    # 如果依赖已存在则跳过
    if os.path.isdir(os.path.join(dest_dir, 'bin')) and os.listdir(os.path.join(dest_dir, 'bin')):
        print(f'Dependencies already present in {dest_dir}, skipping install_packages')
        return
    # ... 原始代码 ...
```

#### 修改 2：禁用 run_shell()（`C:\r\bypy\bypy\main.py` 和 `C:\r\src\bypy\init_env.py`）

构建失败时 bypy 会尝试启动交互式 shell（`C:/cygwin64/bin/zsh`），在没有 Cygwin 的环境下会崩溃。
将所有 `run_shell()` 调用替换为 `pass`。

#### 修改 3：跳过缺失的可选二进制（`C:\r\src\bypy\windows\__main__.py`）

`freeze()` 函数中复制工具 exe 时，改为存在才复制：

```python
for x in ('pdftohtml', 'pdfinfo', ...):
    exe_path = os.path.join(bindir, x + '.exe')
    if os.path.exists(exe_path):
        copybin(exe_path)
    else:
        print(f'WARNING: skipping missing optional binary: {x}.exe')
```

#### 修改 4：避免 DLL 重复复制冲突（`C:\r\src\bypy\windows\__main__.py`）

`copybin()` 函数添加已存在检查：

```python
def copybin(x, dest=env.dll_dir):
    dst = os.path.join(dest, os.path.basename(x)) if os.path.isdir(dest) else dest
    if os.path.exists(dst):
        return  # 跳过已复制的文件
    shutil.copy2(x, dest)
```

---

## 7. 构建产物

构建成功后，在 `C:\r\sw64\dist` 目录下会生成：

| 文件 | 说明 |
|------|------|
| `calibre-64bit-9.12.0.msi` | Windows 64-bit MSI 安装包（约 195MB） |

> 便携版安装程序需要代码签名服务器（`SIGN_SERVER_PORT` 环境变量），
> 在未配置签名服务的环境下不会生成。MSI 安装包不受影响。

---

## 8. 故障排除

### 8.1 找不到 Visual Studio

```
Could not find vswhere.exe
```

确认 Visual Studio Installer 已安装，且路径
`C:\Program Files (x86)\Microsoft Visual Studio\Installer\vswhere.exe` 存在。

### 8.2 找不到 Qt / qmake

```
Failed to run qmake
```

确认 `QMAKE` 环境变量指向正确路径：

```cmd
set QMAKE=C:\r\sw64\sw\qt\bin\qmake.exe
%QMAKE% -query QT_VERSION
```

### 8.3 依赖包下载失败

CI 依赖包地址：
```
https://download.calibre-ebook.com/ci/calibre7/windows-64.tar.xz
```

如果网络不稳定，可尝试：
- 使用代理下载
- 分多次下载（curl 支持 `-C -` 断点续传）
- 手工下载后放到 `D:\下载\windows-64.tar.xz`，再用 7-Zip 解压到 `C:\r\sw64\sw`

```cmd
curl -L -C - -o windows-64.tar.xz https://download.calibre-ebook.com/ci/calibre7/windows-64.tar.xz
```

### 8.4 WiX 构建失败

**问题 1：EULA 未接受**

```
error WIX7015: You must accept the Open Source Maintenance Fee (OSMF) EULA
```

解决：
```cmd
%USERPROFILE%\.dotnet\tools\wix.exe eula accept wix7
```

**问题 2：扩展未安装**

```
Command '['wix.exe', 'extension', 'add', '-g', 'WixToolset.Util.wixext']' returned non-zero exit status 1
```

解决：
```cmd
%USERPROFILE%\.dotnet\tools\wix.exe extension add -g WixToolset.Util.wixext
%USERPROFILE%\.dotnet\tools\wix.exe extension add -g WixToolset.UI.wixext
```

### 8.5 opengl32sw.dll 未找到

```
Mesa DLLs (opengl32sw.dll) not found
```

确认文件存在：

```cmd
dir C:\mesa\64\opengl32sw.dll
```

### 8.6 UnicodeEncodeError: 'gbk' codec can't encode

```
UnicodeEncodeError: 'gbk' codec can't encode character '\ufffd'
```

原因：MSVC 编译器输出包含非 GBK 字符，而控制台默认使用 GBK 编码。

解决：在构建脚本中设置：

```powershell
$env:PYTHONIOENCODING = 'utf-8'
$env:PYTHONUTF8 = '1'
```

### 8.7 PermissionError: 拒绝访问（DLL 锁定）

```
PermissionError: [WinError 5] 拒绝访问 'C:\\t\\t\\build-xxx\\winfrozen\\app\\bin\\xxx.dll'
```

原因：bypy 使用 `C:\t\t` 作为临时目录，上次构建的 DLL 被 Windows Defender 或系统锁定。

解决：
```powershell
# 重命名旧临时目录（而非删除）
Rename-Item "C:\t\t" "C:\t\t_old"
New-Item -ItemType Directory -Path "C:\t\t" -Force
```

> 如果重命名也失败，可能需要重启机器或临时禁用 Windows Defender 实时保护。

### 8.8 ImportError: DLL load failed while importing icu

```
ImportError: DLL load failed while importing icu: 找不到指定的模块
```

原因：依赖包缺少 ICU DLL（`icudt78.dll` 等）。

解决：见 4.3 节“从官方 MSI 提取缺失文件”。

### 8.9 KeyError: 'SIGN_SERVER_PORT'

```
KeyError: 'SIGN_SERVER_PORT'
```

原因：便携版打包步骤尝试连接代码签名服务器。

解决：这不影响 MSI 生成。MSI 在此步骤之前已完成。
如需完整构建便携版，需配置签名服务或修改 `bypy/windows/__main__.py` 跳过签名。

### 8.10 ValueError: Not a valid SSH URL

```
ValueError: Not a valid SSH URL: C:\r\src\bypy\b\windows\vm
```

原因：`setup.py win64` 调用 bypy 的 VM 管理层，试图 SSH 到虚拟机。

解决：不使用 `setup.py win64`，改用 6.1 节的方式直接调用 bypy `program` 子命令。

### 8.11 ModuleNotFoundError: No module named 'PyKCS11'

```
ModuleNotFoundError: No module named 'PyKCS11'
```

原因：`--dont-sign` 但未加 `--dont-notarize`，仍触发签名流程。

解决：同时使用 `--dont-sign --dont-notarize`，或直接使用 6.1 节方式。

---

## 9. 完整构建脚本（一键执行）

将以下内容保存为 `C:\r\run_build.ps1`：

```powershell
# ============================================
#   Calibre Windows Build Script (PowerShell)
# ============================================

# === 编码设置（解决 GBK 编码报错） ===
$env:PYTHONIOENCODING = 'utf-8'
$env:PYTHONUTF8 = '1'

# === 配置区 ===
$ROOT = 'C:\r'
$SRC = "$ROOT\src"
$SW = "$ROOT\sw64\sw"
$PYTHON = "$SW\private\python\python.exe"

# === 环境变量 ===
$env:BUILD_ARCH = '64'
$env:BYPY_ROOT = $ROOT

# === PATH 设置（仅保留必要路径，避免干扰） ===
$env:PATH = "$SW\private\python;$SW\private\python\Lib\site-packages\pywin32_system32;$SW\bin;$SW\qt\bin;C:\Ruby34-x64\bin;C:\Program Files\nodejs;C:\Windows\System32;C:\Windows"

# === 检查依赖 ===
if (-not (Test-Path "$SW\qt\bin\qmake.exe")) {
    Write-Error "Dependencies not found at $SW. Please extract windows-64.tar.xz first."
    exit 1
}
if (-not (Test-Path "$SRC\setup.py")) {
    Write-Error "Calibre source not found at $SRC"
    exit 1
}

# === 执行构建 ===
Set-Location $SRC
Write-Host "Starting calibre build..."
& $PYTHON "$ROOT\bypy" "BYPY_ROOT=$ROOT" 'BUILD_ARCH=64' 'BYPY_ARCH=windows-64' 'PERL=perl' "RUBY=C:\Ruby34-x64\bin\ruby.exe" 'MESA=C:\mesa' 'NODEJS=node' program --skip-tests

if ($LASTEXITCODE -eq 0) {
    Write-Host "`nBuild complete!" -ForegroundColor Green
    Write-Host "Output: $ROOT\sw64\dist\"
    Get-ChildItem "$ROOT\sw64\dist\*.msi" | ForEach-Object { Write-Host "  $_" }
} else {
    Write-Host "`nBuild failed with exit code $LASTEXITCODE" -ForegroundColor Red
}
```

执行：

```cmd
powershell -ExecutionPolicy Bypass -File "C:\r\run_build.ps1"
```

> 构建日志建议重定向到文件以便排查问题：
> ```cmd
> powershell -ExecutionPolicy Bypass -File "C:\r\run_build.ps1" > C:\r\build_log.txt 2>&1
> ```

---

## 10. 注意事项

1. **预编译依赖包版本匹配**：CI 依赖包是为 calibre 7.x/9.x 系列构建的，
   如果你的源码版本差异较大，可能需要自行编译依赖。

2. **代码签名**：未签名的安装程序在 Windows SmartScreen 中会显示警告，但功能完全正常。
   便携版需要签名服务器，无签名环境下只能生成 MSI。

3. **构建时间**：使用预编译依赖包时，整个构建过程约 30-60 分钟
   （主要耗时在 bootstrap 编译 C 扩展和 WiX 打包）。

4. **磁盘空间**：构建过程中会产生大量临时文件（`C:\t\t` 目录可达数 GB），
   确保有足够空间。构建完成后可清理 `C:\t\t` 目录。

5. **bypy 仓库**：bypy 是构建编排工具，必须与 calibre 源码配合使用。
   不要使用 `setup.py win64`，直接调用 bypy 的 `program` 子命令。

6. **临时目录**：bypy 硬编码使用 `C:\t\t` 作为临时目录（见 `bypy/bypy/constants.py`）。
   如果上次构建失败，该目录可能被锁定，需要重命名后重建。

7. **Windows Defender**：实时保护可能锁定构建产物中的 DLL，导致清理失败。
   建议将 `C:\t\t` 和 `C:\r` 加入排除列表。

8. **PowerShell 编码**：在中文 Windows 上，必须设置 `PYTHONIOENCODING=utf-8`，
   否则编译器输出会导致 GBK 编码错误。

9. **依赖包不完整**：官方 `windows-64.tar.xz` 缺少部分 DLL（特别是 ICU），
   需要从官方 MSI 中提取补充（见 4.3 节）。

10. **构建测试**：`--skip-tests` 跳过的测试包括：
    - `pyzstd` 模块缺失（压缩功能）
    - `calibre_extensions.winsapi` 缺失（Windows 特有 API）
    - 7z API 变更、WEBP 透明度、FTS 搜索等
    这些不影响主要功能。

---

## 11. 构建流程总览

```
1. 安装工具链 (VS, Python, Ruby, Node, WiX, Mesa, rapydscript-ng)
2. 克隆源码 (calibre + bypy)
3. 下载并解压依赖包 (windows-64.tar.xz)
4. 从官方 MSI 补充缺失 DLL
5. Bootstrap (py.exe setup.py bootstrap --ephemeral)
6. 修改 bypy 代码 (4 处修改，见 6.3 节)
7. 执行构建 (run_build.ps1)
8. 获取产物 (C:\r\sw64\dist\calibre-64bit-*.msi)
```
