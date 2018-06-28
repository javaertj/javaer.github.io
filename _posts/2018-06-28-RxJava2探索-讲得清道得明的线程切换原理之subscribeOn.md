---
layout:     post
title:      RxJava2探索-讲得清道得明的线程切换原理之subscribeOn
subtitle:   RxJava探索
date:       2018-06-28
author:     YANKEBIN
catalog: true
tags:
     - Android
     - RxJava2
     - 线程切换
     - 响应式编程
     - ReactiveX
---

# 前言

说起来有点丢人，上周去某公司面试，做足了什么像java内存模型、hashmap原理、设计模式、Android多线程、自定义View等等相关的知识准备，然而面试的时候，**前面几个一个没问！！！**自定义view考察了onmeasure和Mnuspace那几个mode以及touch事件传递等，我真想给自己一巴掌，居然把那几个mode给忘了，只记得两个还拼不出单词。。。然后就问了RxJava，虽然没有直接问线程切换原理，但是确实考察的就是线程切换的问题，我只能靠猜了，并且还是以RXJAVA2来问的，因为我的简历上写着**了解RXJAVA**，不是精通，不是精通，不是精通。结果可想而知。

当然，我并不是说面试官有什么不好的地方，只是恨自己学得还不够多不够深，不善于总结和回顾以前的技能，希望自己知耻而后勇吧。俗话说，吃一堑长一智，面试完回来之后，我就把他所问到的问题全部研究了一遍，希望以后不要再出现上一次的窘境了^_^。

>tips:以下所有关于源码的部分都是基于RxJava2.1.15版本

# 一、为什么要写这个东西

你们肯定觉得我是为了写博客装x用的，或者是为了把学到的知识记录下来，以便于以后查阅。我告诉你们，都不是!今天在这里写这篇文章的原因很简单：我看了很多图文并茂的文章讲述RXJAVA线程切换原理,可是呢，最后我竟然是自己翻源码才实实在在的理解了这个原理的。嘿嘿，对，我就是这个意思，他们写得不够好！今天，我就是要让更多的人很快速的就理解RXJAVA线程切换的原理。

装x完毕，开始一本正经吹牛逼了，先盗个图。

