---
title: 三件事：Aar, Manifest和Activity-Alias
tags:
  - Android
comment: true
categories: Android
visible: visible
date: 2016-12-10 16:43:29
---

前几天在做需求的时候，接触了一些之间没有太了解的东西，于是今天找个机会写下来给大家分享一下，说不定大家以后也会有这种需求，可以提前了解一下，主要有三个点：aar文件的类型，Manifest文件的自动合并以及Activity-Alias

<!-- more -->

### 关于aar文件
为什么会存在这种文件格式，和`jar`文件有什么区别？先来句综述：`aar`文件是建立在`jar`文件的基础之上，`aar`是`jar`文件的一个变种。其实他们本质上没有什么区别，它们都是压缩包，只是能包含的内容不一样，aar包括的东西更多一些，jar也能包含资源文件，不过是文本资源和图片资源，不能包含Android平台下的drawable以及各种xml文件，`aar`相对来说更包容一些，`aar`文件结构其实更类似我们应用的apk安装包，我们通过Gradle compile进来的库也都是以`aar`文件引入进来的。

具体怎么生成一个`aar`文件以及引入本地`aar`文件，可以参考Android Developer的文档进行操作，因为官方的文档已经描述的很清楚了，所以我在这里也没必要在赘述什么，有兴趣的可以直接传送。[传送门（自备梯子）](https://developer.android.com/studio/projects/android-library.html)


### Manifest 文件合并
刚才其实忘了说，`aar`文件中除了必要的class文件，还有一些必要的文件，其中就包括`AndroidManifest`文件，但是我们也知道，一个应用只能有一个`Manifest`文件，那当我们引入其他依赖库的Manifest文件是怎么处理的呢？没错，就是合并，最终所有的`Manifest`文件都会被合并成一个`Manifest`文件，那么我们要讨论的问题就出现了，按照什么顺序合并呢？合并出现冲突怎么解决呢？接下来慢慢解释：

#### 合并优先级
优先级从高到低依次为：

1. build.gradle 配置
2. app module 的`Manifest`文件，也就是我们主项目中的`Manifest`文件
3. 其他依赖库的`Manifest`文件

下面这张图解释了优先级的作用
![](http://static.shaohui.me/QQ20161210-0@2x.png)

除了合并的优先级，还有一些潜在的合并规则也需要注意一下：

1. 在 `<manifest>`节点中的属性都会已最高优先级指定的值为准，所以不会出现值不一样导致的冲突。这也解释了为什么我们只要在build.gradle文件中指定了versionCode或者versionName，那么项目的Manifest文件中配置的相应属性总是会被覆盖。

2. 在` <uses-feature>` 和`<uses-library>`节点中的`android:required`属性会在多个配置中采用或运算，也就是说，只有有一个配置设置了`true`，那这条属性就是`true`，表示这个`feature`或者`library`是必需的。

3. 在`<uses-sdk>`中的属性，一般情况下，也是以最高优先级的配置为准，不过也有两种特殊情况：

	1. 如果低优先级的库的`minSdkVersion`更高，这时候需要处理一下冲突，可以使用` overrideLibrary`来覆盖因入库的`minSdkVersion`，这样冲突是解决了，但是这样会存在隐患问题，一般第三方库的`minSdkVersion`用来表示最低可运行版本，你如果通过覆盖的方式解决了冲突，但是在低版本运行的时候，可能会出问题，所以更安全的解决方式就是调高自己的`minSdkVersion`或者换用其他库
	2. 低优先级的库的`targetSdkVersion`更低，表示这个库还没有为更高的版本做好适配的准备，这时候虽然合并Mainfest文件并并不会出现冲突，但是也会存在一些隐患问题，比如在`targetSdkVersion`低于15，并且声明了`READ_CONTACTS`的权限，那么在更高的版本上需要额外的增加`READ_CALL_LOG`权限，以让应用正常运行，而这件事，合并工具会自动帮我们完成。

4. `<intent-filter> `标签不会被匹配，它们每一个都会被认为是独一无二的，然后一起加在相同的父标签里。

除了通过既定的规则，我们可以猜到最后合并的`Manifest`文件大概是什么样子，我们还可以直接在Android Studio中直观地实时看到合并以后的`Manifest`文件，就是在`Manifest`文件的底部有一个Tab可以看到Merged Manifest，在这里，我们可以清楚的看到我们最终的`Manifest`文件是什么结构，看有没有恶意的第三方库在我们的`Manifest`文件中动手脚。
![官方示意图](http://static.shaohui.me/manifest-merged-view_2x.png)

除了这些基本的规则，我们还可以在`Manifest`文件中定义自己的规则，以防止冲突的出现或者当冲突出现的时候，用来解决冲突，因为具体规则比较多，所以就不在这里展开，大家都兴趣或者有需求的可以直接上Android Developer上看官方的文档，[传送门（自备梯子）](https://developer.android.com/studio/build/manifest-merge.html)

### Activity-alias

```
<activity android:name="me.shaohui.shareutil._ShareActivity"
	android:theme="@android:style/Theme.Translucent.NoTitleBar" />
<activity-alias
	android:name="$me.shaohui.shareutil.wxapi.WXEntryActivity"
	android:exported="true"
	android:targetActivity="me.shaohui.shareutil._ShareActivity"/>
```

大家通过字面意思就能看得出来，这个元素就是给`TargetActivity`属性所指定的`Activity`设定一个别名，之前一直不知道在什么情况需要给`Activity`指定别名，这次在做一个社会化分享库的时候，突然明白了`Activity-Alias`存在的意义了。

Android微信的分享SDK为了接受微信请求的回调以及返回值，需要在应用包名相应的目录下新建一个wxapi目录，并且在该目录下新增一个名为`WXEntryActivity`的`Activity`，处理好以后，会在发起微信请求以及请求完成以后，拉起这个`Activity`。这对我们使用者来说，单独为微信创建一个目录存放一个特定的`Activity`是个很蛋疼的一个操作，这样会打乱我们的项目结构，而且看起来也很是不优雅，苦思冥想许久，怎么能避免这样的尴尬，最后遇到了`Activity-Alias`这种解决方案，觉得简直就是碰到了亲人，

最后的解决方案就是大家前面看到的那段代码，我只需要在普通目录下定义我们接收微信回调的`Activity`，然后再给这个`Activity`定义一个`me.shaohui.shareutil.wxapi.WXEntryActivity`别名，这样就既满足了微信的调用需求，还不用给这个`Activity`进行特殊化处理，只要微信通过这个别名就能找到我们的目标`Activity`，我们的目标`Activity`可以随意按照我们既定的目录结构存放，不需要特殊处理，皆大欢喜。

`Activity-Alias`也有一些自己的属性，大部分都是和`Activity`节点的属性类似，只有一个`targetActivity`需要我们重点关注，它定义了这个别名是给哪个`Activity`设置的，而且要求指定的`Activity`必须在`Activity-Alias`之前被声明，否则最后应用安装的时候会出问题。

目前想到的就这些，以后再有新的想法再慢慢扩充，如果有哪写的不对的地方，欢迎大家指出，谢谢！

### 参考链接
1. https://developer.android.com/studio/projects/android-library.html
2. https://developer.android.com/studio/build/manifest-merge.html
