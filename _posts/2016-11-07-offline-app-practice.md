
Offline Android App 架构被提出其实已经有些时间了，市场上也有很多非常优秀的 offline 架构的 app ： Evernote，Pocket， Twitter ...哦，对了，还有 [Diigo](https://play.google.com/store/apps/details?id=com.diigo.android)（算不上优秀，但的确是 offline！逃..） 在具体的实践过程中， 有三个比较重要的点需要去细致的考虑：Local database， Background job， Sync 策略。
		
### Local database
本地数据库的设计是 offline 应用构建中最为重要的一环，设计表时一定要考虑好同步机制引入后带来的表之间的操作关联关系，其次是后续数据表的升级的情况。

* UI 的数据源只来自 database （针对有同步逻辑的数据， 没有 sync 这样逻辑的数据还是按照常规的 networking -> process -> UI 的逻辑）
* UI 交互产生 Action，需要对数据改动时：首先对 local database, offline file 操作，然后发出数据更改 event 通知 UI 刷新和向 job 队列添加 background job.
* 保证新创建数据的唯一性,这里会存在生成本地数据 id 的问题，使用当前时间戳，或者自己维护一个本地 id 的生成器都可以，只要保证唯一性就可以。建议生成最后都为负数，逻辑上清晰。

### Background job
Background job 目前有两个很好的解决方案，分别是 Evernote 出品的 [Android-Job](https://github.com/evernote/android-job) 和 大神 @yigit 维护的 [Android Priority Job Queue (Job Manager)](https://github.com/yigit/android-priority-jobqueue)。 @yigit 一直在向 Android 开发者推荐 offline 这种应用架构，在 Android Dev Summit 2015 有 [Android Application Architecture](https://www.youtube.com/watch?v=BlkJzgjzL0c) 非常精彩的 presentation，之后在 Google IO 2016 再次深入的演讲了这一架构 [Android application architecture: Get ready for the next billion!](https://www.youtube.com/watch?v=70WqJxymPr8)。
项目最终使用的是 Android Priority Job ，关于 Background job 的使用上，会有下面几个要注意到的地方：

* job 定义优先级
* 合理定义好 job失败 的重试策略。在 Android Priority Job 中可设置 重试次数，重试延迟时间等相关配置，要参考具体业务来合理设置。对于达到重试次数的 job 会被 job manager 放弃，此时本地需要有一个记录 failed job 的数据表，否则会造成数据与server不一致的问题。找取合适的时间，再次加入到 job manager 中去执行。
* 合理统一定义 event, 通知 UI 刷新。
* job 的可取消性。Android Priority Job 是没有办法取消正在执行的 job 的，hack 的方式是向 job 加一个执行标志，从而可以控制是否执行 job 的逻辑代码，表面上 job 是执行了，但是逻辑代码没有执行，可以认为是取消 了 job。
 
### Sync
对于 sync 策略和具体设计，需要和写 server api 的同事细致的商讨每一个接口逻辑。server api 的实现可参照下 dev-summit-architecture-demo repo 中 server 的实现，代码是用 ruby 写的。

1. 同步标志 Action id：
	不是太建议用时间戳作为同步标志。比较建议单独维护一个Action id 本地变量，对每一次的同步操作单独设置 Action id作为此次同步操作的标识，递增递减你开心就好，建议有线性规律，这样可以很方便判断数据的新旧性。本地需对本地数据增加 Action id 这个字段，用于判断 sync 从 server(sync 向 client 返回结果时要带上本次 sync 的 Action id) 回来的数据和本地数据间的新旧，merge 还是 update。
2. sync 触发时间：
	* UI交互触发：与UI设计相关，设计不当可能会导致 sync 请求频繁，造成资源浪费。
	* 定时触发（可选）。
	* 网络状态发生变化时触发（可选）。
	
上面只是简略的提出了一些构建 offline 应用时的 tips，坑远远不止文中提到的这些，但是代价是值得的，offline 架构的应用能够很好的弥补常规应用带来的用户体验上的问题，特别是网络不好的情况下那种。最后，用@yigit 大神的演讲的标题结尾：

	Get ready for the next billion!