![](https://user-gold-cdn.xitu.io/2018/4/2/1628386bb887ea63?w=300&h=225&f=jpeg&s=15735)


# 二、切换线程的那些事儿

### 2.1 什么叫线程切换
从程序的角度举个列子，作为一只Android开发攻城狮，大家肯定都能猜到我要以什么列子切入啦，没错，thread+handler更新UI，原始点的方式就是new一个Thread来执行网络请求（子线程），然后通过一个在主线程的handler发送消息到主线程更新UI（主线程），数据就从子线程切换到了主线程。Android里还有一个经典的可以切换线程的东西就是AsyncTask了，这里就不做额外展开了。

换个通俗易懂的例子，我有一片玉米地，玉米成熟了，我请了小明来帮我掰玉米，小明掰完玉米后我让他帮我数一下一共有多少个，小明却说，我没读过书，不会数数，只会掰玉米，没办法，我就请了擅长数学的小红帮我数玉米，小红数完玉米后，我想让她帮我把玉米煮熟了拿来卖，可小红又说了，我只会数玉米，不会煮玉米，我又请了会煮玉米的小刚来帮我煮玉米，然后自己卖玉米了。至此，掰玉米的事情是小明负责的（thread1），数玉米的事情是小红做的（thread2），煮玉米的事情是小刚做的（thread3），他们彼此相互独立却又帮我完成了一整套事情并且效率实现了最大化。

### 2.2 线程切换有什么用

我们再来看个例子，Android系统告诉你，不允许网络耗时任务发生在主线程哦，好嘛，那我就new一个Thread来执行网络请求嘛，等了半天，数据请求回来了，我开开开心心的把请求到的数据拿去渲染UI，结果，Android系统又告诉你，子线程不能更新UI哦。。。WTF？？？逗我玩呢么？Android系统又说了，别着急，我给你个小拖车，你把你取到的东西放在小拖车里，小拖车会来给我的。大家不要喷我，我只是把Handler+Thread这种模式说得复杂了一些。。。

这个套路大家肯定早就聊熟于心，闭着眼睛都能写出来，但是，可能很多人像我一样，并没有深究其表示的意义。因为Android操作系统有自己的一些规则，我们不得不遵守这些规则，在这些规则的束缚下，线程切换就必不可少。

线程切换的作用，在掰玉米那个例子的末尾已经说了，到了程序这一层，它的意义就是：**让代码执行在你认为最适合的地方！何谓适合？正确、尽可能高效！**

# 三、RxJava2的线程切换

**如果还没有RxJava2基础的童鞋，请先去了解下RxJava2在来看下面的内容会更好哦**

请各位往这边走，我好推你们下去。。。

>[传送门《这可能是最好的RxJava 2.x 教程》](https://www.jianshu.com/p/0cd258eecf60)

ps：***如果你们不想去看，也没关系，随便整个工程，引入RxJava2依赖，跟着我的步伐，不停地反复的戳进源码，或许会有另一种意想不到的效果***。

### 3.1从一段简单的代码说起

接下来，我们先看个RxJava2的subscribeOn方法切换线程的例子

1.不调用subscribeOn方法指定线程

		Observable.create((ObservableOnSubscribe<Integer>) observableEmitter -> {
		            System.out.println("ObservableOnSubscribe.subscribe thread : " + Thread.currentThread().getName());
		            observableEmitter.onNext(1);
		            observableEmitter.onComplete();
		        })//顶层Observable
		                .map(integer -> {
		                    System.out.println("map  thread : " + Thread.currentThread().getName());
		                    return String.valueOf(integer);
		                })//第二个Observable
		                .filter(s -> {
		                    System.out.println("filter  thread : " + Thread.currentThread().getName());
		                    return Integer.parseInt(s) > 0;
		                })//第三个Observable
		                .subscribe(new Observer<String>() {
		                    @Override
		                    public void onSubscribe(Disposable disposable) {
		                        System.out.println("onSubscribe  thread : " + Thread.currentThread().getName());
		                    }
		
		                    @Override
		                    public void onNext(String s) {
		                        System.out.println("onNext  thread : " + Thread.currentThread().getName());
		                    }
		
		                    @Override
		                    public void onError(Throwable throwable) {
		                        System.out.println("onError  thread : " + Thread.currentThread().getName());
		                    }
		
		                    @Override
		                    public void onComplete() {
		                        System.out.println("onComplete  thread : " + Thread.currentThread().getName());
		                    }
		                });


此时的输出结果如下：

	onSubscribe  thread : main
	ObservableOnSubscribe.subscribe thread : main
	map  thread : main
	filter  thread : main
	onNext  thread : main
	onComplete  thread : main

2.调用一次subscribeOn方法指定线程

		  Observable.create((ObservableOnSubscribe<Integer>) observableEmitter -> {
	            System.out.println("ObservableOnSubscribe.subscribe thread : " + Thread.currentThread().getName());
	            observableEmitter.onNext(1);
	            observableEmitter.onComplete();
	        })//顶层Observable
	               .subscribeOn(Schedulers.io())//第一次subscribeOn
	                .map(integer -> {
	                    System.out.println("map  thread : " + Thread.currentThread().getName());
	                    return String.valueOf(integer);
	                })//第二个Observable
	                .filter(s -> {
	                    System.out.println("filter  thread : " + Thread.currentThread().getName());
	                    return Integer.parseInt(s) > 0;
	                })//第三个Observable
	                .subscribe(new Observer<String>() {
	                    @Override
	                    public void onSubscribe(Disposable disposable) {
	                        System.out.println("onSubscribe  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onNext(String s) {
	                        System.out.println("onNext  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onError(Throwable throwable) {
	                        System.out.println("onError  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onComplete() {
	                        System.out.println("onComplete  thread : " + Thread.currentThread().getName());
	                    }
	                });
	                
此时的输出结果如下：

	onSubscribe  thread : main
	ObservableOnSubscribe.subscribe thread : RxCachedThreadScheduler-1
	map  thread : RxCachedThreadScheduler-1
	filter  thread : RxCachedThreadScheduler-1
	onNext  thread : RxCachedThreadScheduler-1
	onComplete  thread : RxCachedThreadScheduler-1	                
3.调用两次subscribeOn方法指定线程


		Observable.create((ObservableOnSubscribe<Integer>) observableEmitter -> {
       System.out.println("ObservableOnSubscribe.subscribe thread : " + Thread.currentThread().getName());
                observableEmitter.onNext(1);
                observableEmitter.onComplete();
        })//顶层Observable
                .subscribeOn(Schedulers.io())//第一次subscribeOn
                .map(integer -> {
                    System.out.println("map  thread : " + Thread.currentThread().getName());
                    return String.valueOf(integer);
                })//第二个Observable
                .subscribeOn(Schedulers.newThread())//第二次subscribeOn
                .filter(s -> {
                    System.out.println("filter  thread : " + Thread.currentThread().getName());
                    return Integer.parseInt(s) > 0;
                })//第三个Observable
                .subscribe(new Observer<String>() {
                    @Override
                    public void onSubscribe(Disposable disposable) {
                        System.out.println("filter  thread : " + Thread.currentThread().getName());
                    }

                    @Override
                    public void onNext(String s) {
                        System.out.println("onNext  thread : " + Thread.currentThread().getName());
                    }

                    @Override
                    public void onError(Throwable throwable) {
                        System.out.println("onError  thread : " + Thread.currentThread().getName());
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("onComplete  thread : " + Thread.currentThread().getName());
                    }
                });
                
                
此时的输出结果如下：

	onSubscribe  thread : main
	ObservableOnSubscribe.subscribe thread : RxCachedThreadScheduler-1
	map  thread : RxCachedThreadScheduler-1
	filter  thread : RxCachedThreadScheduler-1
	onNext  thread : RxCachedThreadScheduler-1
	onComplete  thread : RxCachedThreadScheduler-1                
                

从log打印日志可以看到，调用subscribeOn方法后，在他之前的和在他之后的代码执行的线程都是subscribeOn指定的线程——RxCachedThreadScheduler-1（**onSubscribe方法除外，后面会说明原因**），并且是第一次调用subscribeOn方法指定的io线程，那么这里会先有一个结论：**在这个链式调用结构中，无论你调用多少次subscribeOn去指定Observable的工作线程，总是以第一次调用subscribeOn时指定的线程为Observable的工作线程**。关于这个结论，将是我们接下来讨论的重点。

现在把这个事情分为两个问题：

	 1.subscribeOn方法是如何做到线程切换的？
	 
	 2.为什么只有第一次调用subscribeOn方法指定的线程才有效?
 
 那么接下来，请带着问题跟我上车啦。
 
 ![](https://user-gold-cdn.xitu.io/2018/4/2/1628386bb8b9e3f6?w=300&h=300&f=jpeg&s=20580)
 
 
### 3.2 从RTFSC说起

搞Android的，甚至可以说是搞软件的，RTFSC (Read the fucking source code —— Linus)才是生活中最重要的。我们天天就是要读懂别人的，理解别人的，然后再模仿别人的，最后才是创新自己的。人生大半的时间是在学习，所以我们一定要RTFSC。

在开始看源码之前，我门先看看要讲的内容会涉及到的类的类图：

![](https://ykbjson.github.io/blogimage/rxjava2-1/Observable.sunscribeOn%E7%B1%BB%E5%9B%BE.png)

可能有不对的方，还望大家斧正，感激不尽。

**important：这个类图一定要有个大概印象，不然后面你就会把页面滚上来滚下去的，不方便思维连续，或者你可以把这个图片存下来放到一边，以便随时看一眼，后面的内容就很好理解了。**

关于Runnable接口：这里写出来是为了让大家很清楚，Schedule类下面的Worker的schedule方法的参数就是需要一个Runnable，然鹅，ObservableSubscribeOn类里的SubscribeTask恰好又实现了Runnable接口，所以大家可以很容易的就猜到Schedule的scheduleDirect方法实际上最终就是会调用SubscribeTask的run方法。，我们先记住这个结论。

接下来，我们开始解决前面说到的第一个问题：subscribeOn方法是如何做到线程切换的？

### 3.3 subscribeOn方法切换线程的艺术

要看到线程切换，首先，我们先得有个Observable啊，所以我们先把Observable创建出来。

#### 3.3.1 Observable创建的过程

我们以上面只调用了一次subscribeOn方法的列子为基础，去掉filter和map操作，简化后如下
  
  	 Observable.create((ObservableOnSubscribe<String>) observableEmitter -> {
	            System.out.println("ObservableOnSubscribe.subscribe thread : " + Thread.currentThread().getName());
	            observableEmitter.onNext("1");
	            observableEmitter.onComplete();
	        })//顶层Observable
	               .subscribeOn(Schedulers.io())//第一次subscribeOn
	                .subscribe(new Observer<String>() {
	                    @Override
	                    public void onSubscribe(Disposable disposable) {
	                        System.out.println("onSubscribe  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onNext(String s) {
	                        System.out.println("onNext  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onError(Throwable throwable) {
	                        System.out.println("onError  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onComplete() {
	                        System.out.println("onComplete  thread : " + Thread.currentThread().getName());
	                    }
	                });

  
首先，我们先看Observable.create(ObservableOnSubscribe)方法

	@CheckReturnValue
    @SchedulerSupport("none")
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate(source));
    }

简化无关代码后，我们只关注

	new ObservableCreate(source)
	
这句代码，ObservableOnSubscribe使我们new的一个内部内。然后继续往下
	
		 public final class ObservableCreate<T> extends Observable<T> {
		   final ObservableOnSubscribe<T> source;
		
		    public ObservableCreate(ObservableOnSubscribe<T> source) {
		        this.source = source;
		    }
		    //...省略无关代码
		 }
	
这里没什么，就是把我们new的ObservableOnSubscribe传递给了ObservableCreate，这个时候，我们手里持有的对象就是一个Observable。

----> 其次，我们得调用一次subscribeOn方法

	 @CheckReturnValue
    @SchedulerSupport("custom")
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn(this, scheduler));
    }

简化无关代码后，我们只关注

	new ObservableSubscribeOn(this, scheduler)	
这句代码，schedule在这里是我们传递的一个Schedules.io，this的话当然是指的当前对象啦——刚才我们创建的一个Observable。接下来我们继续看

		public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
	    final Scheduler scheduler;
	
	    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
	        super(source);
	        this.scheduler = scheduler;
	    }
	   }

ObservableSource,如果还记得我刚才的类图，你应该知道，Observable实现了ObservableSource这个接口的，所以这里的source是指我们刚才创建的那个Observable。这里同时持有了Schedule的引用，这个后面会用到。

到了这里，请大家一定要很清楚一个问题，那就是**subscribeOn方法返回了一个新的Observable，而这个新的Observable里面持有一个上一层Observable的引用**。那个引用就是source。清楚了这个问题，我们磁能在后面一步豁然开朗。

----> 最后，我们得调用一次Observable的subscribe方法

	Observable.subscribe(new Observer<String>() {
	                    @Override
	                    public void onSubscribe(Disposable disposable) {
	                        System.out.println("onSubscribe  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onNext(String s) {
	                        System.out.println("onNext  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onError(Throwable throwable) {
	                        System.out.println("onError  thread : " + Thread.currentThread().getName());
	                    }
	
	                    @Override
	                    public void onComplete() {
	                        System.out.println("onComplete  thread : " + Thread.currentThread().getName());
	                    }
	                });


现在，大家能不能立马回答我，这里的subscribe方法是属于哪个Observable的？对，是属于subscribeOn方法返回的那个Observable的。明白了这一点，那么我们就要去看Observable的subscribe方法了。


#### 3.3.2 Observable订阅的过程

Observable.subscribe(Observer）

	
    @SchedulerSupport("none")
    public final void subscribe(Observer<? super T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");

        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);
            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");
            //真正订阅的地方
            this.subscribeActual(observer);
        } catch (NullPointerException var4) {
            throw var4;
        } catch (Throwable var5) {
            Exceptions.throwIfFatal(var5);
            RxJavaPlugins.onError(var5);
            NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
            npe.initCause(var5);
            throw npe;
        }
    }


简化无关代码后，我们只需关注subscribeActual方法，但是，还记得我类图的同学不知道有没有注意到，subscribeActual是一个抽象方法，所以，我们要去找他的实现类啦。

这里又要回到我提醒大家关注的问题，当前的Observable是谁？对，是属于subscribeOn方法返回的那个Observable。我们再次戳进
subscribeOn方法

	@CheckReturnValue
    @SchedulerSupport("custom")
    public final Observable<T> subscribeOn(Scheduler scheduler) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        return RxJavaPlugins.onAssembly(new ObservableSubscribeOn(this, scheduler));
    }

这里return的是一个ObservableSubscribeOn，不用猜，ObservableSubscribeOn肯定是继承了Observable的，我类图里已经画的很清楚了，只不过中间多了一层而已，那我们就直接看ObservableSubscribeOn的subscribeActual方法

	public void subscribeActual(Observer<? super T> s) {
        ObservableSubscribeOn.SubscribeOnObserver<T> parent = new ObservableSubscribeOn.SubscribeOnObserver(s);
        s.onSubscribe(parent);
        parent.setDisposable(this.scheduler.scheduleDirect(new ObservableSubscribeOn.SubscribeTask(parent)));
    }
    
    
重头戏来了，**s.onSubscribe(parent)**方法在这里居然就被调用了！！！s是谁？就是我们new的那个Observer。我前面说到过这句话：**调用subscribeOn方法后，在他之前的和在他之后的代码执行的线程都是subscribeOn指定的线程——RxCachedThreadScheduler-1（onSubscribe方法除外，后面会说明原因）**，这里就是原因。此时的Observable的subscribe方法发生在当前线程，所以Observer的onSubscribe方法的执行线程和当前调用Observable的subscribe方法的线程一致！！！

在这里，还有一件事情正悄悄酝酿着，那就是

	this.scheduler.scheduleDirect(new ObservableSubscribeOn.SubscribeTask(parent))
	
#### 3.3.3 切换线程的点点滴滴

请容我深呼一口气。。。

**important：因为这里其实要涉及到java.util.concurrent包下关于线程池的很多类，比如Executors、ScheduledExecutorService等等等等，关于这部分的知识我也没有去仔细研究过，所以在这里我不能做展开。大家只需要知道，这里使用这些工具类就是为了提供一个Thread来执行ObservableSubscribeOn.SubscribeTask**。我后面一定会去研究线程池相关的知识，然后再快来分享给大家。

通过前面的说明和查看类图大家可以知道，ObservableSubscribeOn.SubscribeTask是一个Runnable，他的run方法如下

	 public void run() {
            ObservableSubscribeOn.this.source.subscribe(this.parent);
        }

这里面的ObservableSubscribeOn.this.source是谁？就是我们创建的第一个Observable，ObservableCreate。那么，这个run方法会在哪里被调用呢？我们回顾到上一步我说的有件事情正在悄悄酝酿着那里,即ObservableSubscribeOn的subscribeActual方法里：

	this.scheduler.scheduleDirect(new ObservableSubscribeOn.SubscribeTask(parent))

这里的schedule在本列中是IoSchedule，在类图里也有的，他的scheduleDirect(Runnable)方法是在Schedule里实现的，只是把Worke的创建放到了IoSchedule（这是什么设计模式？？？）。Schedule的scheduleDirect(Runnable)方法最终调用了Schedule的scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit)方法。

**在这里请大家开始记住一个关键点，现在传递的Runnable就是ObservableSubscribeOn.SubscribeTask**

	@NonNull
    public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        Scheduler.Worker w = this.createWorker();
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        Scheduler.DisposeTask task = new Scheduler.DisposeTask(decoratedRun, w);
        w.schedule(task, delay, unit);
        return task;
    }

省略无关代码后，我们只关注w.schedule(task, delay, unit)方法。刚才说了，createWorker是子类实现的，所以我们直接戳进IoSchedule的createWorke方法

	@NonNull
    public Worker createWorker() {
        return new IoScheduler.EventLoopWorker((IoScheduler.CachedWorkerPool)this.pool.get());
    }
    
这里通过IoScheduler.CachedWorkerPool构造了一个IoScheduler.EventLoopWorker，IoScheduler.CachedWorkerPool提供了一个什么呢？一个IoScheduler.ThreadWorker，而IoScheduler.ThreadWorker继承至NewThreadWorker。这几个类的关系在类图里都有的，大家看一下就明白了。

那么很明显，我们接下来就要看的是IoScheduler.EventLoopWorker的schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit)方法

	@NonNull
        public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
            return (Disposable)(this.tasks.isDisposed() ? EmptyDisposable.INSTANCE : this.threadWorker.scheduleActual(action, delayTime, unit, this.tasks));
        }
		
