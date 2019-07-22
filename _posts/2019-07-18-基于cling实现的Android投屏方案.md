---
layout:     post
title:      基于cling实现的Android投屏方案
subtitle:   Android投屏方案
date:       2019-07-18
author:     YANKEBIN
catalog: true
tags:
     - Android
     - cling
     - Android投屏
     - Android DLNA
     - DLAN
     - UPNP
---

# 一 、前言

最近做了一个浏览器&视频播放的项目，是在73.0.3683.90版本的chrome源码上修改而来，涉及到抓取网页里视频的播放地址、播放视频、视频投屏、视频下载、网页内广告屏蔽等方面，了解到ijkplayer、GSYVideoPlayer、ffmpeg、乐播投屏、cling、NanoHttp、adblock等相关技术，现在就准备花点时间把一些技术相关的内容整理一下，分享给大家。

为什么先写的是投屏相关的技术呢？刚开始投屏用的乐播的sdk，乐播的效果肯定是很好的，支持的协议更多，更稳定，但是乐播有一个限制，个人开发者不能获取到APPID和SDK资源，最开始是帮别人做的项目，他们提供了相关的资源，所以就没有去研究过投屏的其他方案。但是后来又有了个新项目，新项目也有一个需求是投屏，但是他们没法提供相关的APPID和SDK，所以我就只能找新的方案，它就是cling。

android相关的投屏方案封装不止cling一个，只是恰巧看到了，并且有人说cling算是封装的比较好的了，所以就直接选择了cling开始做。截止目前，我做的这个项目基本上能正常的投屏图片、音频、视频等资源了，至于控制功能暂时还未尝试，但是相关的方法是有的，只是没有尝试调用。因为需求不同，所以目前我只研究了发送端的功能，至于接收端，我给的参考链接的最后两个链接里是有代码可以参考的。

本来说到投屏技术，一般都会讲到DLNA、AirPlay、UPNP协议等相关基础，但是这方面的介绍文献实在是多如牛毛，我就不在这里浪费时间去复制粘贴别人的劳动成果了，我给出几个当时我找资料时参考的几篇文章，供大家参考：

