---
layout:     post
title:      "android N 文件系统权限适配"
subtitle:   " \"android file provider\""
date:       2017-04-18 12:00:00
author:     "ZhangSong"
header-img: "img/post-bg-2015.jpg"
tags:
    - android
---

android的更新速度已经上天了，android N的装机率还少的可怜，但是适配工作一点也不轻松。
增强的权限限制收紧了应用间的文件共享，转而提供更加安全的```FileProvider```方式来共享。
否则在android N上面会爆出```FileUriExposedException```错误。



### 反射disableDeathOnFileUriExposure()方法
有人提出反射的方式绕开，但是路子太野,也没有实测。
目测是更改StrictMode中的检测方式，会在application中被调用，实现的效果可能和下面的```StrictMode.setVmPolicy()```效果类似。

```
try {
Method ddfu = StrictMode.class.getDeclaredMethod("disableDeathOnFileUriExposure");
ddfu.invoke(null);
} catch (Exception e) {
}
```

### StrictMode.setVmPolicy()设置Vmpolicy

也有方法设置新的VmPolicy，也不太符合现在android的安全规范趋势。
在sdk24中使用一个严格模式，```StrictMode.VmPolicy.Builder.detectFileUriExposure()```,可以给StrictMode设置新的policy.

application启动时设置：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    StrictMode.VmPolicy.Builder builder = new StrictMode.VmPolicy.Builder();
    StrictMode.setVmPolicy(builder.build());
}
```

### File Provoder 方式

官方对于 FileProvider 的解释为：FileProvider 是一个特殊的 ContentProvider 子类，通过 content://Uri 代替 file://Uri 实现不同 App 间的文件安全共享。
当通过包含 Content URI 的 Intent 共享文件时，需要申请临时的读写权限，可以通过 ```Intent.setFlags()``` 方法实现。
而 file://Uri 方式需要申请长期有效的文件读写权限，直到这个权限被手动改变为止，这是极其不安全的做法。因此 Android 从 N 版本开始禁止通过 file://Uri 在不同 App 之间共享文件。

##### 完成整个文件共享的流程，需要配置以下5点：
1. 定义一个 FileProvider
2. 指定有效的文件
3. 为文件生成有效的 Content URI
4. 申请临时的读写权限
5. 发送 Content URI 至其他的 App

通常使用的方法
```
Uri uri = Uri.fromFile(tempFile);
```
这种方法拿到的就是file://Uri形式，需要使用File Provider来处理。

FileProvider继承自ContentProvider，需要提前注册：
```
manifest>
    ...
    <application>
        ...
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="${applicationId}.provider"
            android:exported="false"
            android:grantUriPermissions="true">
                <meta-data
                    android:name="android.support.FILE_PROVIDER_PATHS"
                    android:resource="@xml/file_paths" />
            ...
        </provider>
        ...
    </application>
</manifest>
```

##### 重要的属性包括以下四个：
* 设置 android:name 为android.support.v4.content.FileProvider，这是固定的，不需要手动更改；
* 设置 android:authorities 为 application id + .provider ；
* 设置 android:exported 为 false ，表示 FileProvider 不是公开的；
* 设置 android:grantUriPermissions 为 true 表示允许临时读写文件。

android:authorities 最好是 application id 而不能直接用包名硬编码，因为 Android 系统要求 android:authorities 对于每个 App 而言必须是唯一的。
假如 FileProvider 用在 SDK 中，多个 App 都在调用同一个 SDK，而 SDK 中的 android:authorities 为硬编码，那么 App 之间的 authorities 就会出现冲突，会报 Install shows error in console: INSTALL FAILED CONFLICTING PROVIDER 的错误。
如果 SDK 的 android:authorities 是 application id，那么 authorities 会和宿主 App 的 application id 保持一致，就不会出现 authorities 冲突的问题。
在 Java 代码中调用 getPackageName() 返回的是 application id ，而非 package name ，要验证这一点也很容易，在 build.gradle 文件中定义和包名不同的 application id ，打印代码中 getPackageName() 的返回值，就会发现返回值是 build.gradle 中自定义的 application id ，而非 package name



#### 指定有效的文件
在生成 Content URI 之前你还需要提前指定文件目录，通常的做法是在 res 目录下新建一个 xml 文件夹，然后创建一个 xml 文件，在此文件中指定共享文件的路径和名字，示例如下：
```
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-path name="my_images" path="images/"/>
    ...