this.threadWorker就是刚才我们说的IoScheduler.CachedWorkerPool提供给IoScheduler.EventLoopWorker的，为了怕大家不理解，我们还是看一下IoScheduler.EventLoopWorker的变量和构造方法

	static final class EventLoopWorker extends Worker {
        private final CompositeDisposable tasks;
        private final IoScheduler.CachedWorkerPool pool;
        private final IoScheduler.ThreadWorker threadWorker;
        final AtomicBoolean once = new AtomicBoolean();

        EventLoopWorker(IoScheduler.CachedWorkerPool pool) {
            this.pool = pool;
            this.tasks = new CompositeDisposable();
            this.threadWorker = pool.get();
        }
    }		
    
 到了这里，如果大家要追溯threadWorker的创建，就可以从 this.threadWorker = pool.get()这句代码入手啦，这后面就是关于线程池的内容了，我们暂且打住。。。
 
 现在我们继续查看IoScheduler.ThreadWorker的scheduleActual(action, delayTime, unit, this.tasks)方法，这个方法是由他的父类NewThreadWorker处理的
 
 	 @NonNull
    public ScheduledRunnable scheduleActual(Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);
        if (parent != null && !parent.add(sr)) {
            return sr;
        } else {
            try {
                Object f;
                if (delayTime <= 0L) {
                    f = this.executor.submit(sr);
                } else {
                    f = this.executor.schedule(sr, delayTime, unit);
                }

                sr.setFuture((Future)f);
            } catch (RejectedExecutionException var10) {
                if (parent != null) {
                    parent.remove(sr);
                }

                RxJavaPlugins.onError(var10);
            }

            return sr;
        }
    }
    
