---
title: Hello World
date: 2020-02-24 17:04:51
---
Welcome to [Hexo](https://hexo.io/)! This is your very first post. Check [documentation](https://hexo.io/docs/) for more info. If you get any problems when using Hexo, you can find the answer in [troubleshooting](https://hexo.io/docs/troubleshooting.html) or you can ask me on [GitHub](https://github.com/hexojs/hexo/issues).

## Quick Start

### Create a new post

``` bash
$ hexo new "My New Post"
```

More info: [Writing](https://hexo.io/docs/writing.html)

### Run server

``` bash
$ hexo server
```

More info: [Server](https://hexo.io/docs/server.html)

### Generate static files

``` bash
$ hexo generate
```

More info: [Generating](https://hexo.io/docs/generating.html)

### Deploy to remote sites

``` bash
$ hexo deploy
```

More info: [Deployment](https://hexo.io/docs/one-command-deployment.html)

### enable img

1. 在根目录下配置文件_config.yml 中有 post_asset_folder:false 改为 true

2. 安装插件

```bash
npm install https://github.com/7ym0n/hexo-asset-image --save
```

3. 应用图片

_posts 目录下会有同名目录,文章中引用目录下的 test.jpg

```bash
![说明文字](test.jpg)
```
