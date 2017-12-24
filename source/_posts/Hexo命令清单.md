---
title: Hexo命令清单
date: 2017-12-18 12:37:04
categories:
  - Hexo
tags:
  - Hexo
typora-copy-images-to: ./Hexo命令清单

---

# 常用命令

创建文章, layout可使用 post(默认), draft(草稿), page ...

```
$ hexo new [layout] "Article Name"
```

发布草稿

```
$ hexo publish [layout] <title>
```

生成静态文件

```
$ hexo generate
$ hexo g
```

发布更新

```
$ hexo deploy
$ hexo d
```

生成并发布

```
$ hexo g -d
```

启动本地server查看, 默认： localhost:4000, 可自行指定参数

```
$ hexo server
$ hexo s
```



# 官方地址：

[https://hexo.io](https://hexo.io)

[https://hexo.io/docs/](https://hexo.io/docs/)

[https://hexo.io/docs/troubleshooting.html](https://hexo.io/docs/troubleshooting.html)

[https://github.com/hexojs/hexo/issues](https://github.com/hexojs/hexo/issues)