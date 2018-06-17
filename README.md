# hexo-blog-source
hexo博客源码


## 快速开始

本博客使用[hexo](https://hexo.io/zh-cn/)搭建，主题使用[material](https://github.com/viosey/hexo-theme-material)，并做了符合自己的个性化配置。

### 下载代码

```
git clone git@github.com:schhx/hexo-blog-source.git
cd hexo-blog-source
```

### 安装依赖

```
yarn install
```

### 启动项目

```
hexo s
```

可以使用浏览器打开```http://localhost:4000```预览

### 新建一篇博客

```
hexo new new-blog
```

在source/_post目录下新建一个```new-blog.md```的文件，同时会新建一个同名的文件夹放置资源文件（例如图片）

### 发布到Github Pages

```
hexo clean && hexo d -g
```

Github Pages配置在站配置文件末尾
```
deploy:
  type: git
  repo: git@github.com:schhx/schhx.github.io.git
  branch: master
```

### 提交源码

```
git add .
git commit -m "source"
git push -u origin master
```