</paths>
```

name：一个引用字符串。
path：文件夹“相对路径”，完整路径取决于当前的标签类型。

path可以为空，表示指定目录下的所有文件、文件夹都可以被共享。
```<paths>```这个元素内可以包含以下一个或多个，具体如下：

```<files-path name="name" path="path" />```

物理路径相当于Context.getFilesDir() + /path/。

```<cache-path name="name" path="path" />```

物理路径相当于Context.getCacheDir() + /path/。

```<external-path name="name" path="path" />```

物理路径相当于Environment.getExternalStorageDirectory() + /path/。

```<external-files-path name="name" path="path" />```

物理路径相当于Context.getExternalFilesDir(String) + /path/。

```<external-cache-path name="name" path="path" />```

物理路径相当于Context.getExternalCacheDir() + /path/。

```<root-path name="name" path="path" />```

物理路径相当于/path/，不过不在官方文档里面。


#### 为共享文件生成 Content URI
文件配置完成后还需要生成可以被其他 App 访问的 Content URI，可以直接调用 FileProvider 提供的 getUriForFile(File file) 方法
```
File imagePath = new File(getContext().getFilesDir(), "images");
File newFile = new File(imagePath, "default_image.jpg");
Uri contentUri = FileProvider.getUriForFile(getContext(), "com.mydomain.provider", newFile);
```
getUriForFile：第一个参数是Context；第二个参数，就是我们之前在manifest#provider中定义的android:authorities属性的值；第三个参数是File。

#### 给Uri授予临时权限

临时授权方法如下：

```
protected void onCreate(Bundle savedInstanceState) {
        ...
        // Define a listener that responds to clicks in the ListView
        mFileListView.setOnItemClickListener(
                new AdapterView.OnItemClickListener() {
            @Override
            public void onItemClick(AdapterView<?> adapterView,
                    View view,
                    int position,
                    long rowId) {
                ...
                if (fileUri != null) {
                    // Grant temporary read permission to the content URI
                    mResultIntent.addFlags(
                        Intent.FLAG_GRANT_READ_URI_PERMISSION);
                }
                ...
             }
             ...
        });
    ...
    }
```


文档中指出：
> Calling ```setFlags()``` is the only way to securely grant access to your files using temporary access permissions. Avoid calling ```Context.grantUriPermission()``` method for a file's content URI, since this method grants access that you can only revoke by calling ```Context.revokeUriPermission()```.
关键点在临时二字，给Intent设置flag，真的是临时，在intent跨activity传递文件结束后，授权也失效。

除了这种方式，还有``` Context.grantUriPermission(package, Uri, mode_flags)```方式。这种授权方式会持续到调用```Context.revokeUriPermission()```或者设备重启。

#### 例子

调用相机获取图片可以用如下代码实现：
```
Intent intent = new Intent();
intent.setAction(MediaStore.ACTION_IMAGE_CAPTURE);

// 系统版本大于N的统一用FileProvider处理
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {

    // 将文件转换成content://Uri的形式
    Uri photoURI = FileProvider.getUriForFile(activity,
            activity.getPackageName()+ ".provider",
            new File(photoPath));

    // 申请临时访问权限
    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_GRANT_READ_URI_PERMISSION
            | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
    intent.putExtra(MediaStore.EXTRA_OUTPUT, photoURI);
} else {
    intent.addCategory(Intent.CATEGORY_DEFAULT);
    Uri uri = Uri.parse("file://" + photoPath);
    intent.putExtra(MediaStore.EXTRA_OUTPUT, uri);
}
activity.startActivityForResult(intent, requestCode);
```






Reference:
* [FileProvider](https://developer.android.google.cn/reference/android/support/v4/content/FileProvider.html)
* [ApplicationId 与 PackageName 的区别](http://blog.csdn.net/feelang/article/details/51493501)
