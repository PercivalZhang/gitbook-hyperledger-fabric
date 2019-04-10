# 超级账本v1.4文档

This repo contains 超级账本v1.4文档。

### Gitbook观看浏览

The spec follows gitbook format, and can be viewed using the gitbook tooling.

## GitBook 准备工作

### 安装 Node.js

GitBook 是一个基于 Node.js 的命令行工具，下载安装 [Node.js](https://link.jianshu.com?t=https%3A%2F%2Fnodejs.org%2Fen)，安装完成之后，你可以使用下面的命令来检验是否安装成功。

```shell
$ node -v
```

### 安装 GitBook

输入下面的命令来安装 GitBook。

```shell
$ npm install gitbook-cli -g
```

安装完成之后，你可以使用下面的命令来检验是否安装成功。

```shell
$ gitbook -V
```

更多详情请参照 [GitBook 安装文档](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FGitbookIO%2Fgitbook%2Fblob%2Fmaster%2Fdocs%2Fsetup.md) 来安装 GitBook。

输入以下命令：

```
gitbook serve
```

然后在浏览器地址栏中输入 `http://localhost:4000` 便可预览书籍。

运行该命令后会在书籍的文件夹中生成一个 `_book` 文件夹, 里面的内容即为生成的 html 文件，我们可以使用下面命令来生成网页而不开启服务器。

```shell
gitbook build
```

你也可以输出PDF格式:

```
gitbook pdf
```

(Note: on macOS using the pdf printer may require some extra installation)

## 作者

- [@Percivalzhang](https://github.com/Percivalzhang)