这里就只需要看his.executor.submit(sr)方法和this.executor.schedule(sr, delayTime, unit)方法，其实你戳进去看的话，最后都调用了ScheduledThreadPoolExecutor的schedule(Runnable command,long delay,TimeUnit unit)方法，因为NewThreadWorker的executor就是ScheduledExecutorService，也就是ScheduledThreadPoolExecutor 。

	public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService {
        }

总之呢，到这里的时候，虽然还封装了一个ScheduledRunnable ，不过最后ScheduledRunnable的run方法还是调用了scheduleActual方法传入的Runnable的run方法（这又是什么设计模式？？？）。现在就相当于

	new Thread（ObservableSubscribeOn.SubscribeTask).start();
	
迅速的回过去看一眼ObservableSubscribeOn.SubscribeTask的run方法

	 ObservableSubscribeOn.this.source.subscribe(this.parent);
	 
大声的告诉我，source是谁？是不是我们的ObservableCreate？？？他的subscribe方法会调用他的subscribeActual方法

	protected void subscribeActual(Observer<? super T> observer) {
        ObservableCreate.CreateEmitter<T> parent = new ObservableCreate.CreateEmitter(observer);
        observer.onSubscribe(parent);

        try {
            this.source.subscribe(parent);
        } catch (Throwable var4) {
            Exceptions.throwIfFatal(var4);
            parent.onError(var4);
        }

    }	 
    
  这里的this.source又是谁？？？是不是我们在Observable.create时传入的ObservableOnSubscribe？？？然后 this.source.subscribe(parent)不就是我们new的那个ObservableOnSubscribe的subscribe方法，我们在里面发送数据啦。
  
  	Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> observableEmitter) throws Exception {
                System.out.println("ObservableOnSubscribe.subscribe thread : " + Thread.currentThread().getName());
                observableEmitter.onNext(1);
                observableEmitter.onComplete();
            }
        })
     
     
