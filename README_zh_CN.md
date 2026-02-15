# 思源导入插件

这是一个将 Wolai / Markdown ZIP 导入思源的插件。

## 用法

- 方式 1：在插件中选择本地 `.zip` 文件导入。
- 方式 2：填写 GitHub 压缩包链接（例如 `main.zip`），插件会自动下载、解压并导入。

### GitHub ZIP 导入说明

- 支持 `https://github.com/<owner>/<repo>/archive/refs/heads/main.zip` 这类链接。
- 会按压缩包内目录层级保存附件文件路径。
- Markdown 文件会按目录层级创建思源文档。
