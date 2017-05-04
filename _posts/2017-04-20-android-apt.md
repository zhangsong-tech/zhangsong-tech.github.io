---
layout:     post
title:      "android-apt不再支持"
subtitle:   " \"annotationProcessor粉墨登场\""
date:       2017-04-20 12:00:00
author:     "ZhangSong"
header-img: "img/post-bg-2015.jpg"
tags:
    - android
---


最近项目需要一些编译时处理，外网又很受限制。
以前经常会用到的apt，半天下载不动。

```groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    }
}
apply plugin: 'com.neenbedankt.android-apt'
```

以往在classpath中添加依赖，并且指明android-apt的plugin类型。

最近看到android-apt的作者宣布不再更新，Android-apt可能将退出历史舞台。


Android gradle 2.2 以后推出了替代方案 ，**annotationProcessor**。

需要 将Module 的 build.gradle 文件中使用 apt 引入的依赖修改为使用 annotationProcessor 进行引入，修改前配置如下：


```groovy
dependencies {
  compile 'com.google.dagger:dagger:2.0'
  apt 'com.google.dagger:dagger-compiler:2.0'
}
```

修改为：

```groovy
dependencies {
    compile 'com.google.dagger:dagger:2.0'
    annotationProcessor 'com.google.dagger:dagger-compiler:2.0'
    compile project(':module-annotation')
    annotationProcessor project(':module-compiler')
}
```
那个module需要编译时处理，给哪个module添加

annotationProcessor号称既支持jack，又支持javac，但是最近看jack可能也是命途多舛，但annotationProcessor的官方支持还是很不错的。
和```apt{}```类似，annotationProcessor也支持gradle配置参数，方式如下：

```groovy
android {
...
    defaultConfig {
    ...
        javaCompileOptions { 
            annotationProcessorOptions {
                className 'com.example.MyProcessor'

        // Arguments are optional.
                arguments = [ foo : 'bar' ]
            }
        }
    }
    ...
}
```


#### reference:
* [What's next for android-apt](http://www.littlerobots.nl/blog/Whats-next-for-android-apt/)
* [android-apt Migration](https://bitbucket.org/hvisser/android-apt/wiki/Migration)