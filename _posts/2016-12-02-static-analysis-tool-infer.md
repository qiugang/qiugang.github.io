---
title: Use infer to improve code
---

coding 毕竟是一项人为的操作，平时即使是谨慎的去写代码，难免还是会有一些比较低级的错误。而且最为尴尬的是，就是因为比较低级简单，往往 code review 时还不容易发现。NullPointer, RESOURCE_LEAK 等时不时的还是会出现。

这里介绍 Facebook 开源的静态代码检测工具 [Infer](https://github.com/facebook/infer) 来提高代码质量，在项目中输出高质量代码， 也会大大提升我们编码的信心。Infer 同时对 Java, C, Objective-C 提供了支持。对于 Android 开发者而言，最为利好的是：可以配合 Gradle 使用 🎉🎉🎉

简单的使用 Infer，先使用`./geadlew clean` clean下 project，然后使用命令 `infer -- ./gradlew build` 就可以去倒杯水，上个厕所啥的了(code 不多就默默等下呗)，等待 gradle 执行完 build task 后，infer 就会开始分析代码，如果检测到有问题的代码，结果会像这样
![infer-log](/assets/img/2016-12-02-infer-log.png)

同时在项目根目录下生成 info-out/ , 其中 bug.txt 列举了所有有问题代码的的位置和原因。

官方 [doc](http://fbinfer.com/docs/getting-started.html) 中还有一些高级的用法，后面如果有使用到的话再来总结一下。值得一提的是，infer 配合上 support-annotations ，更有利于 coding 时的规范约束，同时还能增强代码可读性。






