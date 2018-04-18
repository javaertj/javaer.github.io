---
layout:     post
title:      MVP模式探索——Presenter和View解耦的尝试
subtitle:   MVP模式探索4
date:       2018-04-17
author:     YANKEBIN
catalog: true
tags:
     - Android
     - MVP
     - MMVP
     - 设计模式
---

# 前言

关于MVP模式系列的文章，前面已经写了3篇了，本来想等以后对MVP模式有了更深层次的理解后再来总结一下的，但是最近在研究Adapter和Activity或Fragment解耦的时候，突然想到了View和Presenter之间的解耦，索性就尝试了一下，然后来和大家分享一下。

**郑重声明：**

_由于我对架构的经验不足，所以在这里只在MVP框架体系下讨论Presenter和View之间的关系解耦（也可能只能称得上是弱化了他们之间的联系），如果修改到了框架本身的属性，请大家忽略，或者大家提供一个在不改变MVP模式本身的前提下解耦Presenter和View的方式，感激不尽。_

# 一、原理简介

我们先看一下基本的MVP的原理图


![](https://upload-images.jianshu.io/upload_images/1233754-eb5b4bc4fbf757be.png!web?imageMogr2/auto-orient/strip%7CimageView2/2/w/550)

在这种模式下，Presenter不可避免的需要持有View的引用，同时View也不可避免的需要持有Presenter，当然，这是MVP模式本身的一个特性，一个Model和一个Presenter还有一个View构成了MVP模式的最小单元。但是在工作中的业务不可能总是能局限于某种模式，所以我们需要变化，需要更适合业务的模式。

假如我们思考这样一个问题，我们有一个鱼塘用来养鱼，Presenter里面有个叫（捕鱼）fishing的方法提供给View，后来由于市场原因，鱼不好卖了，我们开始在鱼塘里养虾，那么，捕鱼这个方法要改名字，要改成（捕虾）shrimp。大家一想，这个很简单啊，用编辑器的ReName一下就搞定了。但是这个改动的地方可不只是Presenter，还有View，因为View以前是调用Presenter的fishing方法，现在要改成调用Presenter的shrimp方法了，代码肯定会发生变化。

那么有没有一种办法，让View和Presenter互相不关注他们之间联系的方法的名字，而只关心对方提供的功能呢？答案是肯定的，那就是一定有这样的方法，并且不止一种，那么我们今天要谈的就是其中的一种。先看一下原理图

![](https://ykbjson.github.io/blogimage/mvppicture4/mmvp.png)

我们不妨让一个中间件来集中管理View和Presenter的关系，在View层用一个注解告诉中间件我要绑定哪个或哪几个Presenter，View和Presenter提供的每一个业务方法都用一个唯一标识来表示，Presenter和View之间某个方法的调用就是通过中间件发送一个包含某个方法持有的特定标识的Action来找到某个方法，而方法的调用是基于注解提供的标识，所以可以不关心彼此提供的方法的名称。这似乎很难以理解，也很难想象，那么，我就换种方式阐述一下。

我们把这个中间件看成一个Presenter和View的功能映射关系提供者即可，这个这个中间件提供一个注册View的方法，然后根据View的需求（即View上的注解）去生成对应的Presneter，并把他们的关系存储起来，然后提供一个可以发送一个特定Action的方法，这个Action里面有一个发送者的class、接收者class还有一个code和一些参数，这个code应着接收者里某个方法注解上的code，当中间件通过刚才存储的关系，找到发送者的class和接收者class的关系后，然后根据Action里面的code决定执行接收者的哪个方法。这所有的逻辑都在中间件里面完成，无需外部实现，View和Presenter的关系就变得更模糊了。

如果说文字阐述不清楚这个原理，那么就只能上代码了，代码是最好的老师...(真的吗？？？)

# 二、代码实现

库代码结构图

![](https://ykbjson.github.io/blogimage/mvppicture4/mmvpjiegou.png)


接下来，我们一个一个的来为大家解释每一个类的定义与作用，我们先从那两个注解开始吧。

### BindPresenter，顾名思义，就是绑定Presneter

	/**
	 * 包名：com.ykbjson.lib.mmvp.internal
	 * 描述：View绑定与Presenter的注解
	 * 创建者：yankebin
	 * 日期：2018/4/12
	 */
	@Target(ElementType.TYPE)
	@Retention(RetentionPolicy.RUNTIME)
	public @interface BindPresenter {
	    /**
	     * View需要绑定的Presenter数组
	     */
	    Class<? extends MMVPPresenter>[] value();
	}


这里为什么是一个数组呢？因为严格意义上来讲，MVP模式下，Presenter和View是一对一的关系，但是我在想，既然View和Presenter的关系已经变得模糊了，那么我是否可以把Presenter当成一种或一类功能的提供者，供不同的View绑定和调用。但是这样似乎又违背了MVP模式的定义，所以先暂且声明成一个数组吧。

### ActionProcess，这个可能不太好理解，因为我也没有想好特别能解释它作用的名字

	/**
	 * 包名：com.ykbjson.lib.mmvp.internal
	 * 描述：Presenter或View提供的可供反射调用的方法的注解
	 * 创建者：yankebin
	 * 日期：2018/4/12
	 */
	@Target(ElementType.METHOD)
	@Retention(RetentionPolicy.RUNTIME)
	public @interface ActionProcess {
	    /**
	     * action
	     **/
	    String value() default "";
	
	    /**
	     * 是否需要{@link com.ykbjson.lib.mmvp.MMVPAction}参数
	     **/
	    boolean needActionParam() default false;
	
	    /**
	     * 是否需要转换{@link com.ykbjson.lib.mmvp.MMVPAction}，即交换sourceClass和targetClass
	     **/
	    boolean needTransformAction() default false;
	
	    /**
	     * 是否需要{@link com.ykbjson.lib.mmvp.MMVPAction}里面的param参数
	     **/
	    boolean needActionParams() default false;
	
	}
	
	
其实这就是刚才我所说的“**Presenter和View之间某个方法的调用就是通过中间件发送一个包含某个方法持有的特定标识的Action来找到某个方法，而方法的调用是基于注解提供的标识**”，那个注解就是ActionProcess，里面的value对应我所说的code（在代码设计的时候我还是命名成了action），其他的都有详细的注释，后面Demo里也会用到，这里我就不在赘述了。总之，这个注解就是注册于方法之上，告诉中间件，我注册了某个action，需要些什么参数，然后你就可以调用我了。

既然都说到了Action，那我们就继续看看他的内部实现。

### MMVPAction，很明显，这是一个动作或者叫做行为
	
	/**
	 * 包名：com.ykbjson.lib.mmvp
	 * 描述：View和Presenter之间的通信携带
	 * 创建者：yankebin
	 * 日期：2018/4/12
	 */
	public class MMVPAction implements Serializable, Cloneable {
	    private Class<?> sourceClass;
	    private Class<?> targetClass;
	    private MMVPActionDescription action;
	    private Map<String, Object> params;
	    private IMMVPOnDataCallback onDataCallback;
	
	    MMVPAction(Class<?> sourceClass, Class<?> targetClass) {
	        this.sourceClass = sourceClass;
	        this.targetClass = targetClass;
	    }
	
	    public MMVPAction setSourceClass(Class<?> sourceClass) {
	        this.sourceClass = sourceClass;
	        return this;
	    }
	
	    public MMVPAction setTargetClass(Class<?> targetClass) {
	        this.targetClass = targetClass;
	        return this;
	    }
	
	    public MMVPAction setAction(MMVPActionDescription action) {
	        this.action = action;
	        return this;
	    }
	
	    public MMVPAction setParams(Map<String, Object> params) {
	        this.params = params;
	        return this;
	    }
	
	    public MMVPAction setOnDataCallback(IMMVPOnDataCallback onDataCallback) {
	        this.onDataCallback = onDataCallback;
	        return this;
	    }
	
	
	    public Class<?> getSourceClass() {
	        return sourceClass;
	    }
	
	
	    public Class<?> getTargetClass() {
	        return targetClass;
	    }
	
	
	    public MMVPActionDescription getAction() {
	        return action;
	    }
	
	
	    public Map<String, Object> getParams() {
	        return params;
	    }
	
	    public IMMVPOnDataCallback getOnDataCallback() {
	        return onDataCallback;
	    }
	
	    public void send() {
	        send(0);
	    }
	
	    public void send(long delayMills) {
	        MMVPArtist.sendAction(this, delayMills);
	    }
	
	    public MMVPAction transform() {
	        Class<?> exchange = getTargetClass();
	        setTargetClass(getSourceClass());
	        setSourceClass(exchange);
	        return this;
	    }
	
	    public MMVPAction clearParam() {
	        if (null != params && !params.isEmpty()) {
	            params.clear();
	        }
	
	        return this;
	    }
	
	    public MMVPAction putParam(String key, Object value) {
	        if (null == params) {
	            params = new HashMap<>();
	        }
	        params.put(key, value);
	        return this;
	    }
	
	    public <T> T getParam(String key) {
	        if (null == params || params.isEmpty()) {
	            return (T) null;
	        }
	        return (T) params.get(key);
	    }
	
	    void recycle() {
	        sourceClass = null;
	        targetClass = null;
	        action = null;
	        params = null;
	        onDataCallback = null;
	    }
	}


其实我知道，如果把它设计成一个接口，那么扩展性会好一些，但是，我还没有想好该如何去把它抽的很完美，所以就留给大家发挥啦。这个Action很重要，它包含了一次行为或动作里，发起者是谁，接收者是谁，需要什么参数。至于MMVPActionDescription，这是对Action实际需要传达的动作的封装，本来就是一个字符串，但是我在想，万一一个字符串不够表达一个行为该怎么办呢？所以就设计成了一个对象，里面就俩字段

	/**
	 * 包名：com.ykbjson.lib.mmvp
	 * 描述：View和Presenter之间的通信携带描述
	 * 创建者：yankebin
	 * 日期：2018/4/13
	 */
	public class MMVPActionDescription implements Serializable {
	    private String action;
	    private String code;
	
	    public String getAction() {
	        return action;
	    }
	
	    public void setAction(String action) {
	        this.action = action;
	    }
	
	    public String getCode() {
	        return code;
	    }
	
	    public void setCode(String code) {
	        this.code = code;
	    }
	}

是不是非常简洁，根本没有必要设计成对象。

至于IMMVPOnDataCallback，我是这样想的。因为后面大家会看到中间件执行这些方法的时候，都是通过反射执行的，如果不想用反射执行，也不想用这个Action注解方法，而仅仅是需要通过一个Callback把数据返回给我就可以了，那么，这个IMMVPOnDataCallback就有了用处。比如，我在View里发送了一个Action，Action里带着一个IMMVPOnDataCallback，当中间件解析完这个Action后，执行Presenter的方法，Presenter可以根据收到的Action里是否有IMMVPOnDataCallback，如果有，就不用在回传Action，而是直接用这个Callback回传请求到的数据，这样，至少就了一次反射执行View里的方法。

	/**
	 * 包名：com.ykbjson.lib.mmvp
	 * 描述：数据回调接口
	 * 创建者：yankebin
	 * 日期：2018/4/12
	 */
	public interface IMMVPOnDataCallback<T> extends Serializable{
	    void onSuccess(T data);
	
	    void onError(String msg);
	}


接下来我们来看一下IMMVPActionHandler

	/**
	 * 包名：com.ykbjson.lib.mmvp
	 * 描述：{@link MMVPAction}处理接口
	 * 创建者：yankebin
	 * 日期：2018/4/13
	 */
	public interface IMMVPActionHandler {
	
	    void handleAction(@NonNull MMVPAction action);
	
	    @NonNull
	    IMMVPActionHandler get();
	}

为什么会有这么个接口呢？我是这么想的。如果你不想在View或Presneter的方法上加上Action注解，那么也没有关系，我们的View和Presenter基类都是实现了这个接口的，你可以在View或Presenter的handleAction方法里调用任何你想调用的代码（我是不是很机智^_^）。

至于View和Presenter的代码，我都不想贴，又不得不贴

	/**
	 * 包名：com.ykbjson.lib.mmvp
	 * 描述：Presenter接口
	 * 创建者：yankebin
	 * 日期：2018/4/12
	 */
	public interface MMVPPresenter extends IMMVPActionHandler {
	
	}
你看看，Presneter空空如也。在看看View

	/**
	 * 包名：com.ykbjson.lib.mmvp
	 * 描述：View接口
	 * 创建者：yankebin
	 * 日期：2018/4/12
	 */
	public interface MMVPView extends IMMVPActionHandler {
	    String getScope();
	}
	
View这里就多了一个方法，关于这个方法，稍微给大家解释一下。最开始的时候，我所构思的解耦方式设这样的。View和Presenter属于一个作用域，只有作用域相同的View和Presenter才可以交互，可是我思来想去，这个笨拙的脑袋也没有想出什么好的方法去实现作用域这个概念，所以...请大家原谅我，以及这笨拙的脑袋。

接下来就是这重头戏了——MMVPArtist，这个类有点长，有多长呢？
	
	/**
	 * 包名：com.ykbjson.lib.mmvp
	 * 描述：View和Presenter层交互处理器
	 * 创建者：yankebin
	 * 日期：2018/4/12
	 */
	public final class MMVPArtist {
	    private static final String TAG = "MMVPArtist";
	
	    private static final int FLAG_HANDLE_ACTION = 100000;
	    /**
	     * 注册View缓存
	     */
	    private static final List<MMVPView> VIEW_CACHE = new LinkedList<>();
	    /**
	     * View和Presenter关系缓存
	     */
	    private static final Map<Class<?>, List<MMVPPresenter>> VIEW_PRESENTERS_CACHE = new LinkedHashMap<>();
	
	    /**
	     * 注册了{@link ActionProcess}注解的方法缓存
	     */
	    private static final Map<Class<?>, Map<String, Method>> METHODS_CACHE = new LinkedHashMap<>();
	
	    private static boolean enableLog = true;
	
	    @SuppressLint("HandlerLeak")
	    private static Handler dispatchActionHandler = new Handler() {
	        @Override
	        public void handleMessage(Message msg) {
	            super.handleMessage(msg);
	            switch (msg.what) {
	                case FLAG_HANDLE_ACTION:
	                    MMVPAction action = (MMVPAction) msg.obj;
	                    handleAction(action);
	                    break;
	            }
	        }
	    };
	
	    private MMVPArtist() {
	        throw new AssertionError("No instances.");
	    }
	
	    /**
	     * 日志开关
	     *
	     * @param enableLog
	     */
	    public static void setEnableLog(boolean enableLog) {
	        MMVPArtist.enableLog = enableLog;
	    }
	
	    /**
	     * 创建 {@link MMVPAction}
	     *
	     * @param sourceClass 创建Action的class
	     * @param targetClass 接收Action的class
	     * @param action      需要执行的操作
	     * @return {@link MMVPAction}
	     */
	    public static MMVPAction buildAction(Class<?> sourceClass, Class<?> targetClass, String action) {
	        MMVPActionDescription actionContent = new MMVPActionDescription();
	        actionContent.setAction(action);
	        return new MMVPAction(sourceClass, targetClass).setAction(actionContent);
	    }
	
	    /**
	     * 发送HVPAction
	     *
	     * @param action {@link MMVPAction}
	     */
	    static void sendAction(@NonNull MMVPAction action) {
	        sendAction(action, 0);
	    }
	
	    /**
	     * 发送HVPAction
	     *
	     * @param action     {@link MMVPAction}
	     * @param delayMills 延时毫秒数
	     */
	    static void sendAction(@NonNull MMVPAction action, long delayMills) {
	        if (null == action.getAction()
	                || TextUtils.isEmpty(action.getAction().getAction())
	                || null == action.getSourceClass()
	                || null == action.getTargetClass()) {
	            throw new IllegalArgumentException("Invalid MMVPAction");
	        }
	        Message message = dispatchActionHandler.obtainMessage(FLAG_HANDLE_ACTION, action);
	        dispatchActionHandler.sendMessageDelayed(message, delayMills);
	    }
	
	
	    /**
	     * 处理HVPAction
	     *
	     * @param action {@link MMVPAction}
	     */
	    private static void handleAction(@NonNull MMVPAction action) {
	        Class<?> sourceClass = action.getSourceClass();
	        if (MMVPPresenter.class.isAssignableFrom(sourceClass)) {
	            handleActionFromPresenter(action);
	        } else if (MMVPView.class.isAssignableFrom(sourceClass)) {
	            handleActionFromView(action);
	        } else {
	            throw new IllegalArgumentException("Invalid class type of the MMVPAction's targetClass and sourceClass");
	        }
	    }
	
	    /**
	     * 处理Presenter发送来的Action
	     *
	     * @param action {@link MMVPAction}
	     */
	    private static void handleActionFromPresenter(MMVPAction action) {
	        if (VIEW_CACHE.isEmpty()) {
	            if (enableLog) {
	                Log.d(TAG, " Can not find the MMVPAction's targetClass [ " +
	                        action.getTargetClass().getName() + " ],because the VIEW_CACHE is empty");
	            }
	            return;
	        }
	        IMMVPActionHandler find = null;
	        for (MMVPView hvpView : VIEW_CACHE) {
	            if (hvpView.getClass().equals(action.getTargetClass())) {
	                find = hvpView;
	                break;
	            }
	        }
	        if (null == find) {
	            if (enableLog) {
	                Log.w(TAG, " Can not find the MMVPAction's targetClass [ " +
	                        action.getTargetClass().getName() + " ] , it is not registered or has been destroyed ");
	            }
	            return;
	        }
	        if (!execute(action, find)) {
	            find.handleAction(action);
	        }
	    }
	
	    /**
	     * 处理View发送来的Action
	     *
	     * @param action {@link MMVPAction}
	     */
	    private static void handleActionFromView(MMVPAction action) {
	        if (!VIEW_PRESENTERS_CACHE.containsKey(action.getSourceClass())) {
	            if (enableLog) {
	                Log.w(TAG, " The MMVPAction's sourceClass [ " + action.getSourceClass().getName() +
	                        " ]  is not registered  or has been destroyed ");
	            }
	            return;
	        }
	        List<MMVPPresenter> hvpPresenterList = VIEW_PRESENTERS_CACHE.get(action.getSourceClass());
	        if (null == hvpPresenterList || hvpPresenterList.isEmpty()) {
	            if (enableLog) {
	                Log.w(TAG, "PresenterList is empty ,have you ever add annotation BindPresenter" +
	                        " for this view [ " + action.getSourceClass().getName() + " ] ?");
	            }
	            return;
	        }
	        IMMVPActionHandler find = null;
	        for (MMVPPresenter hvpPresenter : hvpPresenterList) {
	            if (action.getTargetClass().equals(hvpPresenter.getClass())) {
	                find = hvpPresenter;
	                break;
	            }
	        }
	
	        if (null == find) {
	            if (enableLog) {
	                Log.w(TAG, " Can not find the MMVPAction's targetClass [ "
	                        + action.getTargetClass().getName() + " ] ");
	            }
	            return;
	        }
	        if (!execute(action, find)) {
	            find.handleAction(action);
	        }
	    }
	
	    /**
	     * 执行action里目标类需要执行的方法
	     *
	     * @param action {@link MMVPAction}
	     * @param find   {@link MMVPView}或{@link MMVPPresenter}
	     * @return
	     */
	    private static boolean execute(MMVPAction action, IMMVPActionHandler find) {
	        Method executeMethod = findRegisterMMVPActionMethod(action);
	        if (null == executeMethod) {
	            if (enableLog) {
	                Log.d(TAG, " Find " + find.getClass().getName() + "'s execute method failure");
	            }
	            return false;
	        }
	        if (enableLog) {
	            Log.d(TAG, " Find  method " + find.getClass().getName() + "." + executeMethod.getName() + " success");
	        }
	
	        List<Object> paramList = new ArrayList<>();
	        ActionProcess methodAnnotation = executeMethod.getAnnotation(ActionProcess.class);
	        if (methodAnnotation.needActionParam()) {
	            if (methodAnnotation.needTransformAction()) {
	                action = action.transform();
	            }
	            paramList.add(action);
	        }
	        if (methodAnnotation.needActionParams() && null != action.getParams() && !action.getParams().isEmpty()) {
	            for (String key : action.getParams().keySet()) {
	                paramList.add(action.getParam(key));
	            }
	        }
	        Object[] params = paramList.isEmpty() ? null : paramList.toArray();
	        try {
	            executeMethod.setAccessible(true);
	            executeMethod.invoke(find, params);
	            if (enableLog) {
	                Log.d(TAG, " Execute "
	                        + find.getClass().getName() + "." + executeMethod.getName() + " success");
	            }
	            return true;
	        } catch (IllegalAccessException e) {
	            e.printStackTrace();
	            if (enableLog) {
	                Log.d(TAG, " Execute "
	                        + action.getTargetClass().getName() + "." + executeMethod.getName() + " failure", e);
	            }
	        } catch (InvocationTargetException e) {
	            e.printStackTrace();
	            if (enableLog) {
	                Log.d(TAG, " Execute "
	                        + action.getTargetClass().getName() + "." + executeMethod.getName() + " failure", e);
	            }
	        }
	
	        return false;
	    }
	
	    /**
	     * 注册View
	     *
	     * @param view {@link MMVPView}
	     */
	    @UiThread
	    public static void registerView(@NonNull MMVPView view) {
	        if (VIEW_CACHE.contains(view)) {
	            if (enableLog) {
	                Log.w(TAG, " ReRegister [ " + view.getClass().getName() + " ]");
	            }
	            return;
	        }
	        BindPresenter annotation = view.getClass().getAnnotation(BindPresenter.class);
	        if (null == annotation) {
	            throw new IllegalArgumentException("Can not find the annotation : BindPresenter, for [ "
	                    + view.getClass().getName() + " ] ");
	        }
	        Class<? extends MMVPPresenter> presenterClasses[] = annotation.value();
	        if (presenterClasses.length < 1) {
	            throw new IllegalArgumentException(" Invalid presenter size for [ " + view.getClass().getName() + " ]");
	        }
	        final LinkedList<MMVPPresenter> presenterLinkedList = new LinkedList<>();
	        for (Class<? extends MMVPPresenter> presenterClass : annotation.value()) {
	            try {
	                MMVPPresenter presenter = presenterClass.newInstance();
	                presenterLinkedList.add(presenter);
	            } catch (InstantiationException e) {
	                e.printStackTrace();
	            } catch (IllegalAccessException e) {
	                e.printStackTrace();
	            }
	        }
	        VIEW_CACHE.add(view);
	        VIEW_PRESENTERS_CACHE.put(view.getClass(), presenterLinkedList);
	        if (enableLog) {
	            Log.d(TAG, " RegisterView [ " + view.getClass().getName() + " ]");
	        }
	    }
	
	
	    /**
	     * 注销View
	     *
	     * @param view {@link MMVPView}
	     */
	    @UiThread
	    public static void unregisterView(@NonNull MMVPView view) {
	        VIEW_CACHE.remove(view);
	        for (Class<?> clazz : VIEW_PRESENTERS_CACHE.keySet()) {
	            List<MMVPPresenter> presenterList = VIEW_PRESENTERS_CACHE.get(clazz);
	            if (null == presenterList || presenterList.isEmpty()) {
	                continue;
	            }
	            for (MMVPPresenter presenter : presenterList) {
	                METHODS_CACHE.remove(presenter.getClass());
	            }
	        }
	        METHODS_CACHE.remove(view.getClass());
	        VIEW_PRESENTERS_CACHE.remove(view.getClass());
	        if (enableLog) {
	            Log.d(TAG, " unregisterView [ " + view.getClass().getName() + " ]");
	        }
	    }
	
	
	    /**
	     * 获取某个Presenter
	     *
	     * @param viewClass      当前viewClass
	     * @param presenterClass 需要的presenterClass
	     * @param <T>            需要的presenter
	     * @return 需要的presenter
	     */
	    @Nullable
	    public static <T extends MMVPPresenter> T getPresenter(@NonNull Class<? extends MMVPView> viewClass,
	                                                           @NonNull Class<? extends MMVPPresenter> presenterClass) {
	        List<MMVPPresenter> presenterList = VIEW_PRESENTERS_CACHE.get(viewClass);
	        if (null == presenterList || presenterList.isEmpty()) {
	            return (T) null;
	        }
	        for (MMVPPresenter presenter : presenterList) {
	            if (presenterClass.equals(presenter.getClass())) {
	                return (T) presenter;
	            }
	        }
	        return (T) null;
	    }
	
	
	    /**
	     * 找到某个类里注册了{@link ActionProcess}的方法,该方法注册的action为传入的{@link MMVPAction}里的action
	     *
	     * @param action {@link MMVPAction}
	     * @return
	     */
	    private static Method findRegisterMMVPActionMethod(@NonNull MMVPAction action) {
	        final Class<?> targetClass = action.getTargetClass();
	        final String methodKey = String.format("%s_$$_$$_%s", targetClass.getCanonicalName(),
	                action.getAction().getAction());
	        Map<String, Method> methodMap = METHODS_CACHE.get(targetClass);
	        Method executeMethod = null;
	        if (null != methodMap && !methodMap.isEmpty()) {
	            executeMethod = methodMap.get(methodKey);
	        }
	        if (null == executeMethod) {
	            Method[] methods = targetClass.getMethods();
	            for (Method method : methods) {
	                ActionProcess methodAnnotation = method.getAnnotation(ActionProcess.class);
	                if (null == methodAnnotation) {
	                    continue;
	                }
	                if (!TextUtils.equals(methodAnnotation.value(), action.getAction().getAction())) {
	                    continue;
	                }
	                executeMethod = method;
	                break;
	            }
	            if (null != executeMethod) {
	                if (null == methodMap) {
	                    methodMap = new LinkedHashMap<>();
	                    METHODS_CACHE.put(targetClass, methodMap);
	                }
	                methodMap.put(methodKey, executeMethod);
	            }
	        }
	        return executeMethod;
	    }
	}

	
不到400行,短小精悍。方法都有注释，命名还算规范吧，后面还会给项目的Github链接地址，所以这里就不打算详述其功能了，大致看一下他们的类图结构吧

![](https://ykbjson.github.io/blogimage/mvppicture4/mmvpuml.png)

最后，整个框架执行的大致的流程如下：

1. View通过registerView方法注册到MMVPArtist里面，然后MMVPArtist根据View的注解，生成对应的Presenter，并保存到一个和View有关的Map里。

2. View调用MMVPArtist的buildAction方法，创建一个MMVPAction，设置好参数，然后调用MMVPAction的send方法，发送到MMVPArtist里。

3. MMVPArtist根据收到的MMVPAction里的sourceClass判断，是View发起的请求还是Presenter发起的请求，如果是View发起的，那就调用handleAcctionFromView，如果是Presenter发起的，则调用handleActionFromPresenter。

4. 在handleAcctionFromView方法里，会找到一个Class是MMVPAction里指定的targetClass的Pressenter，在handleActionFromPresenter方法里，会找到一个Class是MMVPAction里指定的targetClass的View，最终都会执行execute方法.

5. 在execute方法里，会通过findRegisterMMVPActionMethod方法，找到一个View或Presenter用AcctionProcess注解的方法，然后根据注解里的属性，构造该方法需要的参数，反射执行该方法。如果execute方法执行失败（如有异常之类的），那么就会执行Presenter或View的handleAction方法，这个方法继承至IMMVPActionHandler。

6. 如果是View发起的调用请求，Presenter收到该MMVPAction并执行对应的方法之后，要通知发起该动作的View，Presenter只需要交换一下MMVPAction的targetClass和sourceClass，并重设MMVPAction的参数，然后调用MMVPAction的send方法，即可传递给最初动作的发起者View，完成一次信息传递。当然MMVPAction里还有一个IMMVPOnDataCalllback,Presenter也可以直接把数据传递给该Callback，舍弃MMVPAction。

7. 在View销毁的时候，调用MMVPArtist的unregister方法，移除MMVPArtist里关于View的缓存。


# 三、存在的一些问题

1. 如果大家都用过EventBus的话，可能就会觉得我这么做显得很多余，那些注解与Action分发，完全可以用EventBus框架完成。但是我这个思路和EventBus还是有所区别的，至少在针对Action的处理上，不会在发送一个Action后，所有注册该Action的方法都会调用，只有和某个View有关联的Presneter里的方法才会被筛选调用。

2. 就MVP模式而言，这种设计虽然让View和Presenter的关系变得更模糊，但是Presenter和View的设计就有点背离MVP模式了，这不是最初的目的。

3. 基于反射调用，性能问题总是不可避免，但是这里其实也可以像Butterknife一样，用APT技术去解决性能问题，等后面我研究完了再和大家分享。

嗯，暂时我这个笨脑袋能想到的就这么多，如果大家发现了别的什么问题，请不吝赐教，让我可以吃一堑长一智，谢谢。








	