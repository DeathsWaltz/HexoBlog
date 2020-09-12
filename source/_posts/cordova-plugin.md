---
title: cordova插件开发
date: 2020-04-23 16:12:24
tags: [cordova,插件,plugin,cordova-plugin,开发]
---

#### 原因

蓝牙打印机型号：`汉印A300(HM-A300-6624)`

App接入调用蓝牙打印机进行二维码的打印，使用普通`cordova`蓝牙链接库发送指令，无法打印条形码。

#### 解决方案

##### 安装`plugman`

```bash
npm install -g plugman
```

##### 创建插件项目

```bash
# 去掉<>
plugman  create  --name  <插件名>  --plugin_id  <插件id(插件调用名)> --plugin_version <插件版本号(1.0.0)>
```

#####  添加需要开发的平台

```bash
 # 添加android平台
 plugman platform add --platform_name android 
```

#####  规范项目

修改生成的`java`代码包位置。

```bash
|   .gitignore
|   package.json
|   plugin.xml
|
|
+---src
|   \---android
|       \---cn
|           \---akapril
|               \---ble
|                       BlePrintPlugin.java
|
\---www
        BlePrintPlugin.js
```

修改完目录后需要修改`plugin.xml`(添加本地`jar`包处解释)

#####  添加本地`jar`包

在根目录新建目录`libs`,将需要的`jar`包放入目录中。

修改`plugin.xml`配置

```xml
<?xml version='1.0' encoding='utf-8'?>
<plugin id="BlePrintPlugin" version="1.0.0" 
    xmlns="http://apache.org/cordova/ns/plugins/1.0" 
    xmlns:android="http://schemas.android.com/apk/res/android">
    <name>BlePrintPlugin</name>
    <js-module name="BlePrintPlugin" src="www/BlePrintPlugin.js">
        <clobbers target="cordova.plugins.BlePrintPlugin" />
    </js-module>
    <platform name="android">
        <config-file parent="/*" target="res/xml/config.xml">
            <feature name="BlePrintPlugin">
                <!-- 修改java代码目录时候对应包 -->
                <param name="android-package" value="cn.akapril.ble.BlePrintPlugin" />
            </feature>
        </config-file>
        <config-file parent="/*" target="AndroidManifest.xml"></config-file>
        <!-- 对应文件 -->
        <source-file src="libs/CPCL_V1.07.jar" target-dir="libs"/>
        <source-file src="src/android/cn/akapril/ble/BlePrintPlugin.java" target-dir="src/cn/akapril/ble" />
    </platform>
</plugin>
```

#####  生成`package.json`

```bash
#插件目录下生成
plugman createpackagejson .
```

```bash
name: (BlePrintPlugin)
version: (1.0.0)
description:
git repository: (https://github.com/akapril/BlePrintPlugin.git)
author: akapril
license: (ISC)
About to write to E:\workspace\BlePrintPlugin\package.json:     

{
  "name": "BlePrintPlugin",
  "version": "1.0.0",
  "description": "",
  "cordova": {
    "id": "BlePrintPlugin",
    "platforms": [
      "android"
    ]
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/akapril/BlePrintPlugin.git"  
  },
  "keywords": [
    "ecosystem:cordova",
    "cordova-android"
  ],
  "author": "akapril",
  "license": "ISC",
  "bugs": {
    "url": "https://github.com/akapril/BlePrintPlugin/issues"   
  },
  "homepage": "https://github.com/akapril/BlePrintPlugin#readme"
}


Is this OK? (yes) yes
```

#####  添加到`cordova`项目中

```bash
#去掉<>
# 添加插件
cordova plugin add  <插件项目目录> 
# 去除插件
cordova plugin remove <插件名称>
# 获取github上的插件
cordova plugin add https://github.com/akapril/BlePrintPlugin.git
```



参考文档：

[https://cordova.apache.org/docs/en/6.x/guide/hybrid/plugins/index.html](https://cordova.apache.org/docs/en/6.x/guide/hybrid/plugins/index.html)

[https://cordova.apache.org/docs/en/6.x/guide/platforms/android/plugin.html](https://cordova.apache.org/docs/en/6.x/guide/platforms/android/plugin.html)

[https://cordova.apache.org/docs/en/6.x/reference/cordova-cli/index.html#cordova-plugin-command](https://cordova.apache.org/docs/en/6.x/reference/cordova-cli/index.html#cordova-plugin-command)