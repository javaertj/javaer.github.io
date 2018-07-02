---
layout:     post
title:      RxJava2探索-讲得清道得明的线程切换原理之observeOn
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

在前一篇文章[《RxJava2探索-讲得清道得明的线程切换原理之subscribeOn》](https://ykbjson.github.io/2018/06/28/RxJava2%E6%8E%A2%E7%B4%A2-%E8%AE%B2%E5%BE%97%E6%B8%85%E9%81%93%E5%BE%97%E6%98%8E%E7%9A%84%E7%BA%BF%E7%A8%8B%E5%88%87%E6%8D%A2%E5%8E%9F%E7%90%86%E4%B9%8BsubscribeOn/)里我说了我会把RxJava线程切换相关的知识分成两部分来讲，如果大家没看过上一部分，我建议大家戳进去看一下，因为我这篇文章会基于里面的一些结论和原理来讲解。

在[《RxJava2探索-讲得清道得明的线程切换原理之subscribeOn》](https://ykbjson.github.io/2018/06/28/RxJava2%E6%8E%A2%E7%B4%A2-%E8%AE%B2%E5%BE%97%E6%B8%85%E9%81%93%E5%BE%97%E6%98%8E%E7%9A%84%E7%BA%BF%E7%A8%8B%E5%88%87%E6%8D%A2%E5%8E%9F%E7%90%86%E4%B9%8BsubscribeOn/)这篇文章里，我们知道了Observable订阅、发射的流程，以及sunscribeOn是如何切换线程的，以及为什么调用subscrieOn方法之前的Observable的工作线程，只能是在其后第一次调用subscrieOn方法指定的线程。知道了这些以后，我们就要正式开始今天的讲解（吹牛x）啦。

# 一、我认为一点就通的原理

如果大家理解了subscribeOn的原理，那么observeOn就非常好理解了。我们这样想，subscribeOn既然是在“从下往上”调用的过程中“做手脚”，即控制上层的执行线程，那么observeOn就是“从上往下”调用的过程中做手脚啦，只不过subscribeOn因为其特殊的设计导致其只有在第一次调用时才有效，而observeOn却可以多次切换其后代码的执行线程，仅此而已。

如果还是不太理解的话，那么我们先用一个简单的例子来开始吧，这个栗子和前面一篇文章里的大同小异。

![](https://upload-images.jianshu.io/upload_images/3994917-cd3c64eb6fa97663.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/200)

我们先定义一个基础的Observable类

	private static class Observable {
        Observable source;
        String name;

        Observable(Observable source, String name) {
            this.name = name;
            this.source = source;
        }

        void subscribe(Observer observer) {
            subscribeActual(observer);
        }

        void subscribeActual(Observer origin) {
            System.out.println(name + " subscribeActual on " +
                    Thread.currentThread().getName());
            final Observer observerWrapper = wrapperObserver(origin);
            if (null == source) {
                onNext(observerWrapper);
                return;
            }
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println(name + " create thread " +
                            Thread.currentThread().getName());
                    if (null != source) {
                        source.subscribe(observerWrapper);
                    }
                }
            }).start();
        }


        void onNext(Observer observer) {
            System.out.println(name + " onNext on " +
                    Thread.currentThread().getName());
            observer.onNext("哈哈哈哈哈哈哈");
        }


        Observer wrapperObserver(Observer origin) {
            return new ObserverNormal(origin, name);
        }
    }

代码很简单，主要看subscribeActual方法，如果source为空，即到达了顶层Observable，发么就执行onNext方法发射数据，否则就开启一个新线程，调用source的subscribe方法。我们可以通过wrapperObserver方法构造不同的Observer来展示observeOn是如何切换下游线程的。这里面是默认的实现，一个ObserverNormal对象，后面会讲到。

接下来看看Observable的另一种实现，ObservableObserveOn。

	private static class ObservableObserveOn extends Observable {
        ObservableObserveOn(Observable source, String name) {
            super(source, name);
        }

        @Override
        Observer wrapperObserver(Observer origin) {
            return new ObserverObserveOn(origin, name);
        }
    }

这个就很简单了，就是为了实现wrapperObserver方法，返回一个不同的Observer ，ObserverObserveOn。

