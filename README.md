# calibre

<img align="left" src="https://raw.githubusercontent.com/kovidgoyal/calibre/master/resources/images/lt.png" height="200" width="200"/>

calibre is an e-book manager. It can view, convert, edit and catalog e-books 
in all of the major e-book formats. It can also talk to e-book reader 
devices. It can go out to the internet and fetch metadata for your books. 
It can download newspapers and convert them into e-books for convenient 
reading. It is cross platform, running on Linux, Windows and macOS.

For more information, see the [calibre About page](https://calibre-ebook.com/about).

[![Build Status](https://github.com/kovidgoyal/calibre/workflows/CI/badge.svg)](https://github.com/kovidgoyal/calibre/actions?query=workflow%3ACI)

## Screenshots  

[Screenshots page](https://calibre-ebook.com/demo)

## Usage

See the [User Manual](https://manual.calibre-ebook.com).

## Development

[Setting up a development environment for calibre](https://manual.calibre-ebook.com/develop.html).

A [tarball of the source code](https://calibre-ebook.com/dist/src) for the 
current calibre release.

## Bugs

Bug reports and feature requests should be made in the calibre bug tracker at [Launchpad](https://bugs.launchpad.net/calibre).
GitHub is only used for code hosting and pull requests.

## Support calibre

calibre is a result of the efforts of many volunteers from all over the world.
If you find it useful, please consider contributing to support its development.
[Donate to support calibre development](https://calibre-ebook.com/donate).

## Building calibre binaries

See [Build instructions](bypy/README.rst) for instructions on how to build the
calibre binaries and installers for all the platforms calibre supports.

**[为什么要在 Windows 上编译 calibre：一次 AI 驱动的完整踩坑实录](build_calibre_windows.md)**
—— 在纯 Windows 环境下从源码编译生成 MSI 安装包的完整记录（中文）。

**[Calibre Windows 构建指南](windows_build.md)**
—— 纯 Windows 环境下从源码构建 MSI 安装包的详细步骤文档（中文）。

## 本仓库的本地修改

### 保存电子书时使用完整中文文件名

calibre 默认将电子书保存到本地时，会将中文文件名转写为拼音（通过 `ascii_filename` 函数调用 ICU 转写）。
本仓库将文件名生成逻辑改为使用 `sanitize_file_name`，保留原始中文字符，仅替换 Windows 不支持的特殊字符（`\ | ? * < " : > + /` 及控制字符）。

涉及文件：
- `src/calibre/db/backend.py` — `construct_path_name()` / `construct_file_name()`
- `src/calibre/library/database2.py` — 同上（旧版 API）

效果示例：`刘慈欣/三体 (1)/三体 - 刘慈欣.epub`

## calibre package versions in various repositories

[![Packaging Status](https://repology.org/badge/vertical-allrepos/calibre.svg?columns=3&header=calibre)](https://repology.org/project/calibre/versions)
