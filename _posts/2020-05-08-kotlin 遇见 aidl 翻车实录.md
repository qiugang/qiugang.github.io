
#### 背景

最近开发需求时，在某 kt 文件中 使用到了 A.java 类，其在 aidl 中有同名声明(Parcelable)，在 QA 介入完事儿、review 代码后稳入老🐶合入主干。合并完后，群里有同学反馈拉了主干代码后编译不过，代码报错定位到 kotlin 代码中找不到 A 类引用, 实际上该类是在项目主工程内，我在本地打 debug 包、主干打 release 包都 ok 的情况下，此处心情有些复杂。

![编译错误][1]

于是开始挨着排查可疑的点，尝试修改一些可疑的地方，还是报同样错误。最后临时处理方案将 kotlin 代码改为了 java 版本，编译通过。
怀疑为 kotlin 引用 java aidl 同名类的原因，于是另起了一个 demo 工程，配置参照项目，逐个对比了变量，最后实验暂时得出为 buildTools 的 bug，将 buildTools Version 更改为 29.0.3(项目为 29.0.2) ，上诉场景可正常编译通过，查看 29.0.3 版本 change log（常年一句话） 、commit 暂未发现有提及该项场景修复，谜底依旧。。。

对于这种非常诡异的 bug，比较幸运的是实验出了解决方法，这样可以对比着解决方法，来反推导致问题的原因。

#### BuildTools

既然怀疑到 buildTools，首先要先定位 buildTools 下哪个工具的变化，导致了问题，仔细对比了下 29.0.3 与 29.0.2 的编译结果的区别，在 ```app/build/generated/aidl_source_output_dir/debug/compileDebugAidl/out``` 目录下， 针对 A.aidl (声明为 Parcelable), 29.0.2 生成了一个 A.java 的空文件，里面只有一行自动生成的注释，29.0.3 中则没有生成这个空文件，于是大概可以定位到 aidl 工具引起的变更，所以需要查找下 aidl 的源码，看是否在 29.0.3 中做出了这个行为变更

通过定位 29.0.2 与 29.0.3 的 release 日期，在期间的 commit 中找到了这样一条提交

![aidl源码][2]

查看下代码

```
generate_java.cpp#generate_java

// v29.0.2, v29.0.3 移除了这个代码块
if (const AidlParcelable* parcelable_decl = defined_type->AsParcelable();
      parcelable_decl != nullptr) {
    return generate_java_parcel_declaration(filename, io_delegate);
}

```

确实是在 29.0.3 中，不会对声明为 Parcelable 的 aidl 生成空的 java 文件了，所以到底是不是这个空的 java 文件导致 kotlin 的编译错误呢？

#### CompileKotlin Task
回头继续看下编译出错的 task: compileDebugKotlin, 这个 task 是由 kotlin-gradle-plugin 提供的，所以继续跟进源码，终于发现报错信息是在 GradleCompilationResults 中进行打印的，无法提供更多的信息。开启 --debug 编译查看 debug 信息，一段日志成功的引起了注意
![aidl源码][4]

```
-Xjava-source-roots=/Users/qiu/dev/aidltest/app/build/generated/aidl_source_output_dir/debug/compileDebugAidl/out,/Users/qiu/dev/aidltest/app/build/generated/not_namespaced_r_class_sources/debug/r,/Users/qiu/dev/aidltest/app/src/main/java,/Users/qiu/dev/aidltest/app/build/generated/source/buildConfig/debug 

```
如日志打印，kotlin compiler 在执行时，使用 ```-Xjava-source-roots=``` 指定了 java 代码的 path，而且 aidl 的 path 是在  ```src/main/java``` 之前的，会不会因为 kotlin compiler 按照顺序找 java file 编译时，首先找到 aidl 下的 A.java 是个空文件，所以导致了编译错误呢？

#### Kotlin Compiler

简单的针对 kotlin compiler 做个实验， 新建一个如下的测试

```
Main.kt
/path1/Sample.java（empty file）
/path2/Sample.java


// Main.kt
fun main() {
    //...
    Sample().name
    Sample().packageName
}

// Sample.java
public class Sample{
    public final String packageName;
    public final String id;
    public final String name;
}

```
同样安装与项目相同的 kotlin 1.3.50 版本，进行如下编译：

```
kotlinc -Xjava-source-roots=./path1,./path2 Main.kt
```
结果出现了一样的错误
```
Main.kt:3:5: error: unresolved reference: Sample
    Sample().name
    ^
Main.kt:4:5: error: unresolved reference: Sample
    Sample().packageName
    ^
```
当移除 path1 下的空 java 文件或者将 path2 放在前面，编译成功。

#### 总结

问题原因基本上能确定了，kotlin compiler java-source-roots path 的先后顺序是对其有影响的，kotlin compiler 对这种属于异常的情况兼容的不是太美丽。而 aidl 一次优化，又无心的帮助了 koltin compiler 避过了这次坑，所有的 bug 都是命中注定。顺带一提的是，Google 写 release log 什么时候才能走走心，一句话的 change log 简直了=。=

还有一个问题，同样的代码本地、打包机存在编译成功的情况？怀疑可能和编译缓存、或者对于 kotlin compiler 有某种 case，后续再跟进 kotlin compiler 流程，有结论再撸一番外篇。


[1]: /assets/img/compile-debug-kotlin-error.png
[2]: /assets/img/aidl-29.0.3.jpeg
[3]: /assets/img/kotlin-compile-tag.png
[4]: /assets/img/kotlin-compile-java-source.png
