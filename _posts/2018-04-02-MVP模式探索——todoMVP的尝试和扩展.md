---
layout:     post
title:      MVP模式探索——todoMVP的尝试和扩展
subtitle:   MVP模式探索3
date:       2018-04-02
author:     YANKEBIN
catalog: true
tags:
     - Android
     - MVP
     - todoMVP
     - 设计模式
---


# 前言

构思了一段时间，恰巧今天也有空，就把MVP探索系列的文章暂时画上一个句号吧，当然，知识的海洋是无穷无尽的，学海无涯，在以后的工作中，如果鄙人总结出了一些好的相关的经验或是学习到相关的优秀的知识没我还会回来和大家分享的。

其实,todoMVP是google官方出品的在Android项目中使用MVP模式的一种思路，应该算是比较权威的学习资料吧，研究和学习的人肯定很多，我作为一枚Android程序员，还是一枚普通的Android程序员，其实是不太敢轻易去置喙什么的，只是将我在项目里的实际使用心得和变更的过程写出来，供大家参考，使自己更深刻的理解MVP这种模式。

# 一 、todoMVP简介

>[先附上todoMVP的官方示例地址](https://github.com/googlesamples/android-architecture/tree/todo-mvp)

## 1.1 todoMVP的出现的原因猜测

自从 2015下半年来，MVP渐渐崛起成为了现在普遍流行的架构模式。但是各种不同实现方式的MVP架构层出不穷，也让新手不知所措。而Google作为“老大哥”，针对此现象为Android架构做出了“规范示例”：android-architecture。

## 1.2 todoMVP的原理图

![](https://raw.githubusercontent.com/wiki/googlesamples/android-architecture/images/mvp.png)

google把model层更加细化，区分出本地数据库、网络、内存来管理数据；Activity直接实现presenter可以根据自身的生命周期,直接控制presenter的生命周期；Fragment直接实现view，方便Activity控制和管理。总的来说，就是用大家习惯和熟悉的东西实现了MVP模式，不愧是官方出品。

## 1.3 todoMVP架构类图（部分）

这里以TaskDetail和TaskDataSource层为例

TaskDetail

![](https://ykbjson.github.io/blogimage/mvppicture3/task_detail.png)

TaskDataSource

![](https://ykbjson.github.io/blogimage/mvppicture3/task_data_source.png)


## 1.4 todoMVP的一般分包结构

![](http://ofyt9w4c2.bkt.clouddn.com/20170222/20170227212108.png)

根据图片可以看出来，todoMVP模式是按功能模块分包的，项目结构和功能关系一目了然，便于维护。在每个功能模块里，google引入了一个“契约”类——xxxxContract,该类用来定义该模块下所有的prsenter和view要实现的功能接口以及presenter和view的关系。如果想了解某个模块有些什么功能，直接看这个契约类就可以了;如果想了解该模块下presenter和view的关系，也可以直接看这个契约类。我们以TasksContract(位于tasks包下)为例

'

    public interface TasksContract {
	
	    interface View extends BaseView<Presenter> {
	
	        void setLoadingIndicator(boolean active);
	
	        void showTasks(List<Task> tasks);
	
	        void showAddTask();
	
	        void showTaskDetailsUi(String taskId);
	
	        ...
	    }
	
	    interface Presenter extends BasePresenter {
	
	        void result(int requestCode, int resultCode);
	
	        void loadTasks(boolean forceUpdate);
	
	        void addNewTask();
	
	        ...
	    }
	}

'

看了这个类以后，是不是不用去看该模块下所有的代码，就已经知道该模块大体的功能了？

todoMVP相关的信息就简单介绍到这里啦，没什么好总结的，非常简洁。如果想详细的了解更多，可以去刚才给出的链接地址那里慢慢看，英语不好的童鞋请带好翻译...下面我们将探讨todoMVP模式在实际项目里使用的过程和问题。


# 二、todoMVP在实际项目里使用的问题


## 2.1 遇到的问题

我相信，大多数开发者的项目里都多多少少涉及到网络请求吧，同步获取数据的方式我暂且抛开不谈，我们来谈一下在Fragment或Activity里面使用异步方式获取网络数据的时候，大家都会关系的一个问题，就是网络请求的生命周期和界面本身生命周期的问题。

举个简单的例子哈，Retrofit应该有很多人用过吧，非常好用吧，杰克沃尔顿大神的精品之一，标准的RESTFull设计，基于OkHttp，支持同步异步加载数据，用注解把复杂的网络请求变换成Java接口，对于习惯于面向对象的程序员来说，简直就是福利啊。当我们在一个界面（Fragment或Activity）里使用Retrofit发起异步请求的时候，我们是这么做的

'

	// 第1部分：在网络请求接口的注解设置
	@GET("openapi.do?keyfrom=Yanzhikai&key=2032414398&type=data&doctype=json&version=1.1&q=car")
	Call<Translation>  getCall();
	
	// 第2部分：在创建Retrofit实例时通过.baseUrl()设置
	Retrofit retrofit = new Retrofit.Builder()
	                .baseUrl("http://fanyi.youdao.com/") //设置网络请求的Url地址
	                .addConverterFactory(GsonConverterFactory.create()) //设置数据解析器
	                .build();
	
	//第3部分发送网络请求(异步)
        Call<Translation> call= retrofit.create(ITranslation.class).getCall();
        call.enqueue(new Callback<Translation>() {
            //请求成功时回调
            @Override
            public void onResponse(Call<Translation> call, Response<Translation> response) {
                //请求处理,输出结果
                response.body().show();
            }

            //请求失败时候的回调
            @Override
            public void onFailure(Call<Translation> call, Throwable throwable) {
                System.out.println("连接失败");
            }
        });
	
'

我们主要看第3部分，当onResponse或者onFailure方法回调的时候，我们可定时要去渲染UI组件的吧，那如果这个时候，Activity或者Fragment已经destroy了，会出现什么结果？我想，大多数时候出现的都是NPE(NullPointerException),因为destroy方法之后，Activity或者Fragment的UI组件已经被系统回收了，而我们却要去渲染他们，所以一般会出现NPE。

## 2.2 解决思路

当然，作为一枚程序员，解决这个问题非常简单，一般我们有几种方法解决这个问题：

1.我们可以把Call对象放到Activity或者Fragment的全局，在Activity或者Fragment已经destroy的时候调用其cancel方法，取消执行的请求...然而，我可以完全负责任的告诉你，在Retrofit1.x版本里是行不通的，因为你翻看源码会发现，cancel方法只会取消在队列里还没有执行的请求，那些已经执行了的请求是没有效果的。当然啦，Retrofit2.x的Call的cancel方法是有效的，不过这样的方法导致的结果是，**在每个有网络请求的页面，你都得维护一个请求列表或是map**。

2.我们可以在全局定义一个布尔值，在Activity或者Fragment执行destroy的时候去控制这个布尔值，然后当onResponse或者onFailure方法回调的时候，我们根据这个布尔值看是否渲染UI组件。又或者，Activity或者Fragment本身就有获取自身生命周期状态的方法，当onResponse或者onFailure方法回调的时候，我们根据那些方法的返回值看是否渲染UI组件。这样也可以有效的避免NPE问题，不过这样的办法导致的结果是，**在每个页面的每个网络请求回调里你都不得不去写一个或多个判断**。

3.以上两个方法都很好实现，这里就不做详细的展开和尝试了，下面我们着重讨论另外一种办法，就是在回调接口Callback里利用弱引用，将请求网络的真正发起者关联进来，在适当的时候通过**请求网络的真正发起者其自身的方法获取其自身的状态**来决定Callback是否需要继续执行回调。有点拗口，但是其实实现起来很简单的。

>当然，其实还有一些开源框架，可以绑定Activity或者Fragment的生命周期和网络请求的生命周期，这里暂且不做深入讨论。

## 2.3 具体实现

现在，我们回到todoMVP模式来，todoMVP模式里，所有的数据操作，当然也就包括网络数据请求操作，都是封装在xxxRepository里面的，那么，我们是否可以改造一下Repository和Callback，让Repository可已把真正请求网络的发起者传递给Callback，让Callback自己管理其回调流程呢？我们以一个项目里登录模块为例

先看看大致的包结构

![](https://ykbjson.github.io/blogimage/mvppicture3/login_package_info.png)

然后是BasePresenter接口和BaseView接口

BasePresenter接口,和todoMVP里的一模一样

'
	
	public interface BasePresenter {
	
	    void start();
	    
	}

'

BaseView接口，稍微有一点改动，增加了一个获取Context方法（其实后面也没用到）

'

	interface BaseView<T> {
	    @NonNull
	    Context getMContext();
	
	    void setPresenter(@NonNull T presenter);
	}
	

'

我在这里，为了让要充当View的Fragment或Activity方便实绑定和Presenter的关系，所以封装了一个BaseMVPPresenter


'

	public abstract class BaseMVPPresenter<V extends BaseView> implements BasePresenter {
	
	    protected V mView;
	
	    public BaseMVPPresenter(V view) {
	        if (null == view) {
	            throw new NullPointerException("View can not be null");
	        }
	        this.mView = view;
	        //noinspection unchecked
	        this.mView.setPresenter(this);
	    }
	
	    @Override
	    public void start() {
	
	    }
	}

'


然后一个BaseMVPFragment应运而生

'

	public abstract class BaseMVPFragment<T extends BaseMVPPresenter> extends BaseFragmentV4 implements BaseView<T> {
	
	    protected T presenter;
	
	    @Override
	    public void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        if (null != presenter) {
	            presenter.start();
	        }
	    }
	
	    @Override
	    public void onSaveInstanceState(Bundle outState) {
	        if (null == outState) {
	            outState = new Bundle();
	        }
	        outState.putSerializable("FragmentPresenter", presenter);
	        super.onSaveInstanceState(outState);
	    }
	
	    @Override
	    public void onViewStateRestored(@Nullable Bundle savedInstanceState) {
	        if (null == presenter && null != savedInstanceState && savedInstanceState.containsKey("FragmentPresenter")) {
	            try {
	                presenter = (T) savedInstanceState.getSerializable("FragmentPresenter");
	                if (null!=presenter){
	                    presenter.mView=this;
	                }
	            } catch (Exception e) {
	                e.printStackTrace();
	            }
	        }
	        super.onViewStateRestored(savedInstanceState);
	    }
	
		@Override
	    public Context getMContext() {
	        return getContext();
	
	    }
	
		@Override
	    public void setPresenter(T presenter) {
	        this.presenter = presenter;
	    }
	}

'

持有一个BaseMVPPresenter的泛型，实现BaseView接口。**在这里涉及到一个很别扭的问题，后面我会提出来，希望有高手可以指点迷津**。


现在，按照todoMVP的惯例，我们开始写那个契约类

'

	interface LoginContract {
	
	    interface BaseLoginView extends BaseView<BaseLoginPresenter> {
	
	        void onLoginSuccess();
	
	        void onLoginFailed(String reason);
	
	        String getAccount();
	
	        String getPassword();
	    }
	
	    abstract class BaseLoginPresenter extends BaseMVPPresenter<BaseLoginView> {
	        BaseLoginPresenter(BaseLoginView view) {
	            super(view);
	        }
	
	        abstract void doLogin();
	    }
	
	}

'

这个契约类简单明了，LoginView实现四个方法，LoginPresenter实现一个登录操作。


登录的Presenter,LoginPresenter

'

	final class LoginPresenter extends LoginContract.BaseLoginPresenter {
	
	    private LoginRepository repository;
	
	    LoginPresenter(LoginContract.BaseLoginView view) {
	        super(view);
	        repository = new LoginRepository(view);
	    }
	
	    @Override
	    void doLogin() {
	        final String account = mView.getAccount();
	        final String password = mView.getPassword();
	        if (TextUtils.isEmpty(account)) {
	            mView.onLoginFailed("账号不能为空！");
	            return;
	        }
	        if (TextUtils.isEmpty(password)) {
	            mView.onLoginFailed("密码不能为空！");
	            return;
	        }
	
	        repository.doLogin(account, password, new OnLoadDataCallback<User>() {
	            @Override
	            public void onSuccess(User data) {
	                saveUser(data);
	                mView.onLoginSuccess();
	            }
	
	            @Override
	            public void onError(Object error) {
	                mView.onLoginFailed(error.toString());
	            }
	        });
	    }
	
	    /**
	     * 存储用户信息
	     *
	     * @param user
	     */
	    private void saveUser(User user) {
	        ...
	    }
	}

'
这里主要注意的是构造方法里的两句代码，第一句让实现BaseMVPPresenter的类持有了实现BaseView的View层；第二句，就是让Repository也持有了实现BaseView的View层（大部分情况下的原始发起者）。


我们为了减少Repository的代码，抽象出一个BaseRepository


'

	public class BaseRepository {
	    protected Object token;
	    public BaseRepository(Object token) {
	        this.token = token;
	    }
	}

'

在进一步区分本地和网络请求操作，再抽出BaseRemoteRepository

'

	public class BaseRemoteRepository  extends BaseRepository{
	
	    public BaseRemoteRepository(@NonNull Object token) {
	        super(token);
	    }
	
	    protected String combineUrl(String endPoint) {
	        return Constant.URL.URL_BASE_URL_.concat(endPoint);
	    }
	
	    protected <T> T create(Class<T> clazz) {
	        return create(clazz, true);
	    }
	
	    protected <T> T create(Class<T> clazz, boolean needDefaultHeaders) {
	        return RetrofitApi.getInstance().create(clazz, needDefaultHeaders);
	    }
	}

'

这里只是附加了一些便于发起网络请求的方法。


按照Retrofit的模式，我们要定义登录相关的接口啦

'

	public interface ILoginDataSource {
	
	    interface ILogin {
	        /**
	         * 登录
	         */
	        @POST(Constant.URL.URL_FOR_LOGIN)
	        Call<ResponseBean<User>> doLogin(@Body Map<String, Object> map);
	
	        /**
	         * 注销
	         */
	        @POST(Constant.URL.URL_FOR_LOGOUT)
	        Call<ResponseBean> doLogout(@Body Map<String, Object> map);
	    }
	
	    void doLogin(String mobile, String password, OnLoadDataCallback<User> callback);
	
	    void doLogout(OnLoadDataCallback<Void> callback);
	}

'

两个方法，登录、注销。ILogin里是Retrofit要调用的实际的方法。


然后我们来看看登录请求操作的真正操作者LoginRepository

'

	public class LoginRepository extends BaseRemoteRepository implements ILoginDataSource {
	
	    public LoginRepository(@NonNull Object token) {
	        super(token);
	    }
	
	    @Override
	    public void doLogin(String mobile, String password, final
	    OnLoadDataCallback<User> callback) {
	        // TODO: 2018/3/22 replace with real logic
	        TokenBean bean = new TokenBean();
	        bean.setAccessToken("9rirofnvndjsjnsdjv_lfs47qwq^^");
	        User user = new User();
	        user.setAvatar("http://pic.downcc.com/upload/2016-7/20167181357357469.png");
	        user.setName("孙吉吉");
	        user.setRole(1);
	        user.setId(1);
	        user.setMobile(mobile);
	        user.setToken(bean);
	        callback.onSuccess(user);
	
	//        Map<String, Object> map = new HashMap<>();
	//       	create(ILogin.class).doLogin(map)
	//                .enqueue(new RCallback<ResponseBean<User>>(token) {
	//                    @Override
	//                    public void success(ResponseBean<User> responseBean) {
	//                        if (responseBean.isAvailable()) {
	//                            callback.onSuccess(responseBean.getData());
	//                        } else {
	//                            callback.onError(responseBean.getMessage());
	//                        }
	//                    }
	//
	//                    @Override
	//                    public void failure(String s) {
	//                        callback.onError(s);
	//                    }
	//                });
	    }
	
	    @Override
	    public void doLogout(final OnLoadDataCallback<Void> callback) {
	        Map<String, Object> map = new HashMap<>();
	        create(ILogin.class).doLogout(map)
	                .enqueue(new RCallback<ResponseBean>(token) {
	                    @Override
	                    public void success(ResponseBean responseBean) {
	                        if (responseBean.isAvailable()) {
	                            callback.onSuccess(null);
	                        } else {
	                            callback.onError(responseBean.getMessage());
	                        }
	                    }
	
	                    @Override
	                    public void failure(String s) {
	                        callback.onError(s);
	                    }
	                });
	    }
}

'

看doLogin方法里被注释的部分，这就是真正的登录请求逻辑。

到这里，登录的发起逻辑就算完成了，那么我前面说的Callback自己管理回调流程的逻辑是这么实现的呢？这里似乎还看不出一点关系？事实上，细心的人肯定已经看出了端倪，

'

 		 Map<String, Object> map = new HashMap<>();
        create(ILogin.class).doLogin(map)
                .enqueue(new RCallback<ResponseBean<User>>(token) {
                    @Override
                    public void success(ResponseBean<User> responseBean) {
                        if (responseBean.isAvailable()) {
                            callback.onSuccess(responseBean.getData());
                        } else {
                            callback.onError(responseBean.getMessage());
                        }
                    }

                    @Override
                    public void failure(String s) {
                        callback.onError(s);
                    }
                });
                
'


这里有一句代码new RCallback<ResponseBean<User>>(token)，这个token是什么呢？我们回去看一眼LoginPresenter的构造方法你就明白了，我们暂且认为他就是“View”吧。接下来，我们看看Callback把这个View拿去干嘛了。

为了便于实现Callback，我封装了一层SCallback

'

	public abstract class SCallback<T> implements Callback<T> {
	    protected boolean isCanceled;
	    protected ContextHolder<Object> contextHolder;
	
	    public SCallback(@NonNull Object host) {
	        contextHolder = new ContextHolder<>(host);
	    }
	
	    /**
	     * 检测是否符合回调服务器返回数据的条件
	     *
	     * @param call
	     * @return
	     */
	    protected boolean checkCanceled(Call call) {
	        return isCanceled || call.isCanceled() || !contextHolder.isAlive();
	    }
	
	
	    private void cancel() {
	        isCanceled = true;
	    }
	
	    /**
	     * 请求数据失败
	     *
	     * @param t
	     */
	    public abstract void success(T t);
	
	    /**
	     * 请求数据成功
	     *
	     * @param errorMessage
	     */
	    public abstract void failure(String errorMessage);
	}

'

我们关注一下checkCanceled(Call call)方法，我们定义了一个全局变量isCanceled，供外部手动调用cancel方法，这个意义很明显，就是告诉Callback，我不在需要你的回调了。如果外部没有手动调用cancel方法，那么就结合!contextHolder.isAlive()的值来判断是否需要回调。ContextHolder，没什么神奇的，就是个弱引用

'
	public class ContextHolder<T> extends WeakReference<T> {
	    public ContextHolder(T r) {
	        super(r);
	    }
	
	    /**
	     * 判断是否存活
	     *
	     * @return
	     */
	    public boolean isAlive() {
	        T ref = get();
	        if (ref == null) {
	            return false;
	        } else {
	            if (ref instanceof Service) {
	                return isServiceAlive((Service) ref);
	            } else if (ref instanceof Activity) {
	                return isActivityAlive((Activity) ref);
	            } else if (ref instanceof Fragment) {
	                return isFragmentAlive((Fragment) ref);
	            } else if (ref instanceof android.support.v4.app.Fragment) {
	                return isV4FragmentAlive((android.support.v4.app.Fragment) ref);
	            } else if (ref instanceof View) {
	                return isContextAlive(((View) ref).getContext());
	            } else if (ref instanceof Dialog) {
	                return isContextAlive(((Dialog) ref).getContext());
	            }
	        }
	        return true;
	    }
	
	    /**
	     * 判断服务是否存活
	     *
	     * @param candidate
	     * @return
	     */
	    boolean isServiceAlive(Service candidate) {
	        if (candidate == null) {
	            return false;
	        }
	        ActivityManager manager = (ActivityManager) candidate.getSystemService(Context.ACTIVITY_SERVICE);
	        List<ActivityManager.RunningServiceInfo> services = manager.getRunningServices(Integer.MAX_VALUE);
	        if (services == null) {
	            return false;
	        }
	        for (ActivityManager.RunningServiceInfo service : services) {
	            if (candidate.getClass().getName().equals(service.service.getClassName())) {
	                return true;
	            }
	        }
	        return false;
	    }
	
	    /**
	     * 判断activity是否存活
	     *
	     * @param a
	     * @return
	     */
	    boolean isActivityAlive(Activity a) {
	        if (a == null) {
	            return false;
	        }
	        if (a.isFinishing()) {
	            return false;
	        }
	        return true;
	    }
	
	    /**
	     * 判断fragment是否存活
	     *
	     * @param fragment
	     * @return
	     */
	    @TargetApi(Build.VERSION_CODES.HONEYCOMB_MR2)
	    boolean isFragmentAlive(Fragment fragment) {
	        boolean ret = isActivityAlive(fragment.getActivity());
	        if (!ret) {
	            return false;
	        }
	        if (fragment.isDetached()) {
	            return false;
	        }
	        return true;
	    }
	
	    /**
	     * 判断fragment是否存活
	     *
	     * @param fragment
	     * @return
	     */
	    boolean isV4FragmentAlive(android.support.v4.app.Fragment fragment) {
	        boolean ret = isActivityAlive(fragment.getActivity());
	        if (!ret) {
	            return false;
	        }
	        if (fragment.isDetached()) {
	            return false;
	        }
	        return true;
	    }
	
	    /**
	     * 判断是否存活
	     *
	     * @param context
	     * @return
	     */
	    boolean isContextAlive(Context context) {
	        if (context instanceof Service) {
	            return isServiceAlive((Service) context);
	        } else if (context instanceof Activity) {
	            return isActivityAlive((Activity) context);
	        }
	        return true;
	    }
	}

'

我前面说了一句很拗口的话：**“通过请求网络的真正发起者其自身的方法获取其自身的状态来决定Callback是否需要继续执行回调”**，看了这个类之后，是不是有柳暗花明又一村的感觉？这个类就是对所持有的真正的网络发起者的状态进行判断，帮助Callback判断是否需要继续执行回调，其本身也是一个弱引用，所以相对来说，其持有的对象更容易被回收，被回收后，Callback也不会在执行回调流程了（看isAlive方法）。


接下来就是我们实现SCallback的实际参与回调的RCallback

'

	public abstract class RCallback<T extends BaseResponseBean> extends SCallback<T> {
	
	    public RCallback(@NonNull Object host) {
	        super(host);
	    }
	
	    @Override
	    public final void onResponse(Call<T> call, Response<T> response) {
	        //请求失败
	        if (!response.isSuccessful()) {
	            onFailure(call, new IllegalArgumentException("Request data failed"));
	            return;
	        }
	        T baseBen = response.body();
	        //数据为空
	        if (null == baseBen) {
	            onFailure(call, new IllegalArgumentException("Invalid data returned by the server"));
	            return;
	        }
	
	        boolean isTokenError = TextUtils.equals(baseBen.getCode(), ErrorCode.ERROR_CODE_TOKEN_ERROR);
	        //token错误或过期
	        if (isTokenError) {
	            ToolToast.showShort(baseBen.getMessage());
	            Appcontext.assistApp.backToLogin();
	        }
	        //请求的发起者是否还合适接收回调
	        if (checkCanceled(call)) {
	            return;
	        }
	        //回调数据
	        success(baseBen);
	    }
	
	    @Override
	    public final void onFailure(Call<T> call, Throwable t) {
	        Utils.handleException(t);
	        if (checkCanceled(call)) {
	            return;
	        }
	        String errorMessage;
	        if (!ToolNetwork.getInstance().isAvailable()) {
	            errorMessage = "网络未连接，请检查！";
	        } else if (t instanceof UnknownHostException) {
	            errorMessage = "服务器地址无法解析！";
	        } else if (t instanceof HttpException) {
	            errorMessage = "服务器错误！";
	        } else if (t instanceof SocketTimeoutException) {
	            errorMessage = "服务器连接超时！";
	        } else if (t instanceof IOException) {
	            errorMessage = "服务器连接失败！";
	        } else {
	            if (t instanceof IllegalArgumentException) {
	                errorMessage = t.getMessage();
	            } else {
	                errorMessage = "未知错误";
	            }
	        }
	        failure(errorMessage);
	    }

	}

'



RCallback是对SCallback的扩展，可以根据不同项目和需求实现不同的SCallback，这里面我们主要看onResponse方法和onFailure方法。他们都有一个共同的逻辑处理： if (checkCanceled(call)) {
            return;
        }。这个checkCanceled方法刚才我们已经讨论过了，到了这一步，Callback就解决了**“通过请求网络的真正发起者其自身的方法获取其自身的状态来决定Callback是否需要继续执行回调”**的问题。
        
 
前面还说了一有一个很别扭的问题，就是BaseMVPFragment其实已经实现了BaseView，然而我们的LoginFragment也实现了一个实现至BaseView的BaseLoginView。在这个环节上一直让我很困惑：**一个已经实现了“某个“接口的类又去实现了一个继承至”某个“接口的类**，我感觉这里肯定是有问题的，可是自己又说不上来问题到底在哪里，希望有精通设计模式的高手指点迷津，感激不尽。


# 三、结语

其实标题所说的是todoMVP的尝试和扩展，但看到这里，大家可能并没有看到太多todoMVP，这绝对是我水平有限的原因。在这里我所阐述的，就是在实际使用todoMVP模式的过程中，为了适应项目本身，而不得不让todoMVP模式做出了一些调整，比如Presenter的调整，比如Repository的调整，而这个调整的过程并不是像现在这篇文章里寥寥几百字就能阐述完成的，我前面列出的为了解决NPE的三个方法我都一一实践过。在这里还是要强调我前面第一篇文章里说过的话：**没有最好的架构或设计模式，只有最适合的架构或设计模式**。 

MVP模式系列的探讨到此暂时搞一个段落吧，我可能需要花更多的时间去回忆去分析以前的所有实践过程，等哪一天彻底参透了MVP的种种，再来和大家一起分享吧。
        