是谁在发送呢？是不是ObservableCreate.CreateEmitter在发送数据？我们接着看看ObservableCreate.CreateEmitter的部分源代码

		static final class CreateEmitter<T> extends AtomicReference<Disposable> implements ObservableEmitter<T>, Disposable {
        private static final long serialVersionUID = -3434801548987643227L;
        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }

        public void onNext(T t) {
            if (t == null) {
                this.onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
            } else {
                if (!this.isDisposed()) {
                    this.observer.onNext(t);
                }

            }
        }

        public void onError(Throwable t) {
            if (!this.tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }

        }

     
        public void onComplete() {
            if (!this.isDisposed()) {
                try {
                    this.observer.onComplete();
                } finally {
                    this.dispose();
                }
            }

        }
		}
	    
它里面有一个Observer对象，这个对象哪里来的？就是subscribeOn方法返回的那个Observable——ObservableSubscribeOn里的ObservableSubscribeOn.SubscribeOnObserver，我们再看ObservableSubscribeOn的subscribeActual方法

	 public void subscribeActual(Observer<? super T> s) {
        ObservableSubscribeOn.SubscribeOnObserver<T> parent = new ObservableSubscribeOn.SubscribeOnObserver(s);
        s.onSubscribe(parent);
        parent.setDisposable(this.scheduler.scheduleDirect(new ObservableSubscribeOn.SubscribeTask(parent)));
    }	 
    
