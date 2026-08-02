基于 calibre 9.12.0 源码编译，包含以下修改：

- 保存电子书到本地时保留完整中文文件名（不再转拼音）
- 仅替换 Windows 不支持的特殊字符（\ | ? * < " : > + /）
- 包含完整中文语言支持

修改文件：src/calibre/db/backend.py, src/calibre/library/database2.py

效果示例：刘慈欣/三体 (1)/三体 - 刘慈欣.epub
