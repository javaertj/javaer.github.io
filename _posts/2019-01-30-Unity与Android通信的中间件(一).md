---
layout:     post
title:      Unity与Android通信的中间件(一)
subtitle:   Unity3d与android的那些事儿
date:       2019-01-30
author:     YANKEBIN
catalog: true
tags:
     - Android
     - Unity3d
     - Android与Unity3d通信
     - Unity3d与Android通信
---

# 一 、前言
最近有幸接触到unity，也刚好有时间，索性就花了点时间来认识和学习unity，学了差不多一个多月吧，算是窥探到了一点点unity的门路，本想再继续往深处研究下的，但是在继续学习的过程中发现unity和Android通信稍微有点不太畅快，就是unity端和Android端要通信的的话，我觉得有两个问题比较麻烦：

1.两端代码的依赖度比较高。怎么说呢，如果你有一点unity3d基础，你就知道unity要调用Android的话，则必须确切的知道Android端具体的的类名、方法名以及方法需要的参数等信息，这样的话，每个unity项目的代码就比较定制化，扩展性不强；Android端调用unity端倒是比较简单，因为unity的sdk里封装好了对应的方法，不过还是有一点麻烦的地方就是，当要调用的unity端的方法太多差异太大的时候，就没有办法进行较好的封装，不便于维护。

2.unity编译生成的android项目里，呈现unity场景的Activity需要加入大量的代码，并且没有支持在fragment里呈现场景的示例，每次集成都需要做很多重复性工作。

为了解决在学习和使用unity的过程中遇到的这些问题，所以我就花了点时间实现了一个可以帮助android开发者快速对接unity工程的插件工程。这个插件工程支持使用Activity、Fragment、View等组件呈现unity场景，虽然谈不上什么高大上，但是接入简单，只需少量代码即可完成与unity工程的对接。具体的可以去看github的demo和unity3dplugin源码。

>[demo依赖的unity工程](https://code.aliyun.com/modelingwithunity3d/unity3dballgame.git)


>[android端demo和plugin源码工程](https://github.com/ykbjson/Unity3DPlugin


>ps：demo依赖的unity工程有个文件超过了100M，无法上传到github，所以上传到了code.aliyun.com,没账号的童鞋注册后登录了再点击链接就可以了。

想要快速看到效果？

1.clone **demo依赖的unity工程**到本地，用unity编辑器导入，编译并导出为android project备用。

2.新建一个android项目，并参照**android端demo和plugin源码工程**里的demo，集成unity3dplugin，然后复制第一步生成的android project里src/main/assets目录里的内容到你工程的src/main/assets目录下，其他代码参照demo里的即可。

**或者，直接clone demo编译运行。**


>关于unity3d入门，这里有一个很好的链接，大家要耐心看完：[一个小时内用Unity3D制作一个小游戏](https://www.bilibili.com/video/av7532211/)


# 二 、实现思路

>不了解unity3d与android通信的请先戳这里：[Android与Unity交互研究](https://blog.csdn.net/crazy1235/article/details/46733221)

上面说到了这个插件主要是为了解决两个问题：一个是降低android端代码和unity端代码的耦合度，还有一个是降低android端的接入复杂度，那我们就一个一个来解决吧。

## 2.1 解决unity和android两端代码的依赖度比较高的问题

为了降低android端代码和unity端代码的耦合度，我分了两个方面来考虑：一个是android端提供给unity调用的入口统一化，unity所发来的所有消息都经过一个单一对象分发出去，这样的话，两端就不会再出现直接调用彼此具体的某个方法之类的了；另一个是两端通信的消息内容标准化，无论是unity3d调用android还是android调用unity3d，我都让他们传递一个ICallInfo来通信，至于具体要调用哪个类的那个方法，这些信息都封装在ICallInfo里。

### 2.1.1 android端提供给unity调用的单一入口——AndroidCall

		public class AndroidCall {
	    private final String TAG = getClass().getName();
	    public static boolean enableLog;
	
	    private IOnUnity3DCall onUnity3DCall;
	    private SoftReference<Activity> hostContext;
	
	    /**
	     * In order to further relax the restrictions of OnUnity3DCall, let Fragment implement OnUnity3DCall can also load Unity3D view, so added {@link IGetUnity3DCall}
	     *       interface.
	     *
	     * @param iGetUnity3DCall
	     */
	    public AndroidCall(@NonNull IGetUnity3DCall iGetUnity3DCall) {
	        this.onUnity3DCall = iGetUnity3DCall.getOnUnity3DCall();
	        hostContext = new SoftReference<>((Activity) onUnity3DCall.gatContext());
	    }
	
	    public void destroy() {
	        if (null != hostContext) {
	            hostContext.clear();
	        }
	    }
	
	    protected void checkConfiguration() {
	        if (null == hostContext || null == hostContext.get()) {
	            throw new RuntimeException(getClass().getSimpleName() + " ,Invalid Context");
	        }
	    }
	
	    @Nullable
	    public Context getApplicationContext() {
	        final Context context = getContext();
	        return null == context ? null : context.getApplicationContext();
	    }
	
	    @Nullable
	    public Context getContext() {
	        return null == hostContext || null == hostContext.get() ? null : hostContext.get();
	    }
	
	
	    public void onVoidCall(@NonNull String param) {
	        if (enableLog) {
	            Log.d(TAG, "onVoidCall, param : " + param);
	        }
	        onAndroidVoidCall(CallInfo.Builder.create().build(param));
	    }
	
	    public Object onReturnCall(@NonNull String param) {
	        if (enableLog) {
	            Log.d(TAG, "onReturnCall, param : " + param);
	        }
	        return onAndroidReturnCall(CallInfo.Builder.create().build(param));
	    }
	
	    public void onAndroidVoidCall(@NonNull ICallInfo param) {
	        checkConfiguration();
	        if (null == onUnity3DCall) {
	            return;
	        }
	        onUnity3DCall.onVoidCall(param);
	    }
	
	    public Object onAndroidReturnCall(@NonNull ICallInfo callInfo) {
	        checkConfiguration();
	        if (null == onUnity3DCall) {
	            return null;
	        }
	        return onUnity3DCall.onReturnCall(callInfo);
	    }
	}


91行代码，提供的功能也简单，供unity端反射实例化，然后在需要调用android端方法的时候，直接调用这个类的onVoidCall方法或onReturnCall方法，由于unity端传递过来的消息的内容都是json字符串，这里需要把这些json字符串转换成ICallInfo对象，然后再把ICallInfo对象下发给当前持有的IOnUnity3DCall对象，实现了IOnUnity3DCall接口的对象就可以从ICallInfo里读取数据开始处理了。

至于这里面出现的IGetUnity3DCall、IOnUnity3DCall等对象，是为了android端的Fragment、View等也能显示unity场景而设计的扩展接口，后面会讲到。

### 2.1.2 android端和unity通信的标准化载体——ICallInfo

	public interface ICallInfo extends Serializable {
	
	    @NonNull
	    String getCallMethodName();
	
	    @NonNull
	    String getCallModelName();
	
	    @Nullable
	    JSONObject getCallMethodParams();
	
	    @Nullable
	    ICallInfo getParent();
	
	    @Nullable
	    ICallInfo getChild();
	
	    boolean isUnityCall();
	
	    boolean isNeedCallMethodParams();
	
	    void send();
	}

android调用unity和uinty调用android还是有些不同的。android调用unity时，必需要指定modelName；而unity调用android时，不需要指定modelName，因为在unity端看来，当前呈现场景的对象必定是UnityPlayer里的currentActivity。

这里把ICallInfo定义成一个接口，是为了将来能扩展，每个项目可以根据自己实际的需要去扩展ICallInfo，以实现更适合的通信内容。但是就目前来看，我所实现的CallInfo几乎已经可以实现高度的自定义化了，因为我包含实际参数的对象是一个JSONObject对象。

我们来看看ICallInfo的实现，CallInfo

	public class CallInfo implements ICallInfo {
	    private String callModelName;
	    private String callMethodName;
	    private JSONObject callMethodParams = new JSONObject();
	    private CallInfo child;
	    private CallInfo parent;
	    private boolean unityCall = true;
	    private boolean needCallMethodParams = true;
	
	    public CallInfo() {
	    }
	
	    CallInfo(@Nullable Builder builder) {
	        this();
	        if (null != builder) {
	            setUnityCall(builder.unityCall)
	                    .setNeedCallMethodParams(builder.needCallMethodParams)
	                    .setCallModelName(builder.callModelName)
	                    .setCallMethodName(builder.callMethodName)
	                    .setCallMethodParams(builder.callMethodParams)
	                    .setChild(builder.child)
	                    .setParent(builder.parent);
	        }
	    }
	
	    public CallInfo setCallModelName(@Nullable String callModelName) {
	        this.callModelName = callModelName;
	        return this;
	    }
	
	    public CallInfo setCallMethodName(@NonNull String callMethodName) {
	        this.callMethodName = callMethodName;
	        return this;
	    }
	
	    public CallInfo setCallMethodParams(@Nullable JSONObject callMethodParams) {
	        if (null != callMethodParams) {
	            this.callMethodParams.putAll(callMethodParams);
	        }
	        return this;
	    }
	
	    public CallInfo setChild(@Nullable CallInfo child) {
	        this.child = child;
	        return this;
	    }
	
	    public CallInfo setParent(@Nullable CallInfo parent) {
	        this.parent = parent;
	        return this;
	    }
	
	    public CallInfo setUnityCall(boolean unityCall) {
	        this.unityCall = unityCall;
	        return this;
	    }
	
	    public CallInfo setNeedCallMethodParams(boolean needCallMethodParams) {
	        this.needCallMethodParams = needCallMethodParams;
	        return this;
	    }
	
	    public CallInfo addCallMethodParam(@NonNull String key, @Nullable Object value) {
	        this.callMethodParams.put(key, value);
	        return this;
	    }
	
	    @Override
	    @Nullable
	    public String getCallModelName() {
	        return callModelName;
	    }
	
	    @Override
	    @NonNull
	    public String getCallMethodName() {
	        return callMethodName;
	    }
	
	    @Override
	    @Nullable
	    public JSONObject getCallMethodParams() {
	        return callMethodParams;
	    }
	
	    @Override
	    @Nullable
	    public CallInfo getChild() {
	        return child;
	    }
	
	    @Override
	    @Nullable
	    public CallInfo getParent() {
	        return parent;
	    }
	
	    @Override
	    public boolean isUnityCall() {
	        return unityCall;
	    }
	
	    @Override
	    public boolean isNeedCallMethodParams() {
	        return needCallMethodParams;
	    }
	
	    @NonNull
	    @Override
	    public String toString() {
	        return JSONObject.toJSONString(this);
	    }
	
	    @Override
	    public void send() {
	        Unity3DCall.doUnity3DVoidCall(this);
	    }
	
	    public static class Builder {
	        private String callModelName;
	        private String callMethodName;
	        private JSONObject callMethodParams = new JSONObject();
	        private CallInfo child;
	        private CallInfo parent;
	        private boolean unityCall;
	        private boolean needCallMethodParams = true;
	
	        private Builder() {
	
	        }
	
	        public static Builder create() {
	            return new Builder();
	        }
	
	        public Builder callModelName(@Nullable String callModelName) {
	            this.callModelName = callModelName;
	            return this;
	        }
	
	        public Builder callMethodName(@NonNull String callMethodName) {
	            this.callMethodName = callMethodName;
	            return this;
	        }
	
	        public Builder addCallMethodParam(@NonNull String key, @Nullable Object value) {
	            this.callMethodParams.put(key, value);
	            return this;
	        }
	
	        public Builder child(@Nullable CallInfo child) {
	            this.child = child;
	            return this;
	        }
	
	        public Builder parent(@Nullable CallInfo parent) {
	            this.parent = parent;
	            return this;
	        }
	
	        public Builder unityCall(boolean unityCall) {
	            this.unityCall = unityCall;
	            return this;
	        }
	
	        public Builder needCallMethodParams(boolean needCallMethodParams) {
	            this.needCallMethodParams = needCallMethodParams;
	            return this;
	        }
	
	        public CallInfo build() {
	            return new CallInfo(this);
	        }
	
	        public CallInfo build(@Nullable String param) {
	            return JSONObject.parseObject(param, CallInfo.class);
	        }
	    }
	}


很简单，就是一个数据的封装，便于统一unity端和android端的通信信息，使用json传递数据，可以描述非常复杂的数据信息，轻松应对各种奇葩的数据需求。

有了AndroidCall和CallInfo，现在unity端已经可以和android端用CallInfo传递信息了，unity端的script代码也封装了一点，我们稍后再说，我们先来看android端调用unity端的代码。

	public class Unity3DCall {
	    /**
	     * Call Unity3d
	     *
	     * @param callInfo Carrier for Android and Unity3D interaction
	     */
	    public static void doUnity3DVoidCall(@NonNull ICallInfo callInfo) {
	        if(enableLog) {
	            Log.i("Unity3DCall", callInfo.toString());
	        }
	        UnityPlayer.UnitySendMessage(callInfo.getCallModelName(), callInfo.getCallMethodName(),
	                callInfo.isNeedCallMethodParams() ? callInfo.toString() : "");
	    }
	}

很简单吧，三个参数，modelName、methodName、param 。这其中的modelName对应的是unity组件挂载的script文件指定的名字，methodName对应的是unity组件挂载的script文件里的方法名字，param及时那个方法需要的参数啦，由于我们用ICallInfo传递数据，所以一般都是ICallInfo的json字符串。


### 2.1.3 unity端调用android的script的一点封装

上面已经说明了android如何调用unity端，为了unity端也比较容易的调用android端，所以我在unity端也封装了两个script文件：AndroidCaller、CallInfo

我们先看看AndroidCaller

	namespace AndroidCall {
	    /// <summary>
	    /// Android caller.封装AndroidCall
	    /// </summary>
	    public class AndroidCaller {
	        //和Android交互需要的对象
	        private AndroidJavaObject androidCall;
	
	        public AndroidCaller() {
	            //Android端Activity必须持有的对象，是一个FrameLayout
	            AndroidJavaClass unityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
	            //UnityPlayer构造方法取药一个hostActivity
	            AndroidJavaObject currentActivity = unityPlayer.GetStatic<AndroidJavaObject>("currentActivity");
	            //自定义Android和Unity3D交互的一个类
	            androidCall = new AndroidJavaObject("com.ykbjson.lib.unity3dplugin.AndroidCall",
	                                                new System.Object[] { currentActivity });
	        }
	
	        public void OnVoidCall(string param) {
	            CallInfo callInfo = JsonMapper.ToObject<CallInfo>(param);
	            OnVoidCall(callInfo);
	        }
	
	        public void OnVoidCall(CallInfo callInfo) {
	            androidCall.Call("onVoidCall", callInfo.ToString());
	        }
	
	        public object OnReturnCall(string param) {
	            CallInfo callInfo = JsonMapper.ToObject<CallInfo>(param);
	            return OnReturnCall(callInfo);
	        }
	
	        public object OnReturnCall(CallInfo callInfo) {
	            return androidCall.Call<object>("onReturnCall", callInfo.ToString());
	        }
	    }
	}

很简单的封装，把反射android端的AndroidCall的代码封装起来，不用再每个script文件里去写重复代码；把调用android端的方法封装一下，统一入口。

再看看CallInfo，和android端CallInfo大同小异，这边装载数据是用的是Dictionary，对应java的Map结构，因为android端用的是fastjson，他的JSONObject是实现了Map接口的

	namespace AndroidCall {
	    /// <summary>
	    /// CallInfo.调用Android代码时的消息封装
	    /// </summary>
	    [Serializable]
	    public class CallInfo {
	        public String callModelName;
	        public String callMethodName;
	        public Dictionary<string, object> callMethodParams = new Dictionary<string, object>();
	        public CallInfo child;
	        public bool needCallMethodParams = true;
	
	        public override string ToString() {
	            return JsonMapper.ToJson(this);
	        }
	    }
	}

到了这里，两端代码封装基本告一段落，接下来看看两端在封装后如何通信。

### 2.1.4 两端通信的代码片段

unity挂载的一个脚本


	/// <summary>
	/// Ball controller.游戏中小球的脚本
	/// </summary>
	public class BallController : MonoBehaviour {
	    public float speed;//可配置的移动速度
	    public Text countText;//可配置的得分显示
	    public Text winText;//可配置的获胜显示
	    private Rigidbody rb;//当前关联的可碰撞主体
	    private int count;//当前碰撞的Coin个数
	    //和Android交互需要的对象
	    private AndroidCaller androidCall;
	    //是否暂停的标志
	    private bool isPause;
	
	    void Start() {
	        //android端调用时指定的modelName
	        name = "Ball";
	        androidCall = new AndroidCaller();
	        rb = GetComponent<Rigidbody>();
	        count = 0;
	        countText.text = "Coins collected: " + count;
	        winText.text = "You Win!!!";
	        winText.gameObject.SetActive(false);
	
	    }
	
	    void Update() {
	        //判断是否点击了鼠标左键
	        //if (Input.GetMouseButtonDown(0)) {
	        //    isPause = !isPause;
	        //}
	    }
	
	    void FixedUpdate() {
	        //float xMov = Input.GetAxis("Horizontal");
	        //float zMove = Input.GetAxis("Vertical");
	        if (isPause) {
	            rb.velocity = Vector3.zero;
	            return;
	        }
	        float xMov = Input.acceleration.x;
	        float zMove = Input.acceleration.y;
	        Vector3 movemont = new Vector3(xMov, 0, zMove);
	        //钳制加速度向量到单位球
	        if (movemont.sqrMagnitude > 1) {
	            movemont.Normalize();
	        }
	        rb.AddForce(movemont * speed, ForceMode.VelocityChange);
	        //rb.//rb.MovePosition(rb.transform.position + movemont * speed);
	    }
	
	    void OnTriggerEnter(Collider other) {
	        if (other.gameObject.CompareTag("Coin")) {
	            CallInfo callInfo = new CallInfo {
	                callMethodName = "showToast"
	            };
	            other.gameObject.SetActive(false);
	            count++;
	            countText.text = "Coins collected: " + count;
	            if (count == 9) {
	                gameObject.SetActive(false);
	                winText.gameObject.SetActive(true);
	                //通知android端显示toast
	                callInfo.callMethodParams.Add("message", "恭喜你，闯关成功！");
	            } else {
	                //通知android端显示toast
	                callInfo.callMethodParams.Add("message", "又得1分，继续加油哦！");
	            }
	            androidCall.OnVoidCall(callInfo);
	        }
	    }
	
	    //---------------------------响应android调用的方法------------------------------
	
	    /// <summary>
	    /// Sets the pause.
	    /// </summary>
	    /// <param name="param">Parameter.</param>
	    public void SetPause(string param) {
	        CallInfo callInfo = JsonMapper.ToObject<CallInfo>(param);
	        this.isPause = Convert.ToBoolean(callInfo.callMethodParams["isPause"]);
	    }
	}

在Start方法里指定自己的名字为”Ball“,有一个SetPause方法来控制游戏暂停。

android端控制游戏暂停的代码就如下所示

	 //在xml文件里指定的onClick
    public void setPause(View view) {
        isPause = !isPause;
        buttonPause.setText(isPause ? "继续" : "暂停");
        CallInfo.Builder
                .create()
                .callModelName("Ball")//对应的unity组件挂在的script文件指定的名字,本demo中对应BallController
                .callMethodName("SetPause")//对应的unity组件挂在的script文件里的方法名字，本demo中对应BallController的SetPause方法
                .addCallMethodParam("isPause", isPause)////对应的unity组件挂在的script文件里的方法需要的参数
                .build()
                .send();
    }






unity端调用android端的代码，参见上面BallController里的OnTriggerEnter方法
	    

	void OnTriggerEnter(Collider other) {
        if (other.gameObject.CompareTag("Coin")) {
            CallInfo callInfo = new CallInfo {
                callMethodName = "showToast"
            };
            other.gameObject.SetActive(false);
            count++;
            countText.text = "Coins collected: " + count;
            if (count == 9) {
                gameObject.SetActive(false);
                winText.gameObject.SetActive(true);
                //通知android端显示toast
                callInfo.callMethodParams.Add("message", "恭喜你，闯关成功！");
            } else {
                //通知android端显示toast
                callInfo.callMethodParams.Add("message", "又得1分，继续加油哦！");
            }
            androidCall.OnVoidCall(callInfo);
        }
    }


android端响应unity调用的代码如下，注意switch里的showToast，和BallController里的OnTriggerEnter方法里指定的methodName是一致的

	//unity3d发送过来的消息，不需要返回值
    @Override
    public void onVoidCall(@NonNull ICallInfo callInfo) {
        switch (callInfo.getCallMethodName()) {
            case "showToast":
                showToast(callInfo.getCallMethodParams().getString("message"));
                break;

            default:
                break;
        }
    }
    
    /**
     * 显示一个toast
     *
     * @param message
     */
    private void showToast(String message) {
        Toast.makeText(this, "来自Unity的消息: " + message, Toast.LENGTH_SHORT).show();
    }


至于全部的代码，大家可以去我的github看看整个工程的源码，比在这里看这些片段要容易理解得多。


## 2.2 适当的偷一下懒

本来这里要继续讲（吹）解（bi）android端对unity sdk封装的相关问题的，但是我感觉写到这里，篇幅似乎有点过长（贴代码贴得多^_^），我怕很难有人愿意坚持看下去，所在这里就不在继续讲（吹）解（bi）了。其实大家去看了源码的话，看不看我我接下来的文章也没有多大意义了，因为实在是太简单了。迫使我想继续写下去的唯一理由就是我想把当时封装unity sdk时的一些思考分享给大家，让大家真正的理解我为什么会那样去做，而不是直接引用了这个库或者只是翻了翻源码，然后就import到你们的工程开始使用。我希望的是大家可以散发思路，写出更优秀的unity与android通信的中间件，因为就目前来说，这方面的开源资料还是比较少的，希望大家一起来完善它，谢谢。

