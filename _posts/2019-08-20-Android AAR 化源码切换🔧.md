#### 背景

对于稍微大一点的 Android 项目，将业务模块打包成aar依赖项是一种常见的操作，上传到公共或自建 maven。但是对于实际的开发过程，开发者经常需要切换 aar 依赖方式到源码，完成新功能开发或调试。

[DepSwitchPlugin](https://github.com/qiugang/DepSwitchPlugin) 是我之前尝试的一种实现方案，实现很简单，本质上就是根据配置来替换依赖，在上面提到的场景里，它可以 work。当时有收到 issues 提到说
> 多个```module```就会有很多个```dep-switch.json```，如果能把```dep-switch.json```放在项目根目录，全局只有一个```dep-switch.json```文件，这样更合理，每次修改只关注这一个 ```dep-switch.json```

多个module 都需要 apply plugin 的确很麻烦。于是有了下面的实现，对原工程侵入很少，实现利用 gradle 的 api 也更加简单： 1，2，3 switch！

#### 实现

* 切换参数本地配置

	控制切换的参数暂定设计为 ```json``` 格式，并且加入到 ```.gitignore```  中，aar 依赖切换到源码大部分场景仅在各自开发阶段使用，每个人可以按需维护一份自己本地配置，无需繁琐的修改 ```setting.gradle``` 与相关多处 ```build.gradle``` 文件
	
 	```
	{
		"debug": false,
		"configs": [
		    {
		      "remoteName": "com.squareup.retrofit2:retrofit:2.4.0"(example),
		      "localName": "${module-name}",
		      "localPath": "${local-module-path},
		      "switch": true
		    }
		    //...
		]
	}
	```

* 在 setting.gradle 中插入 include 组件本地仓库

	```
	switchConfiguration.forEach {
		settings.include(":${it.localName}")
	    settings.project(":${it.localName}").projectDir = new File(it.localPath)
	}
	```

* hook 目标 aar 依赖，项目中所有命中 config 配置的 aar 远程依赖切换到本地源码仓库

	利用  [ResolutionStrategy](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.ResolutionStrategy.html) 中的 ```dependencySubstitution``` 提供的 ```substitute``` api 可以非常方便的完成 ```module reomte```  与  ```module project``` 的相互替换。不仅如此，配合上 ```configurations.configureEach```，整个项目的依赖都可以命中替换规则

#### 工程改造

* 目标工程根目录新增 ```proj-dep-swich.gradle``` 与 ```proj-dep-swich-config.json``` 
* 项目 ```setting.gradle``` 新增

	```
	apply from: file('proj-dep-swich.gradle')
	hookSettings(getSettings())
	```