再看看ObservableSubscribeOn.SubscribeOnObserver的实现

	static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {
        private static final long serialVersionUID = 8094547886072529208L;
        final Observer<? super T> actual;
        final AtomicReference<Disposable> s;

        SubscribeOnObserver(Observer<? super T> actual) {
            this.actual = actual;
            this.s = new AtomicReference();
        }

        public void onSubscribe(Disposable s) {
            DisposableHelper.setOnce(this.s, s);
        }

        public void onNext(T t) {
            this.actual.onNext(t);
        }

        public void onError(Throwable t) {
            this.actual.onError(t);
        }

        public void onComplete() {
            this.actual.onComplete();
        }

      }       
    
 ObservableSubscribeOn.SubscribeOnObserver里面也有个Observer——actual，这个actual是我们在最外面new的那个Observer。那么这个时候就清楚了，如果ObservableCreate.CreateEmitter调用一次onNext方法，那么后续的执行逻辑就是
 
>ObservableCreate.CreateEmitter.onNext---> ObservableSubscribeOn.SubscribeOnObserver.onNext---->最外面new的那个Observer.onNext

那么onComplete和onError方法也是一样的，就不在赘述了。那么现在大家可以开始记住下面这个结论：**ObservableCreate.CreateEmitter.onNext方法是在ObservableSubscribeOn.SubscribeTask的run方法里被调用的，刚才也说了，经过各种高深复杂的方式把ObservableSubscribeOn.SubscribeTask放到了一个新的Thread（Schedules.io）里面去执行，那么从ObservableCreate.CreateEmitter.onNext方法开始，后续的执行逻辑就也都在一个新的Thread（我们指定的Schedules.io）里面去了，后续我们也没有再切换线程，所以后续代码的工作线程都是我们指定的Schedules.io**。至此，线程切换的原理就清楚了。

