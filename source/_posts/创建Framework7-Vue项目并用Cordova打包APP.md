---
title: 创建Framework7+Vue项目并用Cordova打包APP
date: 2017-09-22 14:56:11
tags: [Framework7,Vue,Cordova]
---

# 一、创建Cordova项目 

## 安装Cordova CLI

下载和安装Node.js。安装完成后你可以在命令行中使用node 和 npm （已安装请忽略）。
安装cordova 模块使用Nodejs的npm工具。cordova模块会被npm工具自动下载。

``` bash
npm install -g cordova
```

## 创建Cordova项目

跳转到你维护源代码的目录中，并创建你的cordova项目：

``` bash
cordova create framework7 com.example.framework7 'Frameword7 Vue'
```

这将会为你的cordova应用创造必须的目录。默认情况下，cordova create命令生成基于web的应用程序的骨骼，项目的主页是 www/index.html 文件。

## 添加平台

跳转到你维护源代码的目录中，并创建你的cordova项目：

cd framework7

给你的App添加目标平台。我们将会添加ios和android平台，并确保他们保存在了config.xml中

``` bash
cordova platform add android --save
cordova platform add ios --save
```


# 二、创建Framework7 Vue框架代码

## 下载安装

下载模板代码或者手动从GitHub库下载（把代码放在Cordova子目录）

``` bash
git clone https://github.com/nolimits4web/Framework7-Vue-Webpack-Template application
```

进入已下载的项目目录并安装依赖

``` bash
cd application
npm install
```

## 修改配置

进入config目录修改index.js文件，把index.html和assets文件配置到www目录下

``` bash
module.exports = {
  build: {
    index: path.resolve(__dirname, '../dist/index.html'),
    assetsRoot: path.resolve(__dirname, '../dist'),
  }
}
```

改成

``` bash
module.exports = {
  build: {
    index: path.resolve(__dirname, '../../www/index.html'),
    assetsRoot: path.resolve(__dirname, '../../www'),
  }
}
```

运行开发环境

``` bash
npm run dev
```

应用程序将运行服务器热重载在localhost:8080

## 创建生产代码

你的应用程序已经准备好了，你需要建立生产:

``` bash
npm run build
```

# 三、打包应用

## 构建App

运行下面命令为android平台构建:

``` bash
cordova build android
```