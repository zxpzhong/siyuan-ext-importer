# SiYuan Import Plugin

[中文版](./README_zh_CN.md)

This plugin imports Wolai / Markdown ZIP content into SiYuan.

## Usage

- Option 1: choose a local `.zip` file.
- Option 2: provide a GitHub ZIP URL (for example `main.zip`) and the plugin will download and import it.

### GitHub ZIP import notes

- Supports links like `https://github.com/<owner>/<repo>/archive/refs/heads/main.zip`.
- Attachments are stored with directory hierarchy from the archive.
- Markdown files are imported into SiYuan documents preserving folder hierarchy.