可能很多童鞋看到这里也如同我刚开始看别人的文章一样的感觉

![](https://upload-images.jianshu.io/upload_images/3994917-ea91d3468b5063d3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/198)
 
不过，如果你愿意顺着我的思路去多戳几下源码，顺便动动笔自己画点图，我相信你肯定可以比我描述的更好，更重要的是，你肯定能记得非常深刻。

好了，我们还有一个问题没有解决，请允许我再吹几分钟的牛x。。。我有个建议，大家可以先不往下看，先把这上面的逻辑理清楚了，后面部分你们根本不用看我的讲解了，真的，骗你是小狗，汪汪汪。。。

### 3.4 subscribeOn第一次有效原理

#### 3.4.1 来一段非常非常simple的代码

先定义一个打印类，它内部可以持有一个其他的打印类source，在他的打印方法里会开启新线程执行source的打印方法
	
	private static class Printer {
        Printer source;
        String name;
        Printer(Printer source, String name) {
            this.name=name;
            this.source = source;

        }
        void print() {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println(name +"-"+ Thread.currentThread().getName());
                    if (null != source) {
                        source.print();
                    }
                }
            }).start();
        }
    }
    
   
然后我们在main方法里执行如下代码
 
 	 public static void main(String args[]) {
        final Printer printer1 = new Printer(null,"printer1");
        final Printer printer2 = new Printer(printer1,"printer2");
        final Printer printer3 = new Printer(printer2,"printer3");
        final Printer printer4 = new Printer(printer3,"printer4");
        printer4.print();
    }
 	   

输出结果如下

	printer4-Thread-0
	printer3-Thread-1
	printer2-Thread-2
	printer1-Thread-3


如果你清楚了这里要表示的这个原理，你可能就已经猜到为什么subscribeOn方法只有第一次指定线程的地方是有效的，我就不用费口舌了，文章到此结束。。。不过，那是不可能的，说好了要再装几分钟的x，怎么可能这么快。产品经理快扶我一把，虽然手断了，但我还要撸代码。

言归正传，这个原理是什么？对于任意一个Printer而言，不管外面包裹了多少层新的线程去调用他的print方法，他的print方法里的执行语句的工作线程永远都是他的print方法里new的那个Thread。那么在这里，对于第一个printer而言，不管你们把我传递了多少层，最后你们调用我的print方法的时候，我print方法里的执行语句的工作线程只能是我new的那个Thread。

类似如下代码

	 new Thread(new Runnable() {
            @Override
            public void run() {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        new Thread(new Runnable() {
                            @Override
                            public void run() {
                                System.out.println(Thread.currentThread().getName());
                            }
                        },"thread3").start();
                    }
                },"thread2").start();
            }
        },"thread1").start();


Thread.currentThread().getName()的结果永远是thread3。清楚了这一点，我们就可以愉快的继续往下讲了。

#### 3.4.2 Observable多次subscribeOn的流程类比

我们用一个例子来解释。首先，定义一个IPrinter接口和一个IPaper接口

	interface IPrinter {
        void subscribe(IPaper paper);

        void preparePrint(IPaper paper);

        void print(IPaper paper);
    }
    
    
    interface IPaper {
        void show();
    }
 
 
 其次，定义一个Printer类来实现IPrinter接口
 
 	private static class Printer implements IPrinter {
        private IPrinter source;
        private String name;
        private static volatile AtomicInteger createCount = new AtomicInteger(0);

        Printer(IPrinter source,String name) {
            this.source = source;
            this.name=name;
        }

        @Override
        public void subscribe(IPaper paper) {
        	  //这个地方非常关键
            IPaper parent = new Paper(paper);
            preparePrint(parent);
        }

        @Override
        public void preparePrint(IPaper paper) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println("Printer-" + name + "-preparePrint-" +
                            Thread.currentThread().getName());
                    if (null != source) {
                        source.subscribe(paper);
                    } else {
                        print(paper);
                    }
                }
            }).start();
        }

        @Override
        public void print(IPaper paper) {
            System.out.println("Printer-" + name + "-startPrint-" +
                    Thread.currentThread().getName());
            paper.show();
        }
    }
    
 再定义一个Paper类实现IPaper接口
 
 	private static class Paper implements IPaper {
        private IPaper actual;
        private static volatile AtomicInteger createCount = new AtomicInteger(0);
        private int mId;

        Paper(IPaper actual) {
            this.actual = actual;
        }

        @Override
        public void show() {
            actual.show(content);
            mId=createCount.incrementAndGet();
            System.out.println("Paper-" + mId + "-show-" +
                    Thread.currentThread().getName());
        }
    }     

