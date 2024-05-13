
安装 n 包

```sh
npm install -g n
```

`n` 包是一个 npm 的包，可以用于 `node.js` 版本管理。

安装新版本的 node

```sh
n lts
// 或者
n latest
```

上面两个命令安装长期支持和最新版本的 `Node.js`。

删除以前安装的版本

```sh
n prune
```

此命令会删除以前安装的版本的缓存版本，只保留最新安装的版本。

更新 npm

```sh
npm install -g npm@latest
```


参考：`https://www.freecodecamp.org/chinese/news/how-to-update-node-and-npm-to-the-latest-version/`