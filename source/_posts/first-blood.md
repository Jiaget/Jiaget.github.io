---
title: first blood
date: 2021-05-28 15:16:52
tags: 
    - hexo
    - 博客
categories: 工具
---
# 0.初次见面

拖延症患者，拖拖拉拉，终于开始搭建自己的博客。本博客使用 `Hexo` 和 `Github page` 。下面是博客搭建的历程。

# 1.准备环境
- [NodeJS](https://nodejs.org/en/) 
- git

# 2.安装Hexo

[Hexo官网](https://hexo.io/zh-cn/)

`npm` 是NodeJS的包管理工具，关于Hexo的安装或者依赖的下载都可以交给它完成

`npm install -g hexo-cli`

安装完成后执行以下命令
```
hexo init myBlog
cd myBlog
npm install
```

## 测试hexo

```
hexo s
```

访问`http://localhost:4040`可以预览博客效果。

# 3.Github Page 部署

- 新建仓库
    - 注意仓库名严格按照`[用户名].github.io`命名
- SSH key 配置
```
git config --global user.name "用户名"
git config --global user.email "邮箱地址"
ssh-keygen -t rsa -C '上面的邮箱'
cat ~/.ssh/id_rsa.pub
```
- 将生成密钥添加到 Github 上
    - `settings -> SSH and GPG keys -> New SSH key`

- 本地部署到Github
    - 博客目录下 `_config.yml`,在文件末尾添加：
        ```
        deploy:
        type: git
        repo:
            github: https://github.com/Jiaget/Jiaget.github.io.git,main
        ```
    - 安装部署插件 `npm install hexo-deployer-git --save`
    - 博客页面生成与上传 `hexo g -d`
- 上传到github到最后效果的显示有段时间延迟，访问 ` https://你的用户名.github.io`就能看到自己的博客了。

# 4.写作

- 新建文章 `hexo new '文章标题'`
    - 文章会生成在 `/source/_posts` 下
- 执行 
```
hexo g
hexo s
```
文章在博客中显示

- 部署
```
hexo clean
hexo g -d
```
`hexo clean ` 是为了清除缓存文件`db.json`以及静态文件`public`。

- 写好的文章暂时不想发布，可以存在草稿里，草稿的生成方法 `exo new draft "文章标题"`
    - 草稿保存在`/source/_drafts`中，需要发布只需要`hexo publicsh [layout]<filename>`

## git 源码管理

hexo会把`./deploy_git`中的静态页面上传给github, 也就是说我们写的markdown文件以及hexo源码都在本地储存。

因此我们还需要将源码托管到github。不需要新建仓库，只需要新的分支管理即可。

关于分支，这里需要提一句，在 `github page` 部署的静态页面应该在 `main` 分支，而不是 `master`分支。github应该是将主分支名称修改了，原先一直是 `master`。

- 将本地源码和 github 仓库连接 `git remote add origin git@github.com:Jiaget/Jiaget.github.io.git`

- 检查 `.gitignore`, 忽视无关文件。如果没有该文件，则新建一个。

```
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
```

最后推送给新的分支

```
git add .
git commit -m "hexo source code"
git push origin master:source
```