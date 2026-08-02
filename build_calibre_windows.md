# 在纯 Windows 环境下从源码编译 calibre：一次 AI 驱动的完整踩坑实录

> 本文记录了在 Windows 11 上从 calibre 源码（v9.12.0）编译生成 MSI 安装包的完整过程。
> 没有 Linux、没有 Docker、没有虚拟机——只有一台真实的 Windows 机器、一个 AI 编程助手，和一个晚上。

---

## 为什么要在 Windows 上编译 calibre？

[calibre](https://calibre-ebook.com/) 是一款功能强大的开源电子书管理工具，支持格式转换、元数据编辑、设备同步等。它的官方构建流程基于 Linux 虚拟机 + bypy 编排系统，Windows 构建被设计为"在 Linux 主机上通过 SSH 远程触发 Windows VM 内的编译"。

这意味着：**官方从未提供过"在纯 Windows 环境下直接构建"的文档。**

### 真正的动机：中文文件名之痛

作为一个曾经重度 calibre 用户，我有一个长期困扰：**calibre 保存到本地的电子书文件名全部是拼音**。比如一本叫《三体》的书，导出后文件名是 `san-ti.epub`。对于中文用户来说，这简直是灾难——你根本无法从文件名辨认出这是什么书。当然这个问题很多年前通过修改源码，每次让calibre通过源码启动，其实已经解决了保存为中文文件名的问题，但是不完美。

有人曾向 calibre 作者 Kovid Goyal 提交过修改建议，希望支持保留中文文件名。但作者的回复很明确：**无意修改**。calibre 的文件名策略（使用作者名 + 书名的拼音/音译）是有意为之的设计，目的是避免跨平台文件系统兼容性问题。

既然上游不会改，那就自己改。而要改源码，首先得能编译。这就是本文的起点。

### 一个大胆的决定：让 AI 来编译

面对一个从未有人走通的构建路径，我做了个大胆的决定：**让 AI 编程助手（Qoder）全程自动化完成编译构建**。

我的角色很简单：
- 提供环境和目标
- 在 AI 遇到需要下载大文件时，手动下载并告知文件位置
- 然后……去睡觉

结果第二天早上醒来，`C:\r\sw64\dist\calibre-64bit-9.12.0.msi`（195 MB）已经安静地躺在那里了。

整个过程，AI 独立解决了十余个构建错误，修改了 4 处 bypy 源码，从官方 MSI 中提取了 50+ 个缺失的 DLL。我唯一的手动操作，是下载了一个 274MB 的依赖包和一个 212MB 的官方 MSI——因为 AI 的网络下载遇到了问题。

这篇文章，就是那个晚上发生的一切的"验尸报告"。

---

## 环境概览

| 项目 | 版本/规格 |
|------|-----------|
| 操作系统 | Windows 11 21H2 (x64) |
| 编译器 | Visual Studio 2026 (MSVC 14.51) |
| 内嵌 Python | 3.14（依赖包自带） |
| 系统 Python | 3.13（用于 bootstrap） |
| calibre 版本 | 9.12.0 |
| 最终产物 | `calibre-64bit-9.12.0.msi`（195 MB） |

---

## 一、工具链准备

calibre 的构建依赖链相当庞大。以下是我实际安装的全部工具：

### 1.1 Visual Studio 2026

安装 Community 版本，勾选：
- MSVC v143+ x64/x86 构建工具
- Windows 11 SDK
- C++ ATL、CMake 工具
- .NET SDK（WiX 需要）
- Git for Windows

关键：确保 `cl.exe` 和 `link.exe` 在 PATH 中可用。

### 1.2 其他工具

```
Python 3.13+（系统级，用于 bootstrap）
Ruby 3.4 x64（calibre 的模板引擎需要）
Node.js LTS（RapydScript 编译器）
Strawberry Perl（部分依赖的构建脚本）
WiX Toolset v7（生成 MSI）
Mesa OpenGL（opengl32sw.dll，Qt 渲染需要）
```

### 1.3 WiX 的隐藏陷阱

安装 WiX 后，大多数人会直接验证版本号然后继续。**别急。** WiX v7 引入了一个"开源维护费 EULA"机制，如果不提前接受，构建到 MSI 打包阶段（可能已经过了 30 分钟）才会报错：

```
error WIX7015: You must accept the Open Source Maintenance Fee (OSMF) EULA to use WiX Toolset v7
```

正确做法是安装后立即执行：

```cmd
wix eula accept wix7
wix extension add -g WixToolset.Util.wixext
wix extension add -g WixToolset.UI.wixext
```

我在构建快完成时才发现这个问题，白白浪费了一轮构建时间。

### 1.4 RapydScript-NG：一个容易忽视的依赖

calibre 的 Web 组件（阅读器、编辑器 UI）使用 RapydScript 编写，构建时需要编译为 JavaScript。calibre 内置了一个基于 Qt WebEngine 的编译器，但在无头/远程桌面环境下，它几乎必然超时：

```
TimeoutError: Creating RapydScript compiler took too long
```

解决方案是安装外部编译器：

```cmd
npm install -g rapydscript-ng
```

安装后 calibre 的构建系统会自动检测并使用它，完全绕过 WebEngine。

---

## 二、源码与依赖

### 2.1 目录结构

我选择了简短的根路径 `C:\r`（避免路径过长问题）：

```
C:\r\
├── src\          ← calibre 源码
├── bypy\         ← 构建编排工具
├── sw64\sw\      ← 预编译依赖（2.4GB）
└── run_build.ps1 ← 构建脚本
```

### 2.2 克隆源码

```cmd
cd C:\r
git clone https://github.com/kovidgoyal/calibre.git src
git clone https://github.com/kovidgoyal/bypy.git
```

### 2.3 预编译依赖包：最大的坑

calibre 官方 CI 提供了预编译的 Windows 依赖包，包含 Qt 6、ICU、OpenSSL、Python、ffmpeg 等全部 C/C++ 依赖：

```
https://download.calibre-ebook.com/ci/calibre7/windows-64.tar.xz
```

文件约 274MB（压缩），解压后 2.4GB。下载后解压到 `C:\r\sw64\sw`。

**然而，这个包是不完整的。**

当我满怀信心地启动构建，编译 C 扩展一切顺利，冻结 Python 字节码也顺利完成，直到运行验证测试时：

```
ImportError: DLL load failed while importing icu: 找不到指定的模块。
```

ICU——Unicode 国际化组件——calibre 最核心的依赖之一，它的 DLL 不在依赖包里。

经过逐一排查，我发现缺失的文件远不止 ICU：

| 缺失文件 | 用途 |
|----------|------|
| `icudt78.dll` / `icuin78.dll` / `icuuc78.dll` / `icuio78.dll` / `icutu78.dll` | Unicode 处理 |
| `espeak-ng.dll` | TTS 语音合成 |
| `freetype.dll` | 字体渲染 |
| `jpeg8.dll` | JPEG 处理 |
| `lcms2-2.dll` | 色彩管理 |
| `brotli*.dll`（3个） | Brotli 压缩 |
| 39 个 SQLite 扩展 DLL | FTS 全文搜索等 |
| `jpegtran-calibre.exe` / `cwebp-calibre.exe` 等 | 图像优化工具 |

### 2.4 解决方案：从官方 MSI 中"借"DLL

既然官方发布的安装包能正常运行，那它里面一定有这些 DLL。思路很简单：

```powershell
# 下载官方 MSI（212MB）
Invoke-WebRequest -Uri "https://download.calibre-ebook.com/9.12.0/calibre-64bit-9.12.0.msi" `
    -OutFile "C:\r\calibre-official.msi"

# 管理安装模式解压（不需要真正安装）
Start-Process msiexec -ArgumentList '/a', 'C:\r\calibre-official.msi', '/qn', `
    'TARGETDIR=C:\r\calibre-extracted' -Wait

# 从解压目录复制缺失文件
$src = "C:\r\calibre-extracted\PFiles64\Calibre2\app\bin"
Copy-Item "$src\icu*.dll" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\espeak-ng.dll" "C:\r\sw64\sw\bin\" -Force
Copy-Item "$src\freetype.dll" "C:\r\sw64\sw\bin\" -Force
# ... 其余文件同理
```

> **踩坑提醒**：不要复制 `libffi-8.dll` 和 `sqlite3.dll`。它们已存在于依赖包的 Python DLLs 目录中，重复复制会导致后续构建时的文件锁定冲突。

---

## 三、Bootstrap：编译 C 扩展

```cmd
cd C:\r\src
py.exe setup.py bootstrap --ephemeral
```

这一步会：
1. 编译约 40 个 C/C++ 扩展模块（使用 MSVC）
2. 生成 ISO 639/3166 语言代码数据
3. 编译翻译文件
4. 构建 GUI 资源（含 RapydScript → JavaScript）
5. 下载 CA 证书

### 3.1 GBK 编码炸弹

在中文 Windows 上，第一个遇到的错误大概率是：

```
UnicodeEncodeError: 'gbk' codec can't encode character '\ufffd' in position 42
```

原因：MSVC 编译器输出包含非 UTF-8 字节（如路径中的特殊字符），Python 将其解码为 `\ufffd`（替换字符），然后尝试用 GBK 编码输出到控制台时失败。

修复：

```powershell
$env:PYTHONIOENCODING = 'utf-8'
$env:PYTHONUTF8 = '1'
```

这两行必须放在所有 Python 调用之前。

### 3.2 RapydScript 编译

如果没装外部 rapydscript-ng，`setup.py resources` 会尝试用内嵌的 WebEngine 编译器，然后超时。安装外部编译器后问题解决。

### 3.3 completed.json 缺失

```
FileNotFoundError: [Errno 2] No such file or directory: 'manual/locale/completed.json'
```

创建一个空的 JSON 文件即可：

```powershell
[System.IO.File]::WriteAllText('C:\r\src\manual\locale\completed.json', '{}')
```

注意必须是无 BOM 的 UTF-8。PowerShell 的 `Set-Content -Encoding UTF8` 会加 BOM，导致 JSON 解析失败。

---

## 四、构建 MSI：绕开 VM 管理层

### 4.1 `setup.py win64` 为什么不能用

按照 calibre 源码中的注释，Windows 构建命令是：

```cmd
py.exe setup.py win64 --dont-sign --dont-shutdown
```

但在纯 Windows 环境下执行，立即报错：

```
ValueError: Not a valid SSH URL: C:\r\src\bypy\b\windows\vm
```

原因：calibre 的构建架构是为 Linux 主机设计的。`setup.py win64` 会调用 bypy 的 VM 管理层，试图通过 SSH 连接到一台 Windows 虚拟机。在纯 Windows 环境下，这个 SSH URL 解析直接失败。

### 4.2 正确方式：直接调用 bypy

绕过 VM 管理层，直接调用 bypy 的 `program` 子命令：

```powershell
& 'C:\r\sw64\sw\private\python\python.exe' 'C:\r\bypy' `
    'BYPY_ROOT=C:\r' 'BUILD_ARCH=64' 'BYPY_ARCH=windows-64' `
    'PERL=perl' 'RUBY=C:\Ruby34-x64\bin\ruby.exe' `
    'MESA=C:\mesa' 'NODEJS=node' `
    program --skip-tests
```

bypy 接受 `KEY=VALUE` 格式的参数作为环境变量注入，`program` 是构建子命令，执行完整的：

```
init_env → build_c_extensions → freeze → build_launchers → 
embed_manifests → copy_crt_and_d3d → create_installer → build_portable
```

### 4.3 必须修改的 bypy 代码

bypy 是为 Linux + Cygwin 环境设计的，在纯 Windows 下有 4 处必须修改：

**修改 1：跳过依赖重新安装**

bypy 的 `install_packages()` 会试图清除已解压的依赖目录并重新安装。但 DLL 文件可能被系统锁定，导致 `PermissionError`。

```python
# C:\r\bypy\bypy\deps.py
def install_packages(which_deps, dest_dir=PREFIX):
    if os.path.isdir(os.path.join(dest_dir, 'bin')) and os.listdir(os.path.join(dest_dir, 'bin')):
        print(f'Dependencies already present in {dest_dir}, skipping install_packages')
        return
    # ... 原始代码 ...
```

**修改 2：禁用 run_shell()**

构建失败时，bypy 会尝试启动交互式 shell 供开发者调试。它硬编码了 `C:/cygwin64/bin/zsh`——一个在纯 Windows 环境下不存在的路径。

```python
# C:\r\bypy\bypy\main.py 和 C:\r\src\bypy\init_env.py
# 将所有 run_shell() 替换为 pass
```

**修改 3：跳过缺失的可选工具**

`freeze()` 函数会复制 `pdftohtml.exe`、`pdfinfo.exe` 等工具。如果依赖包中缺少这些文件，原代码直接抛异常。改为存在才复制：

```python
for x in ('pdftohtml', 'pdfinfo', 'pdftoppm', 'pdftotext', ...):
    exe_path = os.path.join(bindir, x + '.exe')
    if os.path.exists(exe_path):
        copybin(exe_path)
    else:
        print(f'WARNING: skipping missing optional binary: {x}.exe')
```

**修改 4：避免 DLL 重复复制**

某些 DLL（如 `libffi-8.dll`）同时存在于多个源目录中，`copybin()` 会尝试复制两次，第二次因文件已存在且被占用而报权限错误：

```python
def copybin(x, dest=env.dll_dir):
    dst = os.path.join(dest, os.path.basename(x)) if os.path.isdir(dest) else dest
    if os.path.exists(dst):
        return  # 已复制，跳过
    shutil.copy2(x, dest)
```

---

## 五、构建过程中的其他"惊喜"

### 5.1 Windows Defender 锁定 DLL

bypy 使用 `C:\t\t` 作为临时构建目录（硬编码在 `bypy/bypy/constants.py` 中）。构建失败后重试时：

```
PermissionError: [WinError 5] 拒绝访问 'C:\\t\\t\\build-xxx\\winfrozen\\app\\bin\\amatch.dll'
```

Windows Defender 实时保护扫描了构建产物中的 DLL，导致文件被锁定无法删除。

解决：重命名旧目录（而非删除），创建新目录：

```powershell
Rename-Item "C:\t\t" "C:\t\t_old"
New-Item -ItemType Directory -Path "C:\t\t" -Force
```

**长期方案**：将 `C:\t\t` 和 `C:\r` 加入 Windows Defender 排除列表。

### 5.2 构建测试失败

使用 `--skip-tests` 跳过的测试包括：

- `pyzstd` 模块缺失（Zstandard 压缩，非核心功能）
- `calibre_extensions.winsapi` 缺失（Windows 特有 API 绑定）
- 7z API 变更导致的兼容性测试失败
- WEBP 图像透明度处理测试
- FTS 全文搜索测试

这些都不影响 calibre 的核心功能（书库管理、格式转换、阅读器）。

### 5.3 便携版签名

MSI 生成成功后，bypy 继续尝试构建便携版。便携版需要代码签名：

```
KeyError: 'SIGN_SERVER_PORT'
```

这需要一台运行签名服务的服务器。对于个人构建，MSI 已经足够。便携版可以忽略。

---

## 六、最终构建脚本

经过反复调试，最终稳定可用的构建脚本如下：

```powershell
# C:\r\run_build.ps1
# Calibre Windows 构建脚本

# 编码设置（中文 Windows 必须）
$env:PYTHONIOENCODING = 'utf-8'
$env:PYTHONUTF8 = '1'

# 配置
$ROOT = 'C:\r'
$SRC = "$ROOT\src"
$SW = "$ROOT\sw64\sw"
$PYTHON = "$SW\private\python\python.exe"

# 环境变量
$env:BUILD_ARCH = '64'
$env:BYPY_ROOT = $ROOT

# PATH（仅保留必要路径）
$env:PATH = "$SW\private\python;" +
    "$SW\private\python\Lib\site-packages\pywin32_system32;" +
    "$SW\bin;$SW\qt\bin;" +
    "C:\Ruby34-x64\bin;C:\Program Files\nodejs;" +
    "C:\Windows\System32;C:\Windows"

# 前置检查
if (-not (Test-Path "$SW\qt\bin\qmake.exe")) {
    Write-Error "依赖包未就绪，请先解压 windows-64.tar.xz"
    exit 1
}

# 构建
Set-Location $SRC
& $PYTHON "$ROOT\bypy" "BYPY_ROOT=$ROOT" 'BUILD_ARCH=64' `
    'BYPY_ARCH=windows-64' 'PERL=perl' `
    "RUBY=C:\Ruby34-x64\bin\ruby.exe" 'MESA=C:\mesa' `
    'NODEJS=node' program --skip-tests

if ($LASTEXITCODE -eq 0) {
    Write-Host "构建成功！" -ForegroundColor Green
    Get-ChildItem "$ROOT\sw64\dist\*.msi"
} else {
    Write-Host "构建失败 (exit code: $LASTEXITCODE)" -ForegroundColor Red
}
```

执行：

```cmd
powershell -ExecutionPolicy Bypass -File "C:\r\run_build.ps1"
```

---

## 七、构建流程全景图

```
┌─────────────────────────────────────────────────────────┐
│                    工具链安装                             │
│  VS 2026 / Python / Ruby / Node / WiX / Mesa / RS-NG   │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    源码准备                               │
│  git clone calibre → C:\r\src                           │
│  git clone bypy → C:\r\bypy                             │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    依赖准备                               │
│  下载 windows-64.tar.xz → 解压到 C:\r\sw64\sw          │
│  从官方 MSI 提取缺失 DLL → 补充到 sw64\sw\bin          │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    Bootstrap                             │
│  py.exe setup.py bootstrap --ephemeral                  │
│  （编译 C 扩展 / 翻译 / GUI 资源 / RapydScript）        │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    修改 bypy                             │
│  4 处代码修改（见第四节）                                 │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    执行构建                               │
│  run_build.ps1 → bypy program --skip-tests              │
│  init_env → freeze → launchers → WiX → MSI             │
└────────────────────────┬────────────────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    产物                                   │
│  C:\r\sw64\dist\calibre-64bit-9.12.0.msi (195 MB)      │
└─────────────────────────────────────────────────────────┘
```

---

## 八、踩坑清单速查

| # | 错误信息 | 原因 | 解决 |
|---|----------|------|------|
| 1 | `UnicodeEncodeError: 'gbk'` | 中文 Windows 控制台编码 | 设置 `PYTHONIOENCODING=utf-8` |
| 2 | `ImportError: DLL load failed (icu)` | 依赖包缺少 ICU DLL | 从官方 MSI 提取 |
| 3 | `error WIX7015` | WiX v7 EULA 未接受 | `wix eula accept wix7` |
| 4 | `PermissionError: 拒绝访问` | Defender 锁定 DLL | 重命名 `C:\t\t`，加排除 |
| 5 | `ValueError: Not a valid SSH URL` | `setup.py win64` 需要 VM | 直接调用 bypy `program` |
| 6 | `FileNotFoundError: zsh` | bypy 尝试启动 Cygwin shell | 禁用 `run_shell()` |
| 7 | `TimeoutError: RapydScript` | WebEngine 无头超时 | 安装外部 `rapydscript-ng` |
| 8 | `KeyError: 'SIGN_SERVER_PORT'` | 便携版需要签名服务器 | 忽略，MSI 已生成 |
| 9 | `PermissionError: libffi-8.dll` | DLL 重复复制 | `copybin()` 加存在检查 |
| 10 | `FileNotFoundError: completed.json` | 翻译元数据缺失 | 创建空 `{}` 文件 |

---

## 九、经验与思考

### 9.1 calibre 的构建架构

calibre 的构建系统（bypy）是一个相当复杂的编排工具，它的设计假设是：

- **Linux 主机**作为控制中心
- **Windows/macOS 虚拟机**作为编译目标
- **SSH** 作为通信通道
- **Cygwin/zsh** 作为 Windows 上的 shell

这种架构对官方 CI 很合适，但对想在本地 Windows 上直接构建的开发者极不友好。bypy 中大量硬编码的路径（`C:/cygwin64/bin/zsh`、`C:\t\t`）和隐式假设（依赖包完整、签名服务器可用）都需要逐一绕过。

### 9.2 依赖包的不完整性

`windows-64.tar.xz` 是 CI 流水线的中间产物，它假设后续步骤会补充缺失文件。但对于脱离 CI 环境的独立构建者来说，这个"后续步骤"并不存在。从官方 MSI 中提取 DLL 是一个务实的解决方案——毕竟官方产物就是最权威的"正确文件集合"。

### 9.3 中文 Windows 的特殊性

GBK 编码问题是中文 Windows 开发者的"老朋友"了。任何涉及子进程输出捕获的 Python 构建系统，在中文 Windows 上都可能遇到这个问题。`PYTHONUTF8=1` 是 Python 3.7+ 提供的"核选项"，强制所有 I/O 使用 UTF-8，一劳永逸。

### 9.4 构建耗时参考

| 阶段 | 耗时（8 核 CPU） |
|------|------------------|
| 依赖包下载 + 解压 | 15-30 分钟 |
| Bootstrap（C 扩展编译） | 10-15 分钟 |
| Freeze + 打包 | 5-10 分钟 |
| WiX 生成 MSI | 3-5 分钟 |
| **总计** | **约 40-60 分钟** |

---

## 十、写在最后

### 编译成功了，然后呢？

安装自己编译的 MSI，打开 calibre——界面是英文的，语言选择里也只有 English，没有中文选项。

排查后发现根因：bootstrap 阶段翻译仓库（`kovidgoyal/calibre-translations`）克隆失败（网络问题静默跳过），导致 `locales.zip` 和 `stats.calibre_msgpack` 从未生成。calibre 通过 `stats.calibre_msgpack` 判断可用语言列表，文件不存在就只显示英文。

修复很简单：

```powershell
cd C:\r\src
git clone --depth=1 https://github.com/kovidgoyal/calibre-translations.git translations
py.exe setup.py translations
```

编译完成后 `resources/localization/` 下出现 17MB 的 `locales.zip`，重新构建 MSI，安装后中文语言选项正常出现。这个坑提醒我：bootstrap 后一定要检查 `translations` 目录是否存在。

### 下一步：解决中文文件名

编译跑通只是第一步。真正的目标是修改 calibre 的文件名生成逻辑，让导出的电子书保留中文文件名。这涉及到 `calibre/utils/filenames.py` 中的 `ascii_filename()` 函数——它会将所有非 ASCII 字符转写为拼音。我的计划是增加一个选项，允许用户选择保留原始 Unicode 文件名。

现在编译环境已经就绪，后续改一行代码、重新打包，只需要再跑一次 `run_build.ps1`，40 分钟后就能拿到新的安装包。这正是本地编译的价值。

### 关于 AI 辅助构建

这次经历让我对 AI 辅助开发有了新的认识。calibre 的构建系统复杂且文档稀缺，错误信息往往指向不相关的方向。AI 的优势在于：

- **不会疲倦**：凌晨三点遇到 `PermissionError` 依然能冷静分析
- **知识面广**：同时理解 MSVC、WiX、Python 打包、Qt 构建系统
- **不怕试错**：一个方案不行立刻换下一个，没有心理负担

而人类的不可替代性在于：知道"为什么要做这件事"，以及在 AI 卡住时提供那一个关键的手动操作。

### 致谢与期望

在纯 Windows 环境下编译 calibre 并非不可能，但确实需要绕过许多为 CI 环境设计的假设。整个过程最大的挑战不是技术难度，而是**信息的缺失**——官方文档假设你使用 Linux 主机，bypy 代码假设你有 Cygwin，依赖包假设你在 CI 流水线中。

希望这篇文章能为后来者节省一些时间。如果你也成功在 Windows 上编译了 calibre，或者对中文文件名问题有想法，欢迎交流。

---

## 附录：完整文件清单

构建完成后的关键文件位置：

```
C:\r\
├── src\                          ← calibre 源码
├── bypy\                         ← 构建编排（已修改）
├── sw64\
│   ├── sw\                       ← 依赖库（含补充的 DLL）
│   │   ├── bin\                  ← 所有运行时 DLL + 工具
│   │   ├── lib\                  ← 链接库
│   │   ├── qt\                   ← Qt 6
│   │   └── private\python\       ← 内嵌 Python 3.14
│   └── dist\
│       └── calibre-64bit-9.12.0.msi  ← 最终产物
├── run_build.ps1                 ← 构建脚本
├── calibre-official.msi          ← 官方 MSI（用于提取 DLL）
└── calibre-extracted\            ← 官方 MSI 解压目录
```

---

*本文基于 calibre 9.12.0 / bypy (2025-07) 版本撰写。后续版本可能有变化，但核心流程和坑点大概率相同。*