>[Android手机投屏](https://www.jianshu.com/p/bc8a7ecfd083)

>[cling源码解析](https://alleniverson.gitbooks.io/android-source-analysis/chapter3/Cling%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.html)

>[投屏Cling DLNA 播放本地/网络资源方法梳理](https://www.jianshu.com/p/5000abce2de6)

>[我demo参考的github源码](https://github.com/WilburLe/WClingDemo)


本着大家都是着重于“取而用之”的实际需求，这里先附上本次项目的源码

>[基于cling实现的Android投屏方案](https://github.com/ykbjson/simpledlna/tree/master)


# 二 、实现的过程

我这个人呢，有个特别不好的习惯，不是十分喜欢直接抄袭别人的东西，又喜欢重复造轮子，但是呢，能力又有限，所以写出来的东西会和参考的东西有所区别，但是不一定比别人的好，请大家不要见怪。但这次重复造轮子的原因，主要是因为那个demo里的代码我没办法直接用，以及要解决cling2.2.0版本在9.0系统上出现无法解析描述文件的问题。

整个工程的目录结构如下图所示

![](https://raw.githubusercontent.com/ykbjson/ykbjson.github.io/master/blogimage/simpledlna/simpledlna_code_structure.png)

## 2.1源码浅析前的说明

webserver这个module就是基于NanoHttp实现的本地http服务器的代码。

simplepermission整个module是一个权限请求的库，因为整个工程基于androidx，没花时间去找适配androidx的权限库，就自己改吧改吧了一下原来用的一个权限库来用，因为要实现投屏，必须要一些权限，参见screening module的manifest文件。

sereening module是整个项目的核心，有三个地方要先提出来说清楚，一个是log包下的AndroidLoggingHandler，这个类是为了解决cling包里的logger不输出日志的问题，具体的请看

>[How to configure java.util.logging on Android?](https://stackoverflow.com/questions/4561345/how-to-configure-java-util-logging-on-android/9047282#9047282)

另一个是xml包下的几个类，主要是重写了cling里解析设备交互报文的SAX解析器，cling原来的代码，在生成解析器的时候抛了异常，导致设备交互的报文无法被解析，后续流程就中断了，以至于无法发现可以投屏的设备。说到这里，不得不说，大神们写的代码，设计的真的非常强大，扩展性考虑的很好，我本以为只能clone cling的源码下来自己改，没想到这个解析器可以自定义，为作者手动点赞！

最后一个地方呢，就是DLNABrowserService，里面只是重载了AndroidUpnpServiceImpl的一个方法，返回DLNAUDA10ServiceDescriptorBinderSAXImpl，以便于替换cling自带的无法在android9.0上面正常工作的UDA10ServiceDescriptorBinderSAXImpl。所以，在使用这个库的时候，在app module的manifest里声明的就不是AndroidUpnpServiceImpl而是DLNABrowserService，这一点要注意。

至于bean包下的两个类，DeviceInfo是对支持投屏的设备——Device 的一个封装；MediaInfo是为了方便传递要投屏的多媒体信息做的封装。

## 2.2部分源码浅析

接下来我们从listener包开始讲解整个项目的源码，里面有四个回调接口，其实我感觉有些是多余的，但是呢，因为一些操作是异步的，感觉有一个回调接口能更好的控制使用这个库的逻辑，避免出现一些错误。

###初始化DLNAManager回调接口——DLNAStateCallback

	public interface DLNAStateCallback {

	    void onConnected();
	
	    void onDisconnected();

	}


这个其实应该叫DLNAManagerInitCallback，初始化DLNAManager的时候传递的，可以为null，只要你能保证你后续代码时在DLNAManager初始化之后调用的。

###注册设备列表和状态回调接口——DLNARegistryListener

	public abstract class DLNARegistryListener implements RegistryListener {
	    private final DeviceType DMR_DEVICE_TYPE = new UDADeviceType("MediaRenderer");
	
	    public void remoteDeviceDiscoveryStarted(Registry registry, RemoteDevice device) {
	
	    }
	
	    public void remoteDeviceDiscoveryFailed(Registry registry, RemoteDevice device, Exception ex) {
	
	    }
	
	    /**
	     * Calls the {@link #onDeviceChanged(List)} method.
	     *
	     * @param registry The Cling registry of all devices and services know to the local UPnP stack.
	     * @param device   A validated and hydrated device metadata graph, with complete service metadata.
	     */
	    public void remoteDeviceAdded(Registry registry, RemoteDevice device) {
	        onDeviceChanged(build(registry.getDevices()));
	        onDeviceAdded(registry, device);
	    }
	
	    public void remoteDeviceUpdated(Registry registry, RemoteDevice device) {
	
	    }
	
	    /**
	     * Calls the {@link #onDeviceChanged(List)} method.
	     *
	     * @param registry The Cling registry of all devices and services know to the local UPnP stack.
	     * @param device   A validated and hydrated device metadata graph, with complete service metadata.
	     */
	    public void remoteDeviceRemoved(Registry registry, RemoteDevice device) {
	        onDeviceChanged(build(registry.getDevices()));
	        onDeviceRemoved(registry, device);
	    }
	
	    /**
	     * Calls the {@link #onDeviceChanged(List)} method.
	     *
	     * @param registry The Cling registry of all devices and services know to the local UPnP stack.
	     * @param device   The local device added to the {@link org.fourthline.cling.registry.Registry}.
	     */
	    public void localDeviceAdded(Registry registry, LocalDevice device) {
	        onDeviceChanged(build(registry.getDevices()));
	        onDeviceAdded(registry, device);
	    }
	
	    /**
	     * Calls the {@link #onDeviceChanged(List)} method.
	     *
	     * @param registry The Cling registry of all devices and services know to the local UPnP stack.
	     * @param device   The local device removed from the {@link org.fourthline.cling.registry.Registry}.
	     */
	    public void localDeviceRemoved(Registry registry, LocalDevice device) {
	        onDeviceChanged(build(registry.getDevices()));
	        onDeviceRemoved(registry, device);
	    }
	
	    public void beforeShutdown(Registry registry) {
	
	    }
	
	    public void afterShutdown() {
	
	    }
	
	    public void onDeviceChanged(Collection<Device> deviceInfoList) {
	        onDeviceChanged(build(deviceInfoList));
	    }
	
	    public abstract void onDeviceChanged(List<DeviceInfo> deviceInfoList);
	
	    public void onDeviceAdded(Registry registry, Device device) {
	
	    }
	
	    public void onDeviceRemoved(Registry registry, Device device) {
	
	    }
	
	    private List<DeviceInfo> build(Collection<Device> deviceList) {
	        final List<DeviceInfo> deviceInfoList = new ArrayList<>();
	        for (Device device : deviceList) {
	            //过滤不支持投屏渲染的设备
	            if (null == device.findDevices(DMR_DEVICE_TYPE)) {
	                continue;
	            }
	            final DeviceInfo deviceInfo = new DeviceInfo(device, getDeviceName(device));
	            deviceInfoList.add(deviceInfo);
	        }
	
	        return deviceInfoList;
	    }
	
	    private String getDeviceName(Device device) {
	        String name = "";
	        if (device.getDetails() != null && device.getDetails().getFriendlyName() != null) {
	            name = device.getDetails().getFriendlyName();
	        } else {
	            name = device.getDisplayString();
	        }
	
	        return name;
	    }
	}

这个类只是对RegistryListener的封装，因为我当时想着这个类主要是回调当前发现的设备的列表信息，所以就简单封装了一下，每次设备数量改变的时候就把新的设备数量通过一个回调方法传递出去，忽略一些不关注的方法。


###连接设备回调接口——DLNADeviceConnectListener

	public interface DLNADeviceConnectListener {
	
	    int TYPE_DLNA = 1;
	    int TYPE_IM = 2;
	    int TYPE_NEW_LELINK = 3;
	    int CONNECT_INFO_CONNECT_SUCCESS = 100000;
	    int CONNECT_INFO_CONNECT_FAILURE = 100001;
	    int CONNECT_INFO_DISCONNECT = 212000;
	    int CONNECT_INFO_DISCONNECT_SUCCESS = 212001;
	    int CONNECT_ERROR_FAILED = 212010;
	    int CONNECT_ERROR_IO = 212011;
	    int CONNECT_ERROR_IM_WAITTING = 212012;
	    int CONNECT_ERROR_IM_REJECT = 212013;
	    int CONNECT_ERROR_IM_TIMEOUT = 212014;
	    int CONNECT_ERROR_IM_BLACKLIST = 212015;
	
	    void onConnect(DeviceInfo deviceInfo, int errorCode);
	
	    void onDisconnect(DeviceInfo deviceInfo,int type,int errorCode);
	}
	
这个类是给DLNAPlayer连接设备时用的。说到这个所谓的连接设备，其实感觉也不需要这个步骤，cling本身可能已经做好了设备之间的连接，回调回来的设备列表里的设备都是连接过了的，直接可以通信。但是我发现乐播的sdk里就有一个连接设备的方法，必须先调用连接设备的这个方法，在回调里才能继续后续操作，所以我这里也设计了一个连接设备的步骤，我怕万一是cling有专门连接设备的接口，只是我还没发现而已，后面发现了就来改写这个连接设备的方法。


###控制设备回调接口——DLNAControlCallback

	public interface DLNAControlCallback {
	    int ERROR_CODE_NO_ERROR = 0;
	
	    int ERROR_CODE_RE_PLAY = 1;
	
	    int ERROR_CODE_RE_PAUSE = 2;
	
	    int ERROR_CODE_RE_STOP = 3;
	
	    int ERROR_CODE_DLNA_ERROR = 4;
	
	    int ERROR_CODE_SERVICE_ERROR = 5;
	
	    int ERROR_CODE_NOT_READY = 6;
	
	
	    void onSuccess(@Nullable ActionInvocation invocation);
	
	    void onReceived(@Nullable ActionInvocation invocation,@Nullable Object ... extra);
	
	    void onFailure(@Nullable ActionInvocation invocation,
	                   @IntRange(from = ERROR_CODE_NO_ERROR, to = ERROR_CODE_NOT_READY) int errorCode,
	                   @Nullable String errorMsg);
	}

顾名思义，这个类就是发送端在控制接收端做出一系列动作时的回调接口，包括播放、暂停、结束、静音开闭、音量调整、播放进度获取等等。播放、暂停、结束、静音开闭、音量调整等方法只会回调onSuccess和onFailure方法；获取播放进度这种需要获取结果的方法会在onReceived方法里返回结果。


看完这几个类之后，我们应该大致知道这个库整个工作的流程了：初始化DLNAManager -> 注册设备列表回调接口 -> 连接一个设备 -> 控制这个设备。只不过呢，我把连接设备和控制设备部分功能封装到了DLNAPlayer里面，不然DLNAManager会有点臃肿，不便于维护。这里说到了整个库的工作流程，那么接下来我们就从DLNAManager开始接着分析。

###整个库的入口——DLNAManager

	public final class DLNAManager {
	    private static final String TAG = "DLNAManager";
	    private static final String LOCAL_HTTP_SERVER_PORT = "9090";
	
	    private static boolean isDebugMode = false;
	
	    private Context mContext;
	    private AndroidUpnpService mUpnpService;
	    private ServiceConnection mServiceConnection;
	    private DLNAStateCallback mStateCallback;
	
	    private RegistryListener mRegistryListener;
	    private List<DLNARegistryListener> registryListenerList;
	    private Handler mHandler;
	    private BroadcastReceiver mBroadcastReceiver;
	
	    private DLNAManager() {
	        AndroidLoggingHandler.injectJavaLogger();
	        mHandler = new Handler(Looper.getMainLooper());
	        registryListenerList = new ArrayList<>();
	        mRegistryListener = new RegistryListener() {
	
	            @Override
	            public void remoteDeviceDiscoveryStarted(final Registry registry, final RemoteDevice device) {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.remoteDeviceDiscoveryStarted(registry, device);
	                        }
	                    }
	                });
	            }
	
	            @Override
	            public void remoteDeviceDiscoveryFailed(final Registry registry, final RemoteDevice device, final Exception ex) {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.remoteDeviceDiscoveryFailed(registry, device, ex);
	                        }
	                    }
	                });
	            }
	
	            @Override
	            public void remoteDeviceAdded(final Registry registry, final RemoteDevice device) {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.remoteDeviceAdded(registry, device);
	                        }
	                    }
	                });
	            }
	
	            @Override
	            public void remoteDeviceUpdated(final Registry registry, final RemoteDevice device) {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.remoteDeviceUpdated(registry, device);
	                        }
	                    }
	                });
	            }
	
	            @Override
	            public void remoteDeviceRemoved(final Registry registry, final RemoteDevice device) {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.remoteDeviceRemoved(registry, device);
	                        }
	                    }
	                });
	            }
	
	            @Override
	            public void localDeviceAdded(final Registry registry, final LocalDevice device) {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.localDeviceAdded(registry, device);
	                        }
	                    }
	                });
	            }
	
	            @Override
	            public void localDeviceRemoved(final Registry registry, final LocalDevice device) {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.localDeviceRemoved(registry, device);
	                        }
	                    }
	                });
	            }
	
	            @Override
	            public void beforeShutdown(final Registry registry) {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.beforeShutdown(registry);
	                        }
	                    }
	                });
	            }
	
	            @Override
	            public void afterShutdown() {
	                mHandler.post(() -> {
	                    synchronized (DLNAManager.class) {
	                        for (DLNARegistryListener listener : registryListenerList) {
	                            listener.afterShutdown();
	                        }
	                    }
	                });
	            }
	        };
	
	        mBroadcastReceiver = new BroadcastReceiver() {
	            @Override
	            public void onReceive(Context context, Intent intent) {
	                if (null != intent && TextUtils.equals(intent.getAction(), ConnectivityManager.CONNECTIVITY_ACTION)) {
	                    final NetworkInfo networkInfo = getNetworkInfo(context);
	                    if (null == networkInfo) {
	                        return;
	                    }
	                    if (networkInfo.getType() == ConnectivityManager.TYPE_WIFI) {
	                        initLocalMediaServer();
	                    }
	                }
	            }
	        };
	    }
	
	    private static class DLNAManagerCreator {
	        private static DLNAManager manager = new DLNAManager();
	    }
	
	    public static DLNAManager getInstance() {
	        return DLNAManagerCreator.manager;
	    }
	
	    public void init(@NonNull Context context) {
	        init(context, null);
	    }
	
	    public void init(@NonNull Context context, @Nullable DLNAStateCallback stateCallback) {
	        if (null != mContext) {
	            logW("ReInit DLNAManager");
	            return;
	        }
	        if (context instanceof ContextThemeWrapper || context instanceof android.view.ContextThemeWrapper) {
	            mContext = context.getApplicationContext();
	        } else {
	            mContext = context;
	        }
	        mStateCallback = stateCallback;
	        initLocalMediaServer();
	        initConnection();
	        registerBroadcastReceiver();
	    }
	
	    private void initConnection() {
	        mServiceConnection = new ServiceConnection() {
	            @Override
	            public void onServiceConnected(ComponentName name, IBinder service) {
	                mUpnpService = (AndroidUpnpService) service;
	                mUpnpService.getRegistry().addListener(mRegistryListener);
	                mUpnpService.getControlPoint().search();
	                if (null != mStateCallback) {
	                    mStateCallback.onConnected();
	                }
	                logD("onServiceConnected");
	            }
	
	            @Override
	            public void onServiceDisconnected(ComponentName name) {
	                mUpnpService = null;
	                if (null != mStateCallback) {
	                    mStateCallback.onDisconnected();
	                }
	                logD("onServiceDisconnected");
	            }
	        };
	
	        mContext.bindService(new Intent(mContext, DLNABrowserService.class),
	                mServiceConnection, Context.BIND_AUTO_CREATE);
	    }
	
	    /**
	     * 本地视频和图片也可以直接投屏，根目录为sd卡根目录
	     */
	    private void initLocalMediaServer() {
	        checkConfig();
	        try {
	            final PipedOutputStream pipedOutputStream = new PipedOutputStream();
	            System.setIn(new PipedInputStream(pipedOutputStream));
	            new Thread(() -> {
	                final String localIpAddress = getLocalIpStr(mContext);
	                final String localMediaRootPath = Environment.getExternalStorageDirectory().getAbsolutePath();
	                String[] args = {
	                        "--host",
	                        localIpAddress,/*局域网ip地址*/
	                        "--port",
	                        LOCAL_HTTP_SERVER_PORT,/*局域网端口*/
	                        "--dir",
	                        localMediaRootPath/*下载视频根目录*/
	                };
	                SimpleWebServer.startServer(args);
	                logD("initLocalLinkService success,localIpAddress : " + localIpAddress +
	                        ",localVideoRootPath : " + localMediaRootPath);
	            }).start();
	        } catch (IOException e) {
	            e.printStackTrace();
	            logE("initLocalLinkService failure", e);
	        }
	    }
	
	    private void registerBroadcastReceiver() {
	        checkConfig();
	        mContext.registerReceiver(mBroadcastReceiver,
	                new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION));
	    }
	
	    private void unregisterBroadcastReceiver() {
	        checkConfig();
	        mContext.unregisterReceiver(mBroadcastReceiver);
	    }
	
	    public void registerListener(DLNARegistryListener listener) {
	        checkConfig();
	        checkPrepared();
	        if (null == listener) {
	            return;
	        }
	        registryListenerList.add(listener);
	        listener.onDeviceChanged(mUpnpService.getRegistry().getDevices());
	    }
	
	    public void unregisterListener(DLNARegistryListener listener) {
	        checkConfig();
	        checkPrepared();
	        if (null == listener) {
	            return;
	        }
	        mUpnpService.getRegistry().removeListener(listener);
	        registryListenerList.remove(listener);
	    }
	
	    public void startBrowser() {
	        checkConfig();
	        checkPrepared();
	        mUpnpService.getRegistry().addListener(mRegistryListener);
	        mUpnpService.getControlPoint().search();
	    }
	
	    public void stopBrowser() {
	        checkConfig();
	        checkPrepared();
	        mUpnpService.getRegistry().removeListener(mRegistryListener);
	    }
	
	    public void destroy() {
	        checkConfig();
	        registryListenerList.clear();
	        unregisterBroadcastReceiver();
	        SimpleWebServer.stopServer();
	        stopBrowser();
	        if (null != mUpnpService) {
	            mUpnpService.getRegistry().removeListener(mRegistryListener);
	            mUpnpService.getRegistry().shutdown();
	        }
	        if (null != mServiceConnection) {
	            mContext.unbindService(mServiceConnection);
	            mServiceConnection = null;
	        }
	        if (null != mHandler) {
	            mHandler.removeCallbacksAndMessages(null);
	            mHandler = null;
	        }
	        registryListenerList = null;
	        mRegistryListener = null;
	        mBroadcastReceiver = null;
	        mStateCallback = null;
	        mContext = null;
	    }
	
	    private void checkConfig() {
	        if (null == mContext) {
	            throw new IllegalStateException("Must call init(Context context) at first");
	        }
	    }
	
	    private void checkPrepared() {
	        if (null == mUpnpService) {
	            throw new IllegalStateException("Invalid AndroidUpnpService");
	        }
	    }
	
	    //------------------------------------------------------静态方法-----------------------------------------------
	
	    /**
	     * 获取ip地址
	     *
	     * @param context
	     * @return
	     */
	    public static String getLocalIpStr(@NonNull Context context) {
	        WifiManager wifiManager = (WifiManager) context.getSystemService(Context.WIFI_SERVICE);
	        WifiInfo wifiInfo = wifiManager.getConnectionInfo();
	        if (null == wifiInfo) {
	            return "";
	        }
	        return intToIpAddress(wifiInfo.getIpAddress());
	    }
	
	    /**
	     * int类型的ip转换成标准ip地址
	     *
	     * @param ip
	     * @return
	     */
	    public static String intToIpAddress(int ip) {
	        return (ip & 0xff) + "." + ((ip >> 8) & 0xff) + "." + ((ip >> 16) & 0xff) + "." + ((ip >> 24) & 0xff);
	    }
	
	    public static NetworkInfo getNetworkInfo(@NonNull Context context) {
	        final ConnectivityManager connectivityManager = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
	        return null == connectivityManager ? null : connectivityManager.getActiveNetworkInfo();
	    }
	
	    static String tryTransformLocalMediaAddressToLocalHttpServerAddress(@NonNull Context context,
	                                                                        String sourceUrl) {
	        logD("tryTransformLocalMediaAddressToLocalHttpServerAddress ,sourceUrl : " + sourceUrl);
	        if (TextUtils.isEmpty(sourceUrl)) {
	            return sourceUrl;
	        }
	
	        if (!isLocalMediaAddress(sourceUrl)) {
	            return sourceUrl;
	        }
	
	        String newSourceUrl = getLocalHttpServerAddress(context) +
	                sourceUrl.replace(Environment.getExternalStorageDirectory().getAbsolutePath(), "");
	        logD("tryTransformLocalMediaAddressToLocalHttpServerAddress ,newSourceUrl : " + newSourceUrl);
	
	        try {
	            final String[] urlSplits = newSourceUrl.split("/");
	            final String originFileName = urlSplits[urlSplits.length - 1];
	            String fileName = originFileName;
	            fileName = URLEncoder.encode(fileName, "UTF-8");
	            fileName = fileName.replaceAll("\\+", "%20");
	            newSourceUrl = newSourceUrl.replace(originFileName, fileName);
	            logD("tryTransformLocalMediaAddressToLocalHttpServerAddress ,encodeNewSourceUrl : " + newSourceUrl);
	        } catch (UnsupportedEncodingException e) {
	            e.printStackTrace();
	        }
	
	        return newSourceUrl;
	    }
	
	    private static boolean isLocalMediaAddress(String sourceUrl) {
	        return !TextUtils.isEmpty(sourceUrl)
	                && !sourceUrl.startsWith("http://")
	                && !sourceUrl.startsWith("https://")
	                && sourceUrl.startsWith(Environment.getExternalStorageDirectory().getAbsolutePath());
	    }
	
	    /**
	     * 获取本地http服务器地址
	     *
	     * @param context
	     * @return
	     */
	    public static String getLocalHttpServerAddress(Context context) {
	        return "http://" + getLocalIpStr(context) + ":" + LOCAL_HTTP_SERVER_PORT;
	    }
	
	    public static void setIsDebugMode(boolean isDebugMode) {
	        DLNAManager.isDebugMode = isDebugMode;
	    }
	
	
	    static void logV(String content) {
	        logV(TAG, content);
	    }
	
	    public static void logV(String tag, String content) {
	        if (!isDebugMode) {
	            return;
	        }
	        Log.v(tag, content);
	    }
	
	    static void logD(String content) {
	        logD(TAG, content);
	    }
	
	    public static void logD(String tag, String content) {
	        if (!isDebugMode) {
	            return;
	        }
	        Log.d(tag, content);
	    }
	
	
	    static void logI(String content) {
	        logI(TAG, content);
	    }
	
	    public static void logI(String tag, String content) {
	        if (!isDebugMode) {
	            return;
	        }
	        Log.i(tag, content);
	    }
	
	
	    static void logW(String content) {
	        logW(TAG, content);
	    }
	
	    public static void logW(String tag, String content) {
	        if (!isDebugMode) {
	            return;
	        }
	        Log.w(tag, content);
	    }
	
	
	    static void logE(String content) {
	        logE(TAG, content);
	    }
	
	
	    public static void logE(String tag, String content) {
	        logE(tag, content, null);
	    }
	
	
	    static void logE(String content, Throwable throwable) {
	        logE(TAG, content, throwable);
	    }
	
	    public static void logE(String tag, String content, Throwable throwable) {
	        if (!isDebugMode) {
	            return;
	        }
	        if (null != throwable) {
	            Log.e(tag, content, throwable);
	        } else {
	            Log.e(tag, content);
	        }
	    }
	}

