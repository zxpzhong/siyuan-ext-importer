# 思源导入插件

这是一个将外部笔记数据导入思源的插件，当前支持：

- Notion HTML 导出 ZIP
- Wolai ZIP 导出（优先兼容“导出整个空间”）

## 用法

### Notion

#### 导出

![](docs/img/20240926110219.png)
![](docs/img/20240926113400.png)

### Wolai

- 在 Wolai 中导出 `.zip` 文件。
- 在插件中选择导出的 ZIP，插件会自动按后缀识别 ZIP 并尝试按 Wolai 格式导入。


#### Wolai 导出空间 ZIP 结构（优先兼容）

当 ZIP 中包含以下结构时，插件会按空间导出规则处理：

- 根目录索引：`xxx.md`
- 页面目录：`pages/`
- 资源目录：`images/`、`video/`、`file/`
