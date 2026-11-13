# forcetone · 晴耕雨读

个人技术博客，记录云原生、容器、数据库实践，也留一些日常随笔与剪藏。

- 在线站点：<https://pages.tiarea.com.cn>
- 仓库：<https://github.com/nemo1991/nemo1991.github.io>

## 技术栈

| 项 | 说明 |
| --- | --- |
| 静态站点 | Jekyll |
| 主题 | [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) `~> 7.3` |
| 界面语言 | `zh-CN`（使用 Chirpy 内置的中文语言包，无需自建 locales） |
| 评论 | utterances，`issue_term: pathname` |
| 其他 | Google Analytics、PWA 离线缓存、jekyll-archives、jekyll-admin |
| 部署 | GitHub Actions → GitHub Pages |

## 目录结构

```
_config.yml             站点配置：标题、语言、SEO、评论、归档与默认 permalink
_data/                  侧栏联系方式与文章底部分享平台
_plugins/               posts-lastmod-hook.rb：用 git log 自动补 last_modified_at
_posts/                 文章（31 篇）
_tabs/                  about / archives / categories / tags 四个页面
assets/img/             头像、favicon、文章配图
assets/file/            文章引用的配置文件（如 oracle19c.yaml）
assets/gemini/          Gemini Deep Research 生成的 3 个独立 HTML 页面
tools/run.sh            本地预览
tools/test.sh           生产构建 + htmlproofer 链接检查
index.html              首页（layout: home）
```

`assets/gemini/` 下的 3 个页面由《Gemini Deep Research》一文中的链接进入，没有挂到全局导航。

## 本地运行

需要 Ruby 3.x 与 Bundler（CI 使用 Ruby 3.3）。

```bash
bundle install
bash tools/run.sh                 # 绑定 127.0.0.1，带热重载
bash tools/run.sh -H 0.0.0.0      # 允许局域网访问
bash tools/run.sh -p              # JEKYLL_ENV=production
```

构建并检查站内链接：

```bash
bash tools/test.sh
```

## 写作与命名规范

### 文章文件名

统一格式 `YYYY-MM-DD-slug.md`，放在 `_posts/`。

| 规则 | 说明 |
| --- | --- |
| 日期前缀 | `YYYY-MM-DD`，用连字符且不可省略。部分文章没有 front matter `date`，Jekyll 完全依赖文件名推断日期 |
| slug 字符集 | 只允许小写字母、数字、连字符 `-` |
| 词间分隔 | **只用 `-`**；禁止下划线 `_`、空格、大写字母、中文 |
| 示例 | `2026-10-20-docker-volume-manage.md` ✅ / `2026-10-20-docker_volume_manage.md` ❌ |

**URL 由文件名决定，不由标题决定。** 站点配置为 `permalink: /posts/:title/`，其中 `:title` 取的是文件名里的 slug，不是 front matter 的 `title`。因此：

- 改文章标题不会改变 URL；
- **改文件名会改变 URL**，站外旧链接会 404，且 utterances 以 `pathname` 绑定评论，旧评论会留在旧路径对应的 issue 里；
- 确需改名又要保住链接时，先在该文 front matter 写显式 `permalink: /posts/<旧slug>/`，再改文件名（此时 URL 仍是旧样式）。

### 资源文件

- 文章配图放 `assets/img/posts/<日期-slug>/`，**目录名与文章文件名去掉 `.md` 后完全一致**；图片文件名同样小写 + 连字符，禁止空格（空格会被编码成 `%20`）。
- `assets/gemini/` 下的独立 HTML 页面、`assets/file/` 下的附件遵循同一规则：全小写、连字符分词、无空格。
- 正文引用站内资源统一使用相对路径（如 `../assets/img/posts/2025-08-13-oracle-on-k8s/image.png`、`../assets/gemini/ms-on-k8s-network-rule.html`）。该写法由 VS Code Markdown 粘贴插件生成，构建后输出为 `/../assets/...`，浏览器会归一化到站点根 `/assets/...`，在 VSCode 预览与发布后均可正常显示。
- 用 `<!--more-->` 控制首页摘要长度。

### front matter

- 分类一律用复数 `categories`，配合 `tags`、`toc`。
- 日期可省略（由文件名提供）；如需精确到时间，写 `date: YYYY-MM-DD HH:MM:SS +0800`，务必带时区。

## 部署

push 到 `main` 会触发 `.github/workflows/pages-deploy.yml`：Jekyll 生产构建 → 上传 artifact → `actions/deploy-pages` 发布。

自定义域名在仓库 Settings → Pages → Custom domain 配置，因此仓库里没有 `CNAME` 文件；根目录的 `.nojekyll` 用于避免 Pages 对已构建产物做二次 Jekyll 处理。`tools/` 与 `README.md` 已在 `_config.yml` 的 `exclude` 中排除，不会进入站点产物。

---

## 附：早期搭建笔记（原文保留）

> 以下是建站时的操作记录，仅作历史留存，内容可能已过时。

### step by step

1. create repo as username.github.io
   如果要创建用户或组织站点，则存储库必须命名为 `<user>.github.io` 或 `<organization>.github.io`。如果您的用户或组织名称包含大写字母，您必须小写字母
2. 发布源？
   从分支进行发布，关键就是选择分支与目录；或使用 github actions

### install jekyll

[install jekyll on mac](https://jekyllrb.com/docs/installation/macos/)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install chruby ruby-install
ruby-install ruby 3.4.1

echo "source $(brew --prefix)/opt/chruby/share/chruby/chruby.sh" >> ~/.zshrc
echo "source $(brew --prefix)/opt/chruby/share/chruby/auto.sh" >> ~/.zshrc
echo "chruby ruby-3.4.1" >> ~/.zshrc # run 'chruby' to see actual version

ruby -v
```

quick start: [quick start](https://jekyllrb.com/docs/)

```bash
gem install jekyll bundler
jekyll new myblog
cd myblog
bundle exec jekyll serve
```

切换 ruby 源：

```bash
gem sources --add https://gems.ruby-china.com/ --remove https://rubygems.org/
```

参考：<https://blog.csdn.net/weixin_41287260/article/details/102582315>

> 之前购买的域名解析 CNAME 删除后，依然会跳转到绑定的域名。绑定新域名后，删除浏览器缓存或使用隐私模式，好了。
