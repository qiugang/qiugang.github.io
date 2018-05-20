#### previously

18'IO 上 google 介绍了一系列的 architecture 组件，在油管上都有与之对应的 session。众多组件中， WorkManager 是我最喜欢的组件之一，接下来就谈谈 WorkManager 能给我们的开发带来什么。

16年的时候，我写过一篇主题为： [Offline App Practice](http://qiugang.github.io/offline-app-practice) 的文章，里面大致的描述了 [Diigo](https://www.diigo.com) 在实现 offline architecture 的相关思路与想法，当时的实现，主要是参照了 @yigit 的分享，并采用了他所维护的 [andorid-priority-jobqueue](https://github.com/yigit/android-priority-jobqueue) 作为 offline architecture 的核心实现。现在看来，WorkManager 是 andorid-priority-jobqueue 的升级版本：更丰富的功能，更友好的 api。

#### what it's?

WorkManager 是用来创建 **延迟**, **异步** 任务的脚手架。它不仅可以指定任务执行时机，周期而且即便是应用退出，手机重启，添加的任务依然能够不丢失，保持执行！

#### why it's?

让我们来看看 doc 中简单的例子：比如一个相册应用，并且带有 cloud backup 功能。 使用流程为：用户拍照 -> 存储照片 -> 压缩 -> 上传
让我们着重看下 压缩 -> 上传 这个业务需求

1. 两个 task 是与界面无关的异步逻辑，不应阻塞用户行为
2. 两个 task 之前有前置条件依赖，执行顺序需要维护
3. 失败处理逻辑

若自己实现上面逻辑的话，我相信还是要费一番心思，最为主要的是细节不是太好把控，代码实现出来的逻辑不一定是好维护与扩展的（比如压缩与上传中间还要加一个逻辑业务点）。而 WorkManager 就可以很清晰，容易的实现上诉逻辑：

CompressWorker

```
public class CompressWorker extends Worker {
    @Override
    public Worker.WorkerResult doWork() {

        // Do the work here--in this case, compress the stored images.
        // In this example no parameters are passed; the task is
        // assumed to be "compress the whole library."
        myCompress();

        // Indicate success or failure with your return value:
        return WorkerResult.SUCCESS;

        // (Returning RETRY tells WorkManager to try this task again
        // later; FAILURE says not to try again.)
    }
}
```

UploadWorker

```
public class UploadWorker extends Worker {
    @Override
    public Worker.WorkerResult doWork() {

        myUploadLogic();

        return WorkerResult.SUCCESS;

   }
```
Execute

```
OneTimeWorkRequest mCompressWork = new OneTimeWorkRequest.Builder(CompressWorker.class).build();
//...

WorkManager.getInstance()
    .begin(mCompressWork)
    .then(mUploadWork)
    .enqueue();
```

it's done! 使用 WorkManager api 主要定义两种 work ，然后指定一下执行顺序关系，上面略显复杂的业务逻辑就能被很好的拆分实现，并且还有丰富的拓展性：

* Task constraints
  对于上面的例子中，我们认为上面的流程中 compress 是比较耗资源的task，那我们可以使用 task constraints 来解决这个问题：
  
  ```
  Constraints myConstraints = new Constraints.Builder()
    .setRequiresDeviceIdle(true)
    .setRequiresCharging(true)
    // Many other constraints are available, see the
    // Constraints.Builder reference
     .build();
     
  // ...then create a OneTimeWorkRequest that uses those constraints
  OneTimeWorkRequest compressionWork =
                new OneTimeWorkRequest.Builder(CompressWorker.class)
     .setConstraints(myConstraints)
     .build();
  ```
内置的约束有:

   * required_network_type
	* requires_charging
	* requires_device_idle
	* requires_battery_not_low
	* requires_storage_not_low
	* content_uri_triggers

 使用 constraint 可以让我们复杂的业务逻辑得以顺利执行的同时，保证应用性能，提升用户体验。
  
* Recurring tasks
   WorkManager 执行的粒度 Unit 是一个 WorkerRequest，其中又分为两类: OneTimeWorkRequest 与 PeriodicWorkRequest。
   OneTimeWorkRequest 很好理解，就是执行一次，执行完了就没了。 PeriodicWorkRequest 字面意思是可重复的，意思利用它可以做到周期定时 request 的效果，什么意思呢，还是拿上面的例子，我们并不想每次用户拍照后都去手动触发整个逻辑，我们希望这里的流程是周期性的，检查到用户有新的照片，就开始走压缩-> 上传处理流程。PeriodicWorkRequest 为此而生：
   
   ```
   new PeriodicWorkRequest.Builder photoWorkBuilder =
        new PeriodicWorkRequest.Builder(PhotoCheckWorker.class, 12,
                TimeUnit.HOURS);
	// ...if you want, you can apply constraints to the builder here...
	
	// Create the actual work object:
	PeriodicWorkRequest photoWork = photoWorkBuilder.build();
	// Then enqueue the recurring task:
	WorkManager.getInstance().enqueue(invWork);
   ```
 你可以很方便的设置周期时间，自动化一些特殊场景的 task

* Canceling a Task
	每个 WorkRequest 会自动生成一个 unique id，根据 unique id 我们可以尝试取消放入队列中的 task 或者查询该 task 的状态信息（配合 LiveData 堪称完美）。除了 unique id， 还可以执行 WorkRequest 的时候指定 tag，根据 tag 来尝试取消。
   需要说明的是，我上面提及的取消，都是尝试取消。因为取消操作调用的时候，task 或者已经开始执行或者执行完了，但是 WorkManager 会 ```makes it's best effort to cancel```, 官方 doc 的这句话好萌 : ), 已经很努力啦！
* Chained tasks
  上面 demo 中执行task 时使用了 ```begin``` ,  ```then``` 来指定任务关系链，WorkManager 提供了非常 nice 的组织 task 关系链 api，可以满足非常复杂的关系链场景。 整个 api 的定义和 AnimatorSet 定义有些类似，非常好上手和理解，这里就不再赘述啦，具体参见 [workmanager#chained](https://developer.android.com/topic/libraries/architecture/workmanager#chained)
  
  
#### how it works?
 
see  [How WorkManager works](http://qiugang.github.io/how-workmanager-works)

 