这个类有点长，但是要关注的方法就那么几个。init方法里干了几件事：

1.初始化本地投屏服务——initLocalMediaServer，投屏本地视频

2.连接AndroidUpnpService——initConnection，获取控制点和投屏服务

3.注册了一个网络连接变化的广播——registerBroadcastReceiver，网络变化时重启LocalMediaServer，保证本地资源投屏成功的几率

还有就是发起搜索设备的动作、停止搜索设备的动作、注册RegistryListener、移除RegistryListener等方法。剩下一些就是可以封装到工具类里的方法，懒得在添加类了，索性就写到了里面。

这个类还有一个作用就是维护了一个RegistryListener，统一的分发局域网内设备数量、设备状态、设备服务状态变化的回调事件。当你初始化完DLNAManager，并向这个类注册了DLNARegistryListener，然后调用startBrowser发起搜索，如果局域网内有可以接受投屏的设备，你就可以在DLNARegistryListener的onDeviceChanged方法里收到当前局域网内可以投屏的设备列表了。有了可用的设备列表，接下来，我们就可以开始连接接收端设备发送投屏数据以及控制他了。

### 连接和控制接收端设备——DLNAPlayer

	public class DLNAPlayer {
	
	    private static final String DIDL_LITE_FOOTER = "</DIDL-Lite>";
	    private static final String DIDL_LITE_HEADER = "<?xml version=\"1.0\" encoding=\"utf-8\" standalone=\"no\"?>"
	            + "<DIDL-Lite "
	            + "xmlns=\"urn:schemas-upnp-org:metadata-1-0/DIDL-Lite/\" "
	            + "xmlns:dc=\"http://purl.org/dc/elements/1.1/\" "
	            + "xmlns:upnp=\"urn:schemas-upnp-org:metadata-1-0/upnp/\" "
	            + "xmlns:dlna=\"urn:schemas-dlna-org:metadata-1-0/\">";
	
	    /**
	     * 未知状态
	     */
	    public static final int UNKNOWN = -1;
	
	    /**
	     * 已连接状态
	     */
	    public static final int CONNECTED = 0;
	
	    /**
	     * 播放状态
	     */
	    public static final int PLAY = 1;
	    /**
	     * 暂停状态
	     */
	    public static final int PAUSE = 2;
	    /**
	     * 停止状态
	     */
	    public static final int STOP = 3;
	    /**
	     * 转菊花状态
	     */
	    public static final int BUFFER = 4;
	    /**
	     * 投放失败
	     */
	    public static final int ERROR = 5;
	
	    /**
	     * 已断开状态
	     */
	    public static final int DISCONNECTED = 6;
	
	    private int currentState = UNKNOWN;
	    private DeviceInfo mDeviceInfo;
	    private Device mDevice;
	    private MediaInfo mMediaInfo;
	    private Context mContext;//鉴权预留
	    private ServiceConnection mServiceConnection;
	    private AndroidUpnpService mUpnpService;
	    private DLNADeviceConnectListener connectListener;
	    /**
	     * 连接、控制服务
	     */
	    private ServiceType AV_TRANSPORT_SERVICE;
	    private ServiceType RENDERING_CONTROL_SERVICE;
	
	
	    public DLNAPlayer(@NonNull Context context) {
	        mContext = context;
	        AV_TRANSPORT_SERVICE = new UDAServiceType("AVTransport");
	        RENDERING_CONTROL_SERVICE = new UDAServiceType("RenderingControl");
	        initConnection();
	    }
	
	    public void setConnectListener(DLNADeviceConnectListener connectListener) {
	        this.connectListener = connectListener;
	    }
	
	    private void initConnection() {
	        mServiceConnection = new ServiceConnection() {
	            @Override
	            public void onServiceConnected(ComponentName name, IBinder service) {
	                mUpnpService = (AndroidUpnpService) service;
	                currentState = CONNECTED;
	                if (null != mDeviceInfo) {
	                    mDeviceInfo.setState(CONNECTED);
	                    mDeviceInfo.setConnected(true);
	                }
	                if (null != connectListener) {
	                    connectListener.onConnect(mDeviceInfo, DLNADeviceConnectListener.CONNECT_INFO_CONNECT_SUCCESS);
	                }
	            }
	
	            @Override
	            public void onServiceDisconnected(ComponentName name) {
	                currentState = DISCONNECTED;
	                if (null != mDeviceInfo) {
	                    mDeviceInfo.setState(DISCONNECTED);
	                    mDeviceInfo.setConnected(false);
	                }
	                if (null != connectListener) {
	                    connectListener.onDisconnect(mDeviceInfo, DLNADeviceConnectListener.TYPE_DLNA,
	                            DLNADeviceConnectListener.CONNECT_INFO_DISCONNECT_SUCCESS);
	                }
	                mUpnpService = null;
	                connectListener = null;
	                mDeviceInfo = null;
	                mDevice = null;
	                mMediaInfo = null;
	                AV_TRANSPORT_SERVICE = null;
	                RENDERING_CONTROL_SERVICE = null;
	                mServiceConnection = null;
	                mContext = null;
	            }
	        };
	    }
	
	    public void connect(@NonNull DeviceInfo deviceInfo) {
	        checkConfig();
	        mDeviceInfo = deviceInfo;
	        mDevice = mDeviceInfo.getDevice();
	        if (null != mUpnpService) {
	            currentState = CONNECTED;
	            if (null != connectListener) {
	                connectListener.onConnect(mDeviceInfo, DLNADeviceConnectListener.CONNECT_INFO_CONNECT_SUCCESS);
	            }
	            return;
	        }
	        mContext.bindService(new Intent(mContext, DLNABrowserService.class),
	                mServiceConnection, Context.BIND_AUTO_CREATE);
	    }
	
	
	    public void disconnect() {
	        checkConfig();
	        try {
	            mContext.unbindService(mServiceConnection);
	        } catch (Exception e) {
	            DLNAManager.logE("DLNAPlayer disconnect error.", e);
	        }
	    }
	
	    private void checkPrepared() {
	        if (null == mUpnpService) {
	            throw new IllegalStateException("Invalid AndroidUpnpService");
	        }
	    }
	
	    private void checkConfig() {
	        if (null == mContext) {
	            throw new IllegalStateException("Invalid context");
	        }
	    }
	
	    private void execute(@NonNull ActionCallback actionCallback) {
	        checkPrepared();
	        mUpnpService.getControlPoint().execute(actionCallback);
	
	    }
	
	    private void execute(@NonNull SubscriptionCallback subscriptionCallback) {
	        checkPrepared();
	        mUpnpService.getControlPoint().execute(subscriptionCallback);
	    }
	
	    public void play(@NonNull DLNAControlCallback callback) {
	        final Service avtService = mDevice.findService(AV_TRANSPORT_SERVICE);
	        if (checkErrorBeforeExecute(PLAY, avtService, callback)) {
	            return;
	        }
	        execute(new Play(avtService) {
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                currentState = PLAY;
	                callback.onSuccess(invocation);
	                mDeviceInfo.setState(PLAY);
	            }
	
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                currentState = ERROR;
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	                mDeviceInfo.setState(ERROR);
	            }
	        });
	    }
	
	    public void pause(@NonNull DLNAControlCallback callback) {
	        final Service avtService = mDevice.findService(AV_TRANSPORT_SERVICE);
	        if (checkErrorBeforeExecute(PAUSE, avtService, callback)) {
	            return;
	        }
	
	        execute(new Pause(avtService) {
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                currentState = PAUSE;
	                callback.onSuccess(invocation);
	                mDeviceInfo.setState(PAUSE);
	            }
	
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                currentState = ERROR;
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	                mDeviceInfo.setState(ERROR);
	            }
	        });
	    }
	
	
	    public void stop(@NonNull DLNAControlCallback callback) {
	        final Service avtService = mDevice.findService(AV_TRANSPORT_SERVICE);
	        if (checkErrorBeforeExecute(STOP, avtService, callback)) {
	            return;
	        }
	        execute(new Stop(avtService) {
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                currentState = STOP;
	                callback.onSuccess(invocation);
	                mDeviceInfo.setState(STOP);
	            }
	
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                currentState = ERROR;
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	                mDeviceInfo.setState(ERROR);
	            }
	        });
	    }
	
	    public void seekTo(String time, @NonNull DLNAControlCallback callback) {
	        final Service avtService = mDevice.findService(AV_TRANSPORT_SERVICE);
	        if (checkErrorBeforeExecute(avtService, callback)) {
	            return;
	        }
	        execute(new Seek(avtService, time) {
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                callback.onSuccess(invocation);
	            }
	
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                currentState = ERROR;
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	                mDeviceInfo.setState(ERROR);
	            }
	        });
	    }
	
	    public void setVolume(long volume, @NonNull DLNAControlCallback callback) {
	        final Service avtService = mDevice.findService(RENDERING_CONTROL_SERVICE);
	        if (checkErrorBeforeExecute(avtService, callback)) {
	            return;
	        }
	
	        execute(new SetVolume(avtService, volume) {
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                callback.onSuccess(invocation);
	            }
	
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                currentState = ERROR;
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	                mDeviceInfo.setState(ERROR);
	            }
	        });
	    }
	
	    public void mute(boolean desiredMute, @NonNull DLNAControlCallback callback) {
	        final Service avtService = mDevice.findService(RENDERING_CONTROL_SERVICE);
	        if (checkErrorBeforeExecute(avtService, callback)) {
	            return;
	        }
	        execute(new SetMute(avtService, desiredMute) {
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                callback.onSuccess(invocation);
	            }
	
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                currentState = ERROR;
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	                mDeviceInfo.setState(ERROR);
	            }
	        });
	    }
	
	
	    public void getPositionInfo(@NonNull DLNAControlCallback callback) {
	        final Service avtService = mDevice.findService(AV_TRANSPORT_SERVICE);
	        if (checkErrorBeforeExecute(avtService, callback)) {
	            return;
	        }
	
	        final GetPositionInfo getPositionInfo = new GetPositionInfo(avtService) {
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                currentState = ERROR;
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	                mDeviceInfo.setState(ERROR);
	            }
	
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                callback.onSuccess(invocation);
	            }
	
	            @Override
	            public void received(ActionInvocation invocation, PositionInfo info) {
	                callback.onReceived(invocation, info);
	            }
	        };
	
	        execute(getPositionInfo);
	    }
	
	
	    public void getVolume(@NonNull DLNAControlCallback callback) {
	        final Service avtService = mDevice.findService(AV_TRANSPORT_SERVICE);
	        if (checkErrorBeforeExecute(avtService, callback)) {
	            return;
	        }
	        final GetVolume getVolume = new GetVolume(avtService) {
	
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                callback.onSuccess(invocation);
	            }
	
	            @Override
	            public void received(ActionInvocation invocation, int currentVolume) {
	                callback.onReceived(invocation, currentVolume);
	            }
	
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                currentState = ERROR;
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	                mDeviceInfo.setState(ERROR);
	            }
	        };
	        execute(getVolume);
	    }
	
	
	    public void setDataSource(@NonNull MediaInfo mediaInfo) {
	        mMediaInfo = mediaInfo;
	        //尝试变换本地播放地址
	        mMediaInfo.setUri(DLNAManager.tryTransformLocalMediaAddressToLocalHttpServerAddress(mContext,
	                mMediaInfo.getUri()));
	    }
	
	    public void start(final @NonNull DLNAControlCallback callback) {
	        mDeviceInfo.setMediaID(mMediaInfo.getMediaId());
	        String metadata = pushMediaToRender(mMediaInfo);
	        final Service avtService = mDevice.findService(AV_TRANSPORT_SERVICE);
	        if (null == avtService) {
	            callback.onFailure(null, DLNAControlCallback.ERROR_CODE_SERVICE_ERROR, null);
	            return;
	        }
	        execute(new SetAVTransportURI(avtService, mMediaInfo.getUri(), metadata) {
	            @Override
	            public void success(ActionInvocation invocation) {
	                super.success(invocation);
	                play(callback);
	            }
	
	            @Override
	            public void failure(ActionInvocation invocation, UpnpResponse operation, String defaultMsg) {
	                DLNAManager.logE("play error:" + defaultMsg);
	                currentState = ERROR;
	                mDeviceInfo.setState(ERROR);
	                callback.onFailure(invocation, DLNAControlCallback.ERROR_CODE_DLNA_ERROR, defaultMsg);
	            }
	        });
	    }
	
	
	    private String pushMediaToRender(@NonNull MediaInfo mediaInfo) {
	        return pushMediaToRender(mediaInfo.getUri(), mediaInfo.getMediaId(), mediaInfo.getMediaName(),
	                mediaInfo.getMediaType());
	    }
	
	    private String pushMediaToRender(String url, String id, String name, int ItemType) {
	        final long size = 0;
	        final Res res = new Res(new MimeType(ProtocolInfo.WILDCARD, ProtocolInfo.WILDCARD), size, url);
	        final String creator = "unknow";
	        final String parentId = "0";
	        final String metadata;
	
	        switch (ItemType) {
	            case MediaInfo.TYPE_IMAGE:
	                ImageItem imageItem = new ImageItem(id, parentId, name, creator, res);
	                metadata = createItemMetadata(imageItem);
	                break;
	            case MediaInfo.TYPE_VIDEO:
	                VideoItem videoItem = new VideoItem(id, parentId, name, creator, res);
	                metadata = createItemMetadata(videoItem);
	                break;
	            case MediaInfo.TYPE_AUDIO:
	                AudioItem audioItem = new AudioItem(id, parentId, name, creator, res);
	                metadata = createItemMetadata(audioItem);
	                break;
	            default:
	                throw new IllegalArgumentException("UNKNOWN MEDIA TYPE");
	        }
	
	        DLNAManager.logE("metadata: " + metadata);
	        return metadata;
	    }
	
	    /**
	     * 创建投屏的参数
	     *
	     * @param item
	     * @return
	     */
	    private String createItemMetadata(DIDLObject item) {
	        StringBuilder metadata = new StringBuilder();
	        metadata.append(DIDL_LITE_HEADER);
	
	        metadata.append(String.format("<item id=\"%s\" parentID=\"%s\" restricted=\"%s\">", item.getId(), item.getParentID(), item.isRestricted() ? "1" : "0"));
	
	        metadata.append(String.format("<dc:title>%s</dc:title>", item.getTitle()));
	        String creator = item.getCreator();
	        if (creator != null) {
	            creator = creator.replaceAll("<", "_");
	            creator = creator.replaceAll(">", "_");
	        }
	        metadata.append(String.format("<upnp:artist>%s</upnp:artist>", creator));
	        metadata.append(String.format("<upnp:class>%s</upnp:class>", item.getClazz().getValue()));
	
	        DateFormat sdf = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss");
	        Date now = new Date();
	        String time = sdf.format(now);
	        metadata.append(String.format("<dc:date>%s</dc:date>", time));
	
	        Res res = item.getFirstResource();
	        if (res != null) {
	            // protocol info
	            String protocolinfo = "";
	            ProtocolInfo pi = res.getProtocolInfo();
	            if (pi != null) {
	                protocolinfo = String.format("protocolInfo=\"%s:%s:%s:%s\"", pi.getProtocol(), pi.getNetwork(), pi.getContentFormatMimeType(), pi
	                        .getAdditionalInfo());
	            }
	            DLNAManager.logE("protocolinfo: " + protocolinfo);
	
	            // resolution, extra info, not adding yet
	            String resolution = "";
	            if (res.getResolution() != null && res.getResolution().length() > 0) {
	                resolution = String.format("resolution=\"%s\"", res.getResolution());
	            }
	
	            // duration
	            String duration = "";
	            if (res.getDuration() != null && res.getDuration().length() > 0) {
	                duration = String.format("duration=\"%s\"", res.getDuration());
	            }
	
	            // res begin
	            //            metadata.append(String.format("<res %s>", protocolinfo)); // no resolution & duration yet
	            metadata.append(String.format("<res %s %s %s>", protocolinfo, resolution, duration));
	
	            // url
	            String url = res.getValue();
	            metadata.append(url);
	
	            // res end
	            metadata.append("</res>");
	        }
	        metadata.append("</item>");
	
	        metadata.append(DIDL_LITE_FOOTER);
	
	        return metadata.toString();
	    }
	
	    private boolean checkErrorBeforeExecute(int expectState, Service avtService, @NonNull DLNAControlCallback callback) {
	        if (currentState == expectState) {
	            callback.onSuccess(null);
	            return true;
	        }
	
	        return checkErrorBeforeExecute(avtService, callback);
	    }
	
	    private boolean checkErrorBeforeExecute(Service avtService, @NonNull DLNAControlCallback callback) {
	        if (currentState == UNKNOWN) {
	            callback.onFailure(null, DLNAControlCallback.ERROR_CODE_NOT_READY, null);
	            return true;
	        }
	
	        if (null == avtService) {
	            callback.onFailure(null, DLNAControlCallback.ERROR_CODE_SERVICE_ERROR, null);
	            return true;
	        }
	
	        return false;
	    }
	
	}


这个类也很长，因为干事情的就是他，所以他的方法比较多，设定播放数据、播放、暂停、停止、拖动进度、静音控制、音量控制等等都在这个DLNAPlayer里实现的。cling对设定投屏数据、播放、暂停、停止、拖动进度、静音控制、音量控制等功能都做了封装，我这里只是统一了一个回调接口，这些个方法里，只有设定投屏数据的时候才需要发送upnp协议规定的xml数据，其他方法都不需要。构建xml数据的方法也是在上面给出的链接里复制的，反正就是upnp协议规定好的，需要这中格式的数据，如果你想接收端能比较完整的显示投屏的数据信息，传递的MediaInfo可以详细些，我这里都值传递了多媒体地址信息。


# 三、结语

唉，终于贴完代码了，贴的时候感觉好无奈，自己也很反感这中方式，但是这只是对cling的一个简单实用实用示例，技术细节都是别人处理好了的，我只是做了点简单的分层，希望大家看了demo能直接使用cling实现投屏功能，也没什么技术分析，所以就只是贴个代码了。

至于使用的方法，我就更懒得贴了，没有任何意义，大家直接看源码的demo就可以了，我只给大家提几个需要注意的地方：


1.app module的build.gradle文件必须要加上一句
	
		//去重复的引用
    packagingOptions {
        exclude 'META-INF/beans.xml'
    }

这是由于引入jetty引起的文件重复。

2.build.gradle文件里类似如下代码

	  minSdkVersion rootProject.ext.minSdkVersion
     targetSdkVersion rootProject.ext.targetSdkVersion
     versionCode rootProject.ext.versionCode
     versionName rootProject.ext.versionName


里面的ext.minSdkVersion等等，请参见根目录的build.gradle。

3.所有工程的依赖库都基于androidx，所以，如果有需要的童鞋在集成到自己的工程里的时候要慎重，因为androidx库和support库不兼容。

最后，祝大家工作愉快。

