最后在main方法调用

	public static void main(String args[]) {
        final IPrinter printer1 = new Printer(null, "printer1");
        final IPrinter printer2 = new Printer(printer1, "printer2");
        final IPrinter printer3 = new Printer(printer2, "printer3");
        final IPrinter printer4 = new Printer(printer3, "printer4");
        printer4.subscribe(new IPaper() {
            @Override
            public void show() {

            }
        });
    }


输出结果如下

	Printer-printer4-preparePrint-Thread-0
	Printer-printer3-preparePrint-Thread-1
	Printer-printer2-preparePrint-Thread-2
	Printer-printer1-preparePrint-Thread-3
	Printer-printer1-startPrint-Thread-3
	Paper-1-show-Thread-3
	Paper-2-show-Thread-3
	Paper-3-show-Thread-3
	Paper-4-show-Thread-3

代码你们可以直接考出去运行的，看看我有没有说错。subscribeOn之所以只有第一次调用才有效，就是利用的类似这个demo展示的原理。顶层Observer发送数据的线程永远是第一次调用subscribeOn时指定的线程，因为数据的发射流程过程中，我们再也没有去切换过线程了，所以这其实很好理解的吧。

就下面这个段代码做个简单类比分析

	Observable.create((ObservableOnSubscribe<String>) observableEmitter -> {
           System.out.println("ObservableOnSubscribe.subscribe thread : " + Thread.currentThread().getName());
                observableEmitter.onNext("1");
                observableEmitter.onComplete();
        })//顶层Observable
                .subscribeOn(Schedulers.io())//第一次subscribeOn
                .subscribeOn(Schedulers.newThread())//第二次subscribeOn
                .subscribe(new Observer<String>() {
                    @Override
                    public void onSubscribe(Disposable disposable) {
                        System.out.println("filter  thread : " + Thread.currentThread().getName());
                    }

                    @Override
                    public void onNext(String s) {
                        System.out.println("onNext  thread : " + Thread.currentThread().getName());
                    }

                    @Override
                    public void onError(Throwable throwable) {
                        System.out.println("onError  thread : " + Thread.currentThread().getName());
                    }

                    @Override
                    public void onComplete() {
                        System.out.println("onComplete  thread : " + Thread.currentThread().getName());
                    }
                });

根据3.3节讲的subscribeOn切换线程的原理和3.4.1提到的那个原理以及上一个demo我们可以知道，一但一个Observable调用了一次subscribeOn方法后，后续无论你再调用多少次subscribeOn方法都无法在改变Observable发射数据的线程了。

我们简化来看，我们调用了两次subscribeOn方法——subscribeOn(Schedulers.io())、subscribeOn(Schedulers.newThread())，我们就类比成printer1（create）--->printer2（subscribeOn(Schedulers.io()))---->print3（subscribeOn(Schedulers.newThread()))就很好理解了。

# 四、结语

写了这么多，也不知道大家懂了没，虽然我开头写的很霸气，我要写的比别人好怎么怎么的，可是写到这里的时候，我才发现，这个东西自己理解起来感觉很简单，但是要描述清楚却是很难。不怕你们笑话，写到这里我足足用了三天时间，不停地戳源码、揣摩、举例、重写，深怕给大家讲错了，深怕大家不理解。结果就是这个样子了，我只能说，我尽力了^_^我还要继续撸点代码养家糊口。

其实大家明白三个点就可以了
1.Observable subscribe的时候是从下游往上游执行的，只是在这个过程中，使用subscribeOn去切指定了顶层Observable的发射线程；

2.顶层Observable发送数据的时候是从上游往下游的，在这个过程中，发射数据的线程已经被subscribeOn指定过了，这个过程本身不会主动去切换线程，所以数据发射和传递的所有工作线程都是同一个（onSubscribe除外，他也不属于发射流程）。

3.订阅和发射的过程，在思想设计上，是一个圆环，只是在这个圆环闭合的过程中，你可以把它的某些部分的构建过程交给不同的厂家去完成；在代码层面，它又是一根无限接近直线的一条线，仅此而已。

其实我应该把这个部分的知识分成三个部分来讲的，第一部分将创与订阅的流程，第二部分讲subscribeOn切换线程的原理，第三部分讲observerOn切换线程的原理，可是我并不想这样做，我只分了两部分，这是第一部分，还有下一部分就是讲ObservOn切换下游线程的原理。不要问我为什么这么分，原因非常非常的简单，因为我懒啊！！因为我懒啊！！因为我懒啊！！！

 
 ![](https://upload-images.jianshu.io/upload_images/3994917-f60bdfe3afcba051.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/220)
 
 
 
 
 
   