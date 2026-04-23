
## 📌 项目初始化与本地开发指南

### 1. 初始化子模块（Submodule）
本项目使用 Git Submodule 管理主题依赖。克隆仓库后，请务必执行以下命令拉取主题代码：

```bash
git submodule update --init
```

> ✅ 此命令会自动下载并初始化 `themes/` 目录下的主题子模块。


### 2. 开发文章存放路径
所有开发相关的文章请统一放置在以下目录：

```
staticblog/content/dev/
```

Hugo 会自动将该目录下的 Markdown 文件渲染为网站内容。


### 3. 安装 Hugo
本项目需使用 **Hugo Extended** 版本（支持 SCSS、PostCSS 等特性）。

- **当前推荐版本**：  
  `hugo_extended_withdeploy_0.160.1_windows-amd64`

- **下载地址**：  
  [https://github.com/gohugoio/hugo/releases](https://github.com/gohugoio/hugo/releases)

> 💡 请确保 `hugo` 命令已加入系统环境变量，以便在终端中直接调用。


### 4. 本地预览网站
在项目根目录下运行以下命令启动本地开发服务器：

```bash
hugo server
```

- 默认访问地址：[http://localhost:1313](http://localhost:1313)
- 支持热重载（修改文件后浏览器自动刷新）


✅ 完成以上步骤后，即可开始撰写和预览你的开发博客,最后提交PR即可!