# SiYuan Import Plugin

[中文版](./README_zh_CN.md)

This plugin imports external notes into SiYuan. Current supported sources:

- Notion HTML export ZIP
- Wolai ZIP export (space export preferred)

## Usage

### Notion

#### Export

![](docs/img/20240926110219.png)
![](docs/img/20240926113400.png)

### Wolai

- Export your workspace/pages as a `.zip` package from Wolai.
- Select the ZIP in this plugin. The importer auto-detects ZIP input by extension and imports with Wolai rules when it is not a Notion HTML export.


#### Wolai space export ZIP structure (preferred compatibility)

When ZIP content follows Wolai full-space export layout, importer prioritizes this structure:

- root index markdown: `xxx.md`
- pages directory: `pages/`
- assets directories: `images/`, `video/`, `file/`
