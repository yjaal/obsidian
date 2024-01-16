```toc

```
## 入门配置

这里先创建一个`obsidian`的`github`仓库，然后拉去到本地，使用`obsidian`打开即可，可以使用`git`进行同步。

至于手机端，可以使用 `MGit` 同步 `github` 仓库，然后在手机端就可以同步浏览了。

`obsidian` 中添加图片使用的是 `wiki` 方式（![[img/稳定匹配001.png]]），如果想用标准的 `markdown` 方式可以在设置中进行设置。而 `github` 上面无法识别 `wiki` 方式，同时也无法识别数学表达式。不推荐此方式，可以直接用相对路径方式。

`dynamic table of contents` 插件可以动态生成目录，参考文章开头


## ios 使用

首先我购买的是天路云 [vpn](http://91tianlu.click/index.php)，可以同时在三台设备上面使用。而大家熟知的小火箭刚好又被墙了，所以需要找一个替代。而天路云太贴心了，可以使用[教程](http://91tianlu.fashion/knowledgebase.php?action=displayarticle&id=2)进行操作，使用应用 spectre 即可。

然后还需要一个同步代码的 app（working copy），将自己的笔记目录同步下来，然后参考[文章](https://forum.obsidian.md/t/mobile-setting-up-ios-git-based-syncing-with-mobile-app-using-working-copy/16499) j 逆行设置，其实就是首先在 obsidian 创建一个空的笔记目录，然后在 working copy 上面点击分享，分享的时候注意选择"Setup Folder Sync"即可。

感觉 obsidian 白板配合 ipad 会很爽。


## 插件使用

### 手动安装

在相关文档目录的 `.obsidian/plugins` 目录中保存相关插件即可，比如在此目录新建一个目录 `obsidian-markmind`，然后从 `github` 中下载相关文件，一般有 `main.js, manifest.json, style.css`，然后即可重启启用。

### Checklist

安装并启动插件后，在界面中会出现一个 `Todo List` 的标志

#todo
- [x] 任务 1
- [x] 任务 2
- [x] 任务 3

然后我们可以使用 `cmd+p` 展示上面的代办项，如果打上勾之后则代办项则不会再展示了。当然如果配合日记功能，则会在插件 Calendar 上面现实

### 日记+Calendar

日记是核心插件，我们可以创建日记，可以自己创建日记文件夹，然后点击 Calendar 上面对应的日期，这样可以创建选中日期的日记。如果在其中填写相关代办项，那么相关任务会在 Calendar 上提示。

### Dynamic Table of Contents

这个插件是在文件开头生成目录，当然其实 obsidian 本身也有目录展示。使用方式是在开头填入即可。可以看本文开头的使用。

### Easy Typing

这个无需设置，主要用于给汉字中的英文加空格

### Mind Map

这个插件也无需设置，只需编辑普通的 markdown 文档，然后使用 `cmd+p` 调用此插件即可在另外的页面以脑图形式展示。感觉不是很好用，特别是层级多了之后，看下面的插件

### MarkMind

功能比较强大，可以直接创建思维导图文件，然后就可以编辑，支持在思维导图中支持 markdown 编辑。还可以支持与 pdf 文件进行联动。[文档](https://github.com/MarkMindCkm/obsidian-markmind)

### canvas

这是一个核心插件，我们可以选中某个文件夹然后右键创建，也可以通过命令来创建，创建好之后则可以将本笔记仓库中的文档以卡片的形式添加到 canvas 中（右键添加）。添加之后可以修改卡片颜色，大小，连线等等。


## 块引用

例如 [[2、复合数据类型#文本和 HTML 模板]]

## 代码粘贴格式错乱

粘贴请使用 `Ctrl+Shift+v 或者 Cmd + Shift +v`