接下来，我们来看看ObserverNormal和ObserverObserveOn有什么区别呢？ObserverNormal实现了Observer接口，ObserverObserveOn继承至ObserverNormal。

	 interface Observer {
        void onNext(String content);
    }

    private static class ObserverNormal implements Observer {
        Observer actual;
        String printerName;

        ObserverNormal(Observer actual, String printerName) {
            this.actual = actual;
            this.printerName = printerName;
        }

        @Override
        public void onNext(String content) {
            System.out.println(printerName + "'s observer onNext on " +
                    Thread.currentThread().getName());
            actual.onNext(content);
        }
    }

    private static class ObserverObserveOn extends ObserverNormal {

        public ObserverObserveOn(Observer actual, String printerName) {
            super(actual, printerName);
        }

        @Override
        public void onNext(String content) {
            System.out.println(printerName + "'s observer onNext on " +
                    Thread.currentThread().getName());
            new Thread(new Runnable() {
                @Override
                public void run() {
                    System.out.println(printerName + " create thread " +
                            Thread.currentThread().getName());
                    actual.onNext(content);
                }
            }).start();
        }
    }

通过代码可以看到，ObserverNormal和ObserverObserveOn唯一不同地方就是onNext方法。ObserverNormal的onNext方法没有新开启线程去执行actual的onNext方法，而ObserverObserveOn则是新开启了一个线程去调用actual的onNext方法。很多童鞋看到这里可能都已经明白observerOn切换线程的原理了。通过前一篇文章我们可以知道，Observable发射数据的流程是“从上往下”依次调用的，假定前面的Observer都是一个ObserverNormal（actual），当调用到倒数第二个Observer的onNext方法时，假定这个时候倒数第二个Observer是一个ObserverObserveOn，他之后还有一个ObserverNormal，那么最后这个ObserverNormal（ObserverObserveOn里面的actual）的onNext方法是不是会在ObserverObserveOn的onNext方法里被调用？我们在看看ObserverObserveOn的onNext方法，他在里面新开启了一个线程去执行actual（ObserverNormal）的onNext方法，那么ObserverNormal的onNext方法是不是就被“切换”了线程！！！

不信的话，我们在main方法里执行如下代码

	public static void main(String []args){
		 final Observable observable1 = new Observable(null, "observable1");
        final Observable observable2 = new Observable(observable1, "observable2");
        final Observable observable3 = new ObservableObserveOn(observable2, "observable3");
        final Observable observable4 = new Observable(observable3, "observable4");
        observable4.subscribe(new Observer() {
            @Override
            public void onNext(String content) {
                System.out.println(content + " show on " +
                        Thread.currentThread().getName());
            }
        });
	}

这段代码的你们可以直接考进随便一个java里执行，其输出结果如下

	observable4 subscribeActual on main
	observable4 create thread Thread-0
	observable3 subscribeActual on Thread-0
	observable3 create thread Thread-1
	observable2 subscribeActual on Thread-1
	observable2 create thread Thread-2
	observable1 subscribeActual on Thread-2
	observable1 onNext on Thread-2
	observable1's observer onNext on Thread-2
	observable2's observer onNext on Thread-2
	observable3's observer onNext on Thread-2
	observable3 create thread Thread-3
	observable4's observer onNext on Thread-3
	哈哈哈哈哈哈哈 show on Thread-3

到了这里，大家肯定要说，你废话了这么多，这都是你写的测试代码，万一RxJava不是这么个原理呢？那么，接下来，就让我们一起去RTFSC吧。

# 二、Just Read The Fucking Source Code

老规矩，先给大家附上一张鄙人画的“美图”

