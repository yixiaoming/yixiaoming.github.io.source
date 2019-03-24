---
title: Gradle3.0多渠道打包详解
tags:
  - Android
  - Gradle
categories:
  - Android打包
comments: false
date: 2017-12-31 11:41:28
---


## 打包

Android开发中，经常我们需要打不同的包，常用的维度就是debug，release，多个渠道等，如果每个包都需要对应一份代码，那么工作量就巨大并且难以维护（修改代码需要修改多处）。Android中使用Gradle打包，可以很好的解决打包这个问题，提供了充分的可配置选项，让我们在很多维度控制打包的细节，非常灵活，推荐看官方文档（墙外）。

<!-- more -->

## productFlavor

### 基本使用

productFlavor：产品风味。在多渠道打包是一般使用这个属性，可根据不同的渠道配置不同的内容。Gradle3.0之后配置flavor与之前略有不同，需要强制设置flavor的维度：flavorDimensions，否则gradle会报错，来个例子：

```
flavorDimensions('money')
productFlavors {
    free {
        dimension 'money'
    }
    paid {
        dimension 'money'
    }
}
```

例子中定义了一个产品风味维度：money（是否需要钱钱），然后定义了两种风味：free和paid，注意两种风味的`dimension`属性相同都是`money`。然后使用`./gradlew assemble`打包，如果你配置了你会发现app/build/outputs/apk 下会打出这些包：

```
apk/
	free/
		debug/
			xxx.apk
		release/
			xxx.apk
	paid/
		debug/
			xxx.apk
		release/
			xxx.apk
```

xxx.apk 命名规则是：`app-flavor-buildtype.apk`

`./gradlew assemble`很粗暴，会打出所有的包。如果你只想打一个包，比如：paid的release包，那么可以使用以下命令：

```
./gradlew assemblePaidRelease
```

规则：`./gradlew assembleFlavorBuildtype`, buildtype（构建类型）指debug or release or 自己配置的buildtype。

### flavorDimensions

产品风味维度，打包时可以使用多个维度组合的方式。例如定义下面这种方式：

```
flavorDimensions('money', 'channel')
productFlavors {
    free {
        dimension 'money'
        ...
    }
    paid {
        dimension 'money'
    }
    baidu {
        dimension 'channel'
    }
    wandoujia {
        dimension 'channel'
    }
}
```

然后`./gradlew assemble`会打出 2x2=4个包，gralde会自动将两个维度的品味组合打包：free-baidu.apk, paid-baidu.apk, free-wandoujia.apk, paid-wandoujia.apk ...

### 配置依赖

关于不同的风味需要配置哪些属性，这里就不细讲，感兴趣自己查文档。我这里遇到一个需求是，不同的产品风味，需要依赖不同的lib。例如：我有两套log系统（一套会上传log，一套不会），分别写在两个lib中，主工程中定义了Log.java继承与两个lib中的BaseLog.java，BaseLog.java中控制log的细节。现在需要根据log这个风味维度打出两种不同log的包。大概配置如下：

```
flavorDimensions('log')
productFlavors {
    online {
        dimension 'log'
    }
    local {
        dimension 'log'
    }
}
...

dependencies {
    ...
    onlineImplementation project(':libs:lib-log-online')
    localImplementation project(':libs:lib-log-local')
}
```

主要看dependencies中两个配置：`onlineImplementation`，`localImplementation`分配表示online和local两种风味依赖两个本地不同的lib。gradle3.0之前是使用 `comple xxx`这里需要注意修改成`implementation`，命名规则依然是: `flavorImplementation`。

这样就可以实现根据不同的风味，控制不同的依赖。具体的配置还有很多内容，推荐看官方文档，这里不再累述。



## 参考地址

[https://developer.android.com/studio/build/build-variants.html](https://developer.android.com/studio/build/build-variants.html)

[https://developer.android.com/studio/build/index.html](https://developer.android.com/studio/build/index.html)



