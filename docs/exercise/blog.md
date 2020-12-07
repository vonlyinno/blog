# Docsify速成记录

## Why Docsify

1. 方便。从安装到运行，再到部署，不会超过10min。
2. 没有花里胡哨。之前使用过hexo，在搭建过程中就会耗费一些时间（相对来说），还会忍不住精心挑选一些主题。其实作为一个初搭blog的懒人，能坚持运行下去才是重点。
3. 纯markdown，不会编译成html，摆在仓库看起来也舒适度满分。

## How
[官方文档](https://docsify.js.org/#/zh-cn/quickstart)
首先，你应该已经有一个git仓库了。

### 安装

```bash
npm i docsify-cli -g
```

### 初始化
```bash
docsify init ./docs
```

### 运行
```bash
docsify serve docs
```

### 其他
后续再补

## Deploy
[官方部署至github](https://docsify.js.org/#/zh-cn/deploy)

> 这里说下，部署非常方便，但是github.io大概是被墙了，反正我访问不了，即使使用网上说的DNS设置为114.114.114 or 223.5.5.5，都不行。
> 于是使用了国内的gitee，初步使用很方便，还有友好的中文，一键导入github仓库，真香。

## 关于markdown
既然要写markdown，免不了我要上传图片的，之前用的微博的图床，但最近白嫖了腾讯云的代金券，就使用以下对象存储作为图床。
[教程](https://cloud.tencent.com/developer/article/1720465)