![](https://ykbjson.github.io/blogimage/rxjava2-2/rxjava2-observeOn%E7%B1%BB%E5%9B%BE.png)

丑是丑了点，将就着看吧，这是关于observeOn相关的部分类图，还是像前一篇文章里强调的一样，大家先熟悉一下再看后面的内容会更好。

![](https://upload-images.jianshu.io/upload_images/3994917-398ee3ef9fcacfbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/198)

那我们就从Observable.observeOn方法开始走起吧

	@CheckReturnValue
    @SchedulerSupport("custom")
    public final Observable<T> observeOn(Scheduler scheduler) {
        return this.observeOn(scheduler, false, bufferSize());
    }
    
    
多包了一层而已 继续往下

	@CheckReturnValue
    @SchedulerSupport("custom")
    public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        ObjectHelper.requireNonNull(scheduler, "scheduler is null");
        ObjectHelper.verifyPositive(bufferSize, "bufferSize");
        return RxJavaPlugins.onAssembly(new ObservableObserveOn(this, scheduler, delayError, bufferSize));
    }
    
省略无关代码后，我们只关心new ObservableObserveOn(this, scheduler, delayError, bufferSize)这句代码

	public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
	    final Scheduler scheduler;
	    final boolean delayError;
	    final int bufferSize;
	
	    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
	        super(source);
	        this.scheduler = scheduler;
	        this.delayError = delayError;
	        this.bufferSize = bufferSize;
	    }
	
	    protected void subscribeActual(Observer<? super T> observer) {
	        if (this.scheduler instanceof TrampolineScheduler) {
	            this.source.subscribe(observer);
	        } else {
	            Worker w = this.scheduler.createWorker();
	            this.source.subscribe(new ObservableObserveOn.ObserveOnObserver(observer, w, this.delayError, this.bufferSize));
	        }
	
	    }
	}        

AbstractObservableWithUpstream，大家应该有点印象吧，继承至Observable，所以Observable.observeOn返回的还是个Observable。

至于这后面的流程，我们都清楚啦，如果在observeOn之后调用subscribe（Observer）方法，那么会把Observer通过每个Observable的subscribe方法”从下往上“一层一层的传递，当传递到observeOn 方法生成的ObservableObserveOn这里时，我们知道，最后会调用ObservableObserveOn的subscribeActual方法

	protected void subscribeActual(Observer<? super T> observer) {
	        if (this.scheduler instanceof TrampolineScheduler) {
	            this.source.subscribe(observer);
	        } else {
	            Worker w = this.scheduler.createWorker();
	            this.source.subscribe(new ObservableObserveOn.ObserveOnObserver(observer, w, this.delayError, this.bufferSize));
	        }
	
	    }


我们知道，subscribeOn方法就是在这里做的”手脚“，那么observeOn会不会也是在这里做的”手脚“呢？我们继续看ObservableObserveOn.ObserveOnObserver。

	 static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T> implements Observer<T>, Runnable {
        private static final long serialVersionUID = 6576896619930983584L;
        final Observer<? super T> actual;
        final Worker worker;
        final boolean delayError;
        final int bufferSize;
        SimpleQueue<T> queue;
        Disposable s;
        Throwable error;
        volatile boolean done;
        volatile boolean cancelled;
        int sourceMode;
        boolean outputFused;

        ObserveOnObserver(Observer<? super T> actual, Worker worker, boolean delayError, int bufferSize) {
            this.actual = actual;
            this.worker = worker;
            this.delayError = delayError;
            this.bufferSize = bufferSize;
        }

        public void onSubscribe(Disposable s) {
            if (DisposableHelper.validate(this.s, s)) {
                this.s = s;
                if (s instanceof QueueDisposable) {
                    QueueDisposable<T> qd = (QueueDisposable)s;
                    int m = qd.requestFusion(7);
                    if (m == 1) {
                        this.sourceMode = m;
                        this.queue = qd;
                        this.done = true;
                        this.actual.onSubscribe(this);
                        this.schedule();
                        return;
                    }

                    if (m == 2) {
                        this.sourceMode = m;
                        this.queue = qd;
                        this.actual.onSubscribe(this);
                        return;
                    }
                }

                this.queue = new SpscLinkedArrayQueue(this.bufferSize);
                this.actual.onSubscribe(this);
            }

        }

        public void onNext(T t) {
            if (!this.done) {
                if (this.sourceMode != 2) {
                    this.queue.offer(t);
                }

                this.schedule();
            }
        }

        public void onError(Throwable t) {
            if (this.done) {
                RxJavaPlugins.onError(t);
            } else {
                this.error = t;
                this.done = true;
                this.schedule();
            }
        }

        public void onComplete() {
            if (!this.done) {
                this.done = true;
                this.schedule();
            }
        }
        //省略无关代码
	}

这个类有点长，我们先看它重要的几个方法。毫无疑问，肯定实现了Observer接口，因为他也要包装一下Observer，不然怎么”做手脚“呢。

