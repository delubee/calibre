# Calibre Windows 构建指南（纯 Windows 环境）

本文档描述如何在真实 Windows 机器上，从源码构建 calibre Windows 安装文件（MSI + 便携版）。

源码仓库：`https://github.com/delubee/calibre.git`

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

### 2.5 Mesa OpenGL

下载 `opengl32sw.dll`：
https://download.qt.io/development_releases/prebuilt/llvmpipe/windows/

放置到：

```
C:\mesa\64\opengl32sw.dll
```

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

> 注意：该文件较大（约 1-2GB），下载和解压需要时间。
> 解压后 `C:\r\sw64\sw` 下应包含 `bin`、`lib`、`include`、`qt`、`private` 等目录。

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
- 构建 GUI 资源
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
py.exe setup.py resources      :: 其他资源
py.exe setup.py cacerts        :: CA 证书
```

---

## 6. 构建安装文件

### 6.1 方式一：通过 bypy 构建（推荐）

克隆 bypy 仓库（与 calibre 源码同级）：

```cmd
cd C:\r
git clone https://github.com/kovidgoyal/bypy.git
```

设置 bypy 位置环境变量：

```cmd
set BYPY_LOCATION=C:\r\bypy
```

在 calibre 源码目录执行：

```cmd
cd C:\r\src
py.exe setup.py win64 --dont-sign --dont-shutdown
```

参数说明：
- `--dont-sign`：跳过代码签名（你没有签名证书）
- `--dont-shutdown`：构建完不关机

输出文件位于 `C:\r\src\dist\`：
- `calibre-64bit-<version>.msi` — MSI 安装包
- `calibre-portable-<version>.zip.lz` → 打包为便携版安装程序
- `calibre-portable-installer-<version>.exe` — 便携版安装程序

### 6.2 方式二：手动执行打包脚本

如果 bypy 方式遇到 VM 管理相关问题，可以直接在 Windows 上运行打包脚本。

需要确保 bypy 包可被 Python 导入：

```cmd
cd C:\r\bypy
py.exe -m pip install -e .
```

然后设置必要的环境变量并运行 `bypy/windows/__main__.py`：

```cmd
set SW=C:\r\sw64\sw
set SRC=C:\r\src
set CALIBRE_DIR=C:\r\src

cd C:\r\bypy
py.exe -c "import runpy; runpy.run_path('bypy/windows/__main__.py', run_name='__main__')"
```

> 注意：此方式需要 bypy 的 `constants.py` 正确识别环境，
> 可能需要根据实际报错调整环境变量。

---

## 7. 构建产物

构建成功后，在 `dist` 目录下会生成：

| 文件 | 说明 |
|------|------|
| `calibre-64bit-9.12.0.msi` | Windows 64-bit MSI 安装包 |
| `calibre-portable-installer-9.12.0.exe` | 便携版安装程序 |

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

```cmd
curl -L -C - -o windows-64.tar.xz https://download.calibre-ebook.com/ci/calibre7/windows-64.tar.xz
```

### 8.4 WiX 构建失败

确认 WiX 已正确安装：

```cmd
%USERPROFILE%\.dotnet\tools\wix.exe --version
```

首次构建时 WiX 会自动下载扩展（WixToolset.Util.wixext、WixToolset.UI.wixext），
需要网络连接。

### 8.5 opengl32sw.dll 未找到

```
Mesa DLLs (opengl32sw.dll) not found
```

确认文件存在：

```cmd
dir C:\mesa\64\opengl32sw.dll
```

### 8.6 内存不足（编译 Qt WebEngine 相关）

如果选择从源码编译依赖（而非使用预编译包），Qt WebEngine 需要至少 8GB 内存。
使用预编译依赖包可完全避免此问题。

---

## 9. 完整构建脚本（一键执行）

将以下内容保存为 `C:\r\build.bat`：

```bat
@echo off
setlocal enabledelayedexpansion

echo ============================================
echo   Calibre Windows Build Script
echo ============================================

:: === 配置区 ===
set ROOT=C:\r
set SRC=%ROOT%\src
set SW=%ROOT%\sw64\sw
set BUILD_DIR=%ROOT%\build

:: === 环境变量 ===
set QMAKE=%SW%\qt\bin\qmake.exe
set CALIBRE_QT_PREFIX=%SW%\qt
set OPENSSL_MODULES=%SW%\lib\ossl-modules
set PIPER_TTS_DIR=%SW%\piper
set CALIBRE_ESPEAK_DATA_DIR=%SW%\share\espeak-ng-data
set MESA=C:\mesa
set BYPY_LOCATION=%ROOT%\bypy

:: === PATH 设置 ===
set PATH=%SW%\private\python\bin;%PATH%
set PATH=%SW%\private\python\Lib\site-packages\pywin32_system32;%PATH%
set PATH=%SW%\bin;%PATH%
set PATH=%SW%\qt\bin;%PATH%

:: === 步骤 1：检查依赖 ===
echo [1/4] Checking dependencies...
if not exist "%SW%\qt\bin\qmake.exe" (
    echo ERROR: Dependencies not found at %SW%
    echo Please download and extract windows-64.tar.xz first.
    exit /b 1
)
if not exist "%SRC%\setup.py" (
    echo ERROR: Calibre source not found at %SRC%
    exit /b 1
)

:: === 步骤 2：Bootstrap ===
echo [2/4] Bootstrapping calibre source...
cd /d %SRC%
py.exe setup.py bootstrap --ephemeral
if %ERRORLEVEL% neq 0 (
    echo ERROR: Bootstrap failed
    exit /b 1
)

:: === 步骤 3：构建安装文件 ===
echo [3/4] Building Windows installer...
cd /d %SRC%
py.exe setup.py win64 --dont-sign --dont-shutdown
if %ERRORLEVEL% neq 0 (
    echo ERROR: Build failed
    exit /b 1
)

:: === 步骤 4：完成 ===
echo [4/4] Build complete!
echo.
echo Output files:
dir /b %SRC%\dist\*.msi 2>nul
dir /b %SRC%\dist\*.exe 2>nul
echo.
echo Done!

endlocal
```

---

## 10. 注意事项

1. **预编译依赖包版本匹配**：CI 依赖包是为 calibre 7.x/9.x 系列构建的，
   如果你的源码版本差异较大，可能需要自行编译依赖。

2. **代码签名**：使用 `--dont-sign` 跳过签名。未签名的安装程序在
   Windows SmartScreen 中会显示警告，但功能完全正常。

3. **构建时间**：使用预编译依赖包时，整个构建过程约 30-60 分钟
   （主要耗时在 bootstrap 编译 C 扩展和 WiX 打包）。

4. **磁盘空间**：构建过程中会产生大量临时文件，确保有足够空间。
   构建完成后可清理 `C:\r\build` 目录。

5. **bypy 仓库**：bypy 是构建编排工具，必须与 calibre 源码配合使用。
   如果 `setup.py win64` 报错找不到 bypy，设置 `BYPY_LOCATION` 环境变量。
