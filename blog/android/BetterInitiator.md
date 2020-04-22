android启动优化---BetterInitiator

  APP的启动速度对用户的体验，和留存有很大的影响，所以android的启动优化还是比较重要的，也是各厂android程序猿的绩效指标。

#### 1.对于启动优化，常规操作有以下几点：

		1.视觉上的优化：就是将启动APP后的白屏页给替换掉，让用户在视觉上感到快。
	
		2.使用异步线程：将一些耗时的操作放到工作线程中去。
	
		3.延迟加载：将一些不是立即需要的初始化可以延后执行。


​	

#### 2.存在的问题：

		1.在进行初始化时，各种第三方库的初始化之间有先后顺序，比如必须先初始化完网络库，才能初始化埋点上报。你可能会说，只要让这两个库的初始化放在同一个线程不就好了吗？确实可以这样做，但是不够优雅。
	
		2.某些初始化必须在特定阶段完成，比如网络库的初始化，必须在Application的onCreate方法中执行完，到了Splash页以后会使用网络库进行网络请求。



为了解决以上问题，BetterInitiator就诞生了。BetterInitiator可以用来进行启动优化，它将最小单元定义为一个Task，具体的拆分粒度要根据你具体的业务进行。比如我可以把网络的初始化放在一个单独的Task,把Push的初始化也单独放在一个Task，这样代码更清晰，优雅，也更符合单一职责原则。



#### 3.使用BetterInitiator

	github地址：https://github.com/121880399/betterInitiator
	
	第一步：在你项目的根目录下的build.gradle文件中添加如下

```css
allprojects {
		repositories {
			...
			maven { url 'https://jitpack.io' }
		}
	}
```

	第二步：添加依赖

```css
dependencies {
	        implementation 'com.github.121880399:betterInitiator:1.0.5'
	}
```

	第三步：创建Task
	
	继承Task类，实现run方法，在run方法中执行你要初始化的内容。

```java
/**
 * ETask依赖于CTask和DTask
 * 在主线程中运行
 * ETask中需要传入BTask中返回的参数
 * @作者 Zhouzhengyi
 * @创建日期 2019/7/31
 */
public class ETask extends Task {

    String data;

    public ETask(){
        EventBus.getDefault().register(this);
    }

    @Subscribe(sticky = true)
    public void onInitEvent(InitEvent event){
        data = event.getResult();
    }

    @Override
    public void run() {
        LogUtils.i("ETask is running");
        if(!TextUtils.isEmpty(data)) {
            LogUtils.i("input data:" + data);
        }
        SystemClock.sleep(1000);
        EventBus.getDefault().unregister(this);
    }

    @Override
    public List<Class<? extends Task>> dependsOn() {
        List<Class<? extends Task>> list = new ArrayList<>();
        list.add(CTask.class);
        list.add(DTask.class);
        return list;
    }

    @Override
    public boolean runOnMainThread() {
        return false;
    }

     @Override
    public int threadPoolType() {
        return ITask.IO;
    }

    @Override
    public boolean needWait() {
        return true;
    }

}

```

这是demo中其中一个Task，这个Task的执行依赖于前面BTask的返回值，所以这里用EventBus将参数传入，当然还有其他的办法。在dependsOn这个方法中，定义了依赖关系，ETask必须在CTask和DTask执行完毕以后才能执行。runOnMainThread指定当前任务是否在主线程中执行，threadPoolType指定当前是在IO型线程池中执行，这里可以选择是用IO线程池还是CPU线程池。needWait如果返回true，那么在完成这个Task之前，是不会往下执行的，所以这里有可能造成ANR噢，使用的时候需要注意。当然你可以通过setWaitOutTime方法来指定超时时长。

	第四步：将Task添加进调度器

```java
TaskDispatcher taskDispatcher = new TaskDispatcher();
taskDispatcher.init(this)
                .addTask(new ATask())
                .addTask(new BTask())
                .addTask(new CTask())
                .addTask(new DTask())
                .addTask(new ETask())
                .addTask(new FTask(FTaskCallback))
                .start();
```

	这样就会根据每个Task的依赖关系进行执行啦。在这里我给一张Demo中Task的关系图给大家。

![微信图片_20190806105658.png](https://upload-images.jianshu.io/upload_images/1870458-79edd3bc337118f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


	ATask是在主线程中执行，它不依赖于任何Task。BTask也是在主线程中执行，它必须在ATask之后执行。CTask和DTask是在工作线程中执行，他们必须在BTask执行完毕以后再执行。ETask是在工作线程中执行，它必须等CTask和DTask同时执行完毕后再执行，并且如果ETask没有执行完毕不会进入到MainActivity界面。FTask在IO线程中执行，它不依赖于任何Task。
	
	在这里的依赖关系只是为了模拟复杂的情况，在正常情况下不会出现子线程中的任务要等待主线程的情况，如果是这种情况，也没必要使用子线程了。

#### 4.原理

	介绍完使用以后，我们来说说原理。首先所有的Task加入到调度器以后，会区分哪些是有依赖的Task，依赖关系是怎么样的，然后会通过图的拓扑排序来重新定义存在依赖的Task的先后执行顺序，接着会区分哪些Task是在主线程中执行，哪些Task是在工作线程中执行，并交给相应的线程进行执行。
	
	这里讲一下拓扑排序的原理，首先拓扑排序会将所有的节点放进一张邻接表里面，这个邻接表有点类似于我们的Hashmap，只是在每个顶点表节点中加入了一个字段用于记录当前的入度。

```java
 public Vector<Integer> topologicalSort(){
        //用于记录每个节点的入度
        int indegree[] = new int[mVerticeCount];
        //初始化所有节点的入度
        for (int i = 0 ; i < mVerticeCount ; i++){
            ArrayList<Integer> temp = (ArrayList<Integer>) mAdj[i];
            for(int node : temp){
                indegree[node]++;
            }
        }

        //找出所有入度为0的节点，放入队列中
        Queue<Integer> queue = new LinkedList<>();
        for(int i = 0 ; i < mVerticeCount ;i++){
            if(indegree[i] == 0){
                queue.add(i);
            }
        }
        //记录节点数
        int count = 0;
        //从队列中取节点，并且遍历该节点之后的节点加入到Vector中
        //Vector是线程安全的，并且容量是动态扩展的
        Vector<Integer> order = new Vector<>();
        while(!queue.isEmpty()){
            //从队列中获取入度为0的节点
            int u = queue.poll();
            order.add(u);
            //遍历该节点的相邻节点
            for(int node : mAdj[u]){
                //节点的入度减一，如果发现入度为0，加入到队列中
                if(--indegree[node] == 0){
                    queue.add(node);
                }
            }
            count++;
        }
        //如果取出的节点数量不等于所有节点数量，说明存在环
        //原因是，有环的情况下，环内各节点的入度不能消减为0，所以count值只能环外的节点个数
        if(count != mVerticeCount){
            throw new IllegalStateException("Exists a cycle in the graph");
        }
        return order;
    }
```

上面的注释写的很详细了，首先会遍历得到所有节点入度。然后找出入度为0的节点，放入一个队列中。每次执行完入度为0的节点后，该节点的相邻节点的入度减1，如果减1后入度为0，则放入到队列中，然后只要队列不为空，就从队列中不断的获取入度为0的节点进行执行。如果获取节点的次数，跟节点的总次数不相等，那么说明出现了环。

	至于如何实现Task之间的等待，我们用到了CountDownLatch这个类，这个类具体的用法大家可以自行百度一下，这里就不做详细的介绍了。