---
layout:     post
title:      "android processor常用的dir"
subtitle:   " \"android processor常用的dir\""
date:       2017-04-19 12:00:00
author:     "ZhangSong"
header-img: "img/post-bg-2015.jpg"
tags:
    - android
---

android annotation processor 可以在编译时完成一些任务，常见方式是生成一些辅助代码。但是想改变生成代码或者文件的位置，会比较绕。
大体有两种思路，一种在processor处理中处理这些内容，另一种是processor处理完，在gradle挂个task，借助gradle中相关路径属性。


## processor 中处理

自定义processor常规做法继承```AbstractProcessor```，在```void init(ProcessingEnvironment processingEnvironment) {}```方法中拿到相应的路径变量。
 ```rocessingEnvironment.getFiler()```虽然可以拿到```Filer```变量，但是很不巧，他没有相应的工程路径的信息，但是可以根据特定的工程目录结构，拿到我们想要的位置。
比如我们是一个android gradle类型的工程结构，可以通过类似下面的方式获取工程路径,

```
private Path getProjectDir(Filer mFiler){
        Path projectPath = null;
        try {
            FileObject resource = mFiler.createResource(StandardLocation.CLASS_OUTPUT, "", "tmp", (Element[]) null);
            //E:\project\android_project\TestModuleInject\app\build\intermediates\classes\debug\tmp
            projectPath = Paths.get(resource.toUri()).getParent().getParent().getParent().getParent().getParent();
            System.out.println(projectPath);
            resource.delete();
        } catch (IOException e) {
            e.printStackTrace();
        }

        return projectPath;
    }
```
虽然拿到，但是这种方式其丑无比，利用Filer创建文件的方法创建```FileObject```，随之一路回溯，一串的```getParent()```看着令人心烦意乱.

## gradle 中挂task处理

android processor处理过程中可能会触发多次，原因是如果生成的代码中还有编译时注解，还会继续处理。
processor在compile过程中完成，可以找到相应的task来处理任务。


```
afterEvaluate{
    tasks.matching{
        it.name.startsWith("compileDebugJavaWithJavac")
        it.name.startsWith("compileReleaseJavaWithJavac")
    }.each{
        println "taskName:${it.name}"
    }
}
```


gradle中有一些路径变量可供使用：


#### projectDir
> The directory containing the project build file.
### rootDir
> The root directory of this project. The root directory is the project directory of the root project.
### buildDir
> The build directory of this project. The build directory is the directory which all artifacts are generated into. The default value for the build directory is projectDir/build