通过前一篇文章我们知道，当Observable把Observer包装了多层后，最终到达顶层Observable后，会开始发射数据，再通过各种包装过的Observer的onNext、onComplete等方法”从上往下“一层一层的传递发射的数据，当往下传递遇到了我们的observeOn生成的Observer时，我们就要去看ObservableObserveOn的onNext等方法和普通的Observer的onNext方法有什么差别了。

从上面的代码可以看到，ObservableObserveOn的onNext、onComplete、onError方法里面都调用了schedule方法，而普通的Observer是没有的，那么我们就去看看这个schedule方法在搞什么鬼

	 void schedule() {
            if (this.getAndIncrement() == 0) {
                this.worker.schedule(this);
            }

    }
    
this.worker，还记得他么？在onbserveOn方法生成的ObservableObserveOn的subscribeActual方法里
 
 	protected void subscribeActual(Observer<? super T> observer) {
        if (this.scheduler instanceof TrampolineScheduler) {
            this.source.subscribe(observer);
        } else {
            Worker w = this.scheduler.createWorker();
            this.source.subscribe(new ObservableObserveOn.ObserveOnObserver(observer, w, this.delayError, this.bufferSize));
        }

    }
 
 这个worker是我们调用observeOn（Schedule）方法时传入的Schedule的Worker，比如IoSchedule的ThreadWorker，最终，worker会执行其
 
	 scheduleActual(Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent)
 
 方法
 
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

最后无非是用线程池提供一个线程，执行ObservableObserveOn.ObserveOnObserver的run方法，ObservableObserveOn.ObserveOnObserver是实现了Runnable接口的。那么我们直接看ObservableObserveOn.ObserveOnObserver的run方法

	 public void run() {
            if (this.outputFused) {
                this.drainFused();
            } else {
                this.drainNormal();
            }

    }

outputFused——google翻译为”输出熔断“，正常情况下，都没有发生输出熔断，那么我们直接看this.drainNormal()方法吧

	 void drainNormal() {
            int missed = 1;
            SimpleQueue<T> q = this.queue;
            Observer a = this.actual;

            do {
                if (this.checkTerminated(this.done, q.isEmpty(), a)) {
                    return;
                }

                while(true) {
                    boolean d = this.done;

                    Object v;
                    try {
                        v = q.poll();
                    } catch (Throwable var7) {
                        Exceptions.throwIfFatal(var7);
                        this.s.dispose();
                        q.clear();
                        a.onError(var7);
                        this.worker.dispose();
                        return;
                    }

                    boolean empty = v == null;
                    if (this.checkTerminated(d, empty, a)) {
                        return;
                    }

                    if (empty) {
                        missed = this.addAndGet(-missed);
                        break;
                    }

                    a.onNext(v);
                }
            } while(missed != 0);

     }


各位撸友，这个方法不算难理解吧，校验当前状态是否适合继续下发数据，如果适合，从队列里取出数据，往下一个Observer继续传递，只不过这个时候，下一个Observero的onNext ——（ 看代码里a.onNext(v)）等方法的执行线程就是ObservableObserveOn.ObserveOnObserver的worker提供的了！整个过程就是这么简单。


# 三、一点点感悟

不得不说，ReactiveX（Reactive Extensions）这种编程模型真的非常非常棒啊，尤其是对我们这种搞Android的攻城狮非常有帮助，我们遇到异步数据流的地方实在是太多太多啦，按照以前的回调方式去处理异步数据流，在逻辑连贯性上是非常的不好做的，代码也不易于维护。如今我们有了RxJava以及衍生出来的RxAndroid、RxBiding等等，都能很好的帮我们处理异步数据流啦，并且还包括界面上常用的Click、touch、onTextChanged等事件数据的传递。ReactiveX的实用性，用过Retrofit的童鞋肯定都深有体会吧！

RxJava2有很多操作符，这里讲道德observeOn和subscribeOn只是其辅助类操作符里的两个而已，他还有创建、变换、过滤、结合等多种操作符集合，每种操作符集合下面还有很多具体的操作符，详细的内容，大家可以去看看gitbook，这里就不在赘述了。

>[《ReactiveX文档中文翻译》](https://mcxiaoke.gitbooks.io/rxdocs/content/Intro.html)

如果看翻译比较晦涩，大家可以去看看这位大神的系列文章

>[《这可能是最好的RxJava 2.x 教程》](https://www.jianshu.com/p/0cd258eecf60)



   