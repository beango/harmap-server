---
title: HEXO+Github搭建博客
date: 2017-08-21 15:28:46
category: web
tags: [Hexo,GitHub Pages,HEXO+Github搭建博客]
---

## GitHub 个人主页

即通过 http://[username].github.com/ 访问到的页面。这个比较简单，只要在 GitHub 上建立一个名为 [username].github.com 的项目，把页面内容通过 Git 工具从本地推送上去即可。

<!--more-->

## GitHub 项目演示页面

### HEXO安装调试

即通过 http://[username].github.com/[projectname] 访问到的项目演示页面。
下面分解HEXO建立博客的步骤。

安装Hexo：

``` bash
sudo npm install -g hexo
```

然后初始化:

``` bash
hexo init
```

至此，安装成功。接下来就是折腾了。

生成静态页面

``` bash
hexo generate（hexo g也可以）
```

启动本地服务，：

``` bash
hexo server
```

浏览器输入 http://localhost:4000 就可以预览了。

### 配置Github

建立Repository

### 配置hexo

配置代码仓库：

``` bash
deploy:
     type: git
     repo: https://github.com/[username]/[repository].github.io.git
     branch: master
```

配置主题：

``` bash
mkdir themes/next
git clone https://github.com/iissnan/hexo-theme-next themes/next
```

修改_config.yml
``` bash
theme: next
```

然后执行命令：

``` bash
npm install hexo-deployer-git --save
```

执行部署命令：

``` bash
hexo deploy
```

### 部署步骤

每次部署的步骤，可按以下三步来进行。

``` bash
hexo clean
hexo generate
hexo deploy
```

一些常用命令：

``` bash
hexo new"postName" #新建文章
hexo new page"pageName" #新建页面
hexo generate #生成静态页面至public目录
hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
hexo deploy #将.deploy目录部署到GitHub
hexo help # 查看帮助
hexo version #查看Hexo的版本
```

### 绑定域名

Settings翻到GitHub Pages节，select source 选“gh-pages branch”并save，Custom Domain 填写域名并save。
将域名解析到[username].github.io代码仓库的IP地址，并添加CNAME记录目标为[username].github.io即可。

## 集成 Algolia 搜索

如果你是使用的 [Next](https://github.com/iissnan/hexo-theme-next) 主题，其5.1以上版本已经集成该功能

### 激活

在站点配置文件_config.yml中编辑配置

``` bash
algolia:
  applicationID: 'applicationID'
  apiKey: 'apiKey'
  adminApiKey: 'adminApiKey'
  indexName: 'indexName'
  chunkSize: 5000
```

在主题配置文件_config.yml中打开 Algolia 搜索开关

``` bash
algolia_search:
  enable: true
```

### 将索引数据上传

[注册帐号](https://www.algolia.com/)

新建 Index ，并获取其 applicationID , apiKey , adminApiKey

接下来安装hexo-algoliasearch插件

``` bash	
npm install --save hexo-algolia
```
生成索引数据并上传：

``` bash
hexo algolia
```

注意，hexo-algolia 需要 "hexo-algolia": "^0.2.0", 修改API Keys权限为Add records, Delete records, List indices, Delete index
上传不成功先hexo clean。
