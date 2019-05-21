---
layout:     post
title:      Unity与Android通信的中间件(二)
subtitle:   Unity3d与android的那些事儿
date:       2019-05-21
author:     YANKEBIN
catalog: true
tags:
     - Android
     - Unity3d
     - Android与Unity3d通信
     - Unity3d与Android通信
---

# 一 、前言
最近都好忙好忙，感觉很累，曾好几次想继续把关于Unity和Android相互通信的这部分技术分享的博客写完，但是实在是无法提起写博客的精神，所以就一拖再拖，从一月份拖到了五月份，好在当时的思路想法都还在，今天就让这部分博客画个句号吧。

在前一篇文章：[Unity与Android通信的中间件(一)](https://blog.csdn.net/yankebin/article/details/86715053) 里我说了我写这个插件主要是为了解决两个问题：一个是降低android端代码和unity端代码的耦合度，还有一个是降低android端的接入复杂度。降低android端代码和unity端代码的耦合度，在那篇文章里已经解释清楚了，并且提供了源码，大家直接去看源码就可以了，今天就接着来讲降低android端的接入复杂度的问题。


# 二 、实现思路
其实降低android端的接入复杂度的问题根本不算问题，这个肯定是仁者见仁智者见智，我在这里只是提供一种我所想到的实现思路，说白了就是封装一些基础代码，让开发的人可以添加少量的代码即可完成Unity和Android相互通信的功能。由于我本身技术的问题，封装的可能就是写laji，希望大家轻拍，谢谢。

## 2.1 解决android端的接入复杂度高的问题

Unity最终提供给Android端的其实就是个View，所谓降低android端的接入复杂度，就是让Android端可以装载View的一些模块可以简单、快速的实现显示Unity端视图的功能，比如ViewGroup、Dialog、Activity、Fragment，等等。基于以上考虑，所以我只是封装了Activity、Fragment的基类，至于ViewGroup、Dialog,完全可以在Fragment和Activity里显示，我就不做过多示例了。

### 2.1.1 Fragment和Activity都需要实现的接口——IBaseView

	/**
	 * Description：Basic interface of all {@link Activity}
	 * or
	 * {@link Fragment}
	 * or
	 * {@link android.app.Fragment}
	 * <p>
	 * Creator：yankebin
	 * CreatedAt：2018/12/18
	 */
	public interface IBaseView {
	
	    /**
	     * Return the layout resource
	     *
	     * @return Layout Resource
	     */
	    @LayoutRes
	    int contentViewLayoutId();
	
	    /**
	     * Call after {@link Activity#onCreate(Bundle)}
	     * or
	     * {@link Fragment#onCreateView(LayoutInflater, ViewGroup, Bundle)}
	     * or
	     * {@link android.app.Fragment#onCreateView(LayoutInflater, ViewGroup, Bundle)}
	     *
	     * @param params
	     * @param contentView
	     */
	    void onViewCreated(@NonNull Bundle params, @NonNull View contentView);
	
	    /**
	     * Call after
	     * {@link Activity#onDestroy()} (Bundle)}
	     * or
	     * {@link Fragment#onDestroyView()}
	     * or
	     * {@link android.app.Fragment#onDestroyView()}
	     */
	    void onVieDestroyed();
	}


这个类很简单的，只是统一下Activity、Fragment、v4包的Fragment的一些抽象方法，方便封装BaseActivity、BaseFragment、BaseFragmentV4。比如contentViewLayoutId方法，就是让开发者可以直接返回布局的id，而不用自己去写加载布局的代码。


### 2.1.2 显示Unity端视图的模块的基类——BaseActivity、BaseFragment、BaseFragmentV4

	/**
	 * Description：Activity基类
	 * Creator：yankebin
	 * CreatedAt：2018/12/18
	 */
	public abstract class BaseActivity extends AppCompatActivity implements IBaseView {
	    protected View mainContentView;
	    protected Unbinder unbinder;
	
	    @Override
	    protected void onCreate(@Nullable Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        mainContentView =  getLayoutInflater().inflate(contentViewLayoutId(),
	                (ViewGroup) getWindow().getDecorView(), false);
	        setContentView(mainContentView);
	        unbinder = ButterKnife.bind(this, mainContentView);
	        Bundle bundle = getIntent().getExtras();
	        if (null == bundle) {
	            bundle = new Bundle();
	        }
	        if (null != savedInstanceState) {
	            bundle.putAll(savedInstanceState);
	        }
	        onViewCreated(bundle, mainContentView);
	    }
	
	    @Override
	    protected void onDestroy() {
	        onVieDestroyed();
	        if (null != unbinder) {
	            unbinder.unbind();
	        }
	        mainContentView = null;
	        super.onDestroy();
	    }
	}
	
	/**
	 * Description：app包下Fragment作为基类封装
	 * Creator：yankebin
	 * CreatedAt：2018/12/18
	 */
	@Deprecated
	public abstract class BaseFragment extends Fragment implements IBaseView {
	    protected Unbinder unbinder;
	    protected View mainContentView;
	
	    @Nullable
	    @Override
	    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
	                             @Nullable Bundle savedInstanceState) {
	        if (null == mainContentView) {
	            mainContentView = inflater.inflate(contentViewLayoutId(), container, false);
	            unbinder = ButterKnife.bind(this, mainContentView);
	            Bundle bundle = getArguments();
	            if (null == bundle) {
	                bundle = new Bundle();
	            }
	            if (null != savedInstanceState) {
	                bundle.putAll(savedInstanceState);
	            }
	            onViewCreated(bundle, mainContentView);
	        } else {
	            unbinder = ButterKnife.bind(this, mainContentView);
	        }
	        return mainContentView;
	    }
	
	    @Override
	    public void onDetach() {
	        mainContentView = null;
	        super.onDetach();
	    }
	
	    @Override
	    public void onDestroyView() {
	        onVieDestroyed();
	        if (null != unbinder) {
	            unbinder.unbind();
	        }
	        if (mainContentView != null && mainContentView.getParent() != null) {
	            ((ViewGroup) mainContentView.getParent()).removeView(mainContentView);
	        }
	        super.onDestroyView();
	    }
	}

	/**
	 * Description：V4包下Fragment作为基类封装
	 * Creator：yankebin
	 * CreatedAt：2018/12/18
	 */
	public abstract class BaseFragmentV4 extends Fragment implements IBaseView {
	    protected Unbinder unbinder;
	    protected View mainContentView;
	
	    @Nullable
	    @Override
	    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
	                             @Nullable Bundle savedInstanceState) {
	        if (null == mainContentView) {
	            mainContentView = inflater.inflate(contentViewLayoutId(), container, false);
	            unbinder = ButterKnife.bind(this, mainContentView);
	            Bundle bundle = getArguments();
	            if (null == bundle) {
	                bundle = new Bundle();
	            }
	            if (null != savedInstanceState) {
	                bundle.putAll(savedInstanceState);
	            }
	            onViewCreated(bundle, mainContentView);
	        } else {
	            unbinder = ButterKnife.bind(this, mainContentView);
	        }
	        return mainContentView;
	    }
	
	    @Override
	    public void onDetach() {
	        mainContentView = null;
	        super.onDetach();
	    }
	
	    @Override
	    public void onDestroyView() {
	        onVieDestroyed();
	        if (null != unbinder) {
	            unbinder.unbind();
	        }
	        if (mainContentView != null && mainContentView.getParent() != null) {
	            ((ViewGroup) mainContentView.getParent()).removeView(mainContentView);
	        }
	        super.onDestroyView();
	    }
	}


这三个类主要是简化了开发者加载布局相关的代码，以及简化生命周期回调的方法数量，让开发者只关心该关注的生命周期回调。


### 2.1.3 显示Unity端视图的模块的基类进一步封装——UnityPlayerActivity、UnityPlayerFragment、UnityPlayerFragmentV4

UnityPlayerActivity：
		
	/**
	 * Desription：Base Unity3D Player Activity.
	 * <BR/>
	 * If you want to implement Fragment to load the Unity scene, then the Fragment needs to refer to {@link UnityPlayerFragment} and {@link UnityPlayerFragmentV4}
	 *    To implement, then overload the {@link #generateIOnUnity3DCallDelegate(UnityPlayer, Bundle)} method to return the conforming Fragment.
	 * <BR/>
	 * Creator：yankebin
	 * <BR/>
	 * CreatedAt：2018/12/1
	 */
	public abstract class UnityPlayerActivity extends BaseActivity implements IGetUnity3DCall, IOnUnity3DCall, IUnityPlayerContainer {
	    protected UnityPlayer mUnityPlayer; // don't change the name of this variable; referenced from native code
	    protected IOnUnity3DCall mOnUnity3DCallDelegate;
	
	    @Override
	    @CallSuper
	    public void onViewCreated(@NonNull Bundle params, @NonNull View contentView) {
	        initUnityPlayer(params);
	    }
	
	    /**
	     * Initialize Unity3D related
	     *
	     * @param bundle {@link Bundle}
	     */
	    protected void initUnityPlayer(@NonNull Bundle bundle) {
	        mUnityPlayer = new UnityPlayer(this);
	        mOnUnity3DCallDelegate = generateIOnUnity3DCallDelegate(mUnityPlayer, bundle);
	        if (null != mOnUnity3DCallDelegate) {
	            if (mOnUnity3DCallDelegate instanceof Fragment) {
	                getSupportFragmentManager().beginTransaction().replace(unityPlayerContainerId(),
	                        (Fragment) mOnUnity3DCallDelegate, ((Fragment) mOnUnity3DCallDelegate)
	                                .getClass().getName()).commit();
	            } else if (mOnUnity3DCallDelegate instanceof android.app.Fragment) {
	                getFragmentManager().beginTransaction().replace(unityPlayerContainerId(),
	                        (android.app.Fragment) mOnUnity3DCallDelegate, ((android.app.Fragment) mOnUnity3DCallDelegate)
	                                .getClass().getName()).commit();
	            } else if (mOnUnity3DCallDelegate instanceof View) {
	                final ViewGroup unityContainer = findViewById(unityPlayerContainerId());
	                unityContainer.addView((View) mOnUnity3DCallDelegate);
	            } else {
	                throw new IllegalArgumentException("Not support type : " + mOnUnity3DCallDelegate.toString());
	            }
	        } else {
	            mOnUnity3DCallDelegate = this;
	            final ViewGroup unityContainer = findViewById(unityPlayerContainerId());
	            unityContainer.addView(mUnityPlayer);
	            mUnityPlayer.requestFocus();
	        }
	    }
	
	    @Override
	    protected void onNewIntent(Intent intent) {
	        // To support deep linking, we need to make sure that the client can get access to
	        // the last sent intent. The clients access this through a JNI api that allows them
	        // to get the intent set on launch. To update that after launch we have to manually
	        // replace the intent with the one caught here.
	        setIntent(intent);
	    }
	
	    // Quit Unity
	    @Override
	    protected void onDestroy() {
	        if (null != mUnityPlayer) {
	            mUnityPlayer.quit();
	        }
	        super.onDestroy();
	    }
	
	    // Pause Unity
	    @Override
	    protected void onPause() {
	        super.onPause();
	        if (null != mUnityPlayer) {
	            mUnityPlayer.pause();
	        }
	    }
	
	    // Resume Unity
	    @Override
	    protected void onResume() {
	        super.onResume();
	        if (null != mUnityPlayer) {
	            mUnityPlayer.resume();
	        }
	    }
	
	    @Override
	    protected void onStart() {
	        super.onStart();
	        if (null != mUnityPlayer) {
	            mUnityPlayer.start();
	        }
	    }
	
	    @Override
	    protected void onStop() {
	        super.onStop();
	        if (null != mUnityPlayer) {
	            mUnityPlayer.stop();
	        }
	    }
	
	    // Low Memory Unity
	    @Override
	    public void onLowMemory() {
	        super.onLowMemory();
	        if (null != mUnityPlayer) {
	            mUnityPlayer.lowMemory();
	        }
	    }
	
	    // Trim Memory Unity
	    @Override
	    public void onTrimMemory(int level) {
	        super.onTrimMemory(level);
	        if (level == TRIM_MEMORY_RUNNING_CRITICAL) {
	            if (null != mUnityPlayer) {
	                mUnityPlayer.lowMemory();
	            }
	        }
	    }
	
	    // This ensures the layout will be correct.
	    @Override
	    public void onConfigurationChanged(Configuration newConfig) {
	        super.onConfigurationChanged(newConfig);
	        if (null != mUnityPlayer) {
	            mUnityPlayer.configurationChanged(newConfig);
	        }
	    }
	
	    // Notify Unity of the focus change.
	    @Override
	    public void onWindowFocusChanged(boolean hasFocus) {
	        super.onWindowFocusChanged(hasFocus);
	        if (null != mUnityPlayer) {
	            mUnityPlayer.windowFocusChanged(hasFocus);
	        }
	    }
	
	    // For some reason the multiple keyevent type is not supported by the ndk.
	    // Force event injection by overriding dispatchKeyEvent().
	    @Override
	    public boolean dispatchKeyEvent(KeyEvent event) {
	        if (event.getAction() == KeyEvent.ACTION_MULTIPLE) {
	            if (null != mUnityPlayer) {
	                return mUnityPlayer.injectEvent(event);
	            }
	        }
	        return super.dispatchKeyEvent(event);
	    }
	
	    // Pass any events not handled by (unfocused) views straight to UnityPlayer
	    @Override
	    public boolean onKeyUp(int keyCode, KeyEvent event) {
	        if (keyCode == KeyEvent.KEYCODE_BACK) {
	            return super.onKeyUp(keyCode, event);
	        }
	        if (null != mUnityPlayer) {
	            return mUnityPlayer.injectEvent(event);
	        }
	        return super.onKeyUp(keyCode, event);
	    }
	
	    @Override
	    public boolean onKeyDown(int keyCode, KeyEvent event) {
	        if (keyCode == KeyEvent.KEYCODE_BACK) {
	            return super.onKeyDown(keyCode, event);
	        }
	        if (null != mUnityPlayer) {
	            return mUnityPlayer.injectEvent(event);
	        }
	        return super.onKeyDown(keyCode, event);
	    }
	
	    @Override
	    public boolean onTouchEvent(MotionEvent event) {
	        if (null != mUnityPlayer) {
	            return mUnityPlayer.injectEvent(event);
	        }
	        return super.onTouchEvent(event);
	    }
	
	    /*API12*/
	    public boolean onGenericMotionEvent(MotionEvent event) {
	        if (null != mUnityPlayer) {
	            return mUnityPlayer.injectEvent(event);
	        }
	        return super.onGenericMotionEvent(event);
	    }
	
	    @Nullable
	    @Override
	    public Context gatContext() {
	        return this;
	    }
	
	    @NonNull
	    @Override
	    public IOnUnity3DCall getOnUnity3DCall() {
	        //Perhaps this method is called after Unity is created after the activity is created,
	        // so there is no problem for the time being.
	        return mOnUnity3DCallDelegate;
	    }
	
	    @Nullable
	    protected IOnUnity3DCall generateIOnUnity3DCallDelegate(@NonNull UnityPlayer unityPlayer,
	                                                            @Nullable Bundle bundle) {
	        return null;
	    }
	}


别看这个类有200多行，其实绝大部分代码都是重载，是为了满足Unity端的需求，真正要关注的方法就那么三四个，只有generateIOnUnity3DCallDelegate(@NonNull UnityPlayer unityPlayer,@Nullable Bundle bundle)这个方法需要继承的实现者去关注，这个方法的作用就是生成真正去加载和显示Unity端使视图的模块，不关你是View也好，Fragment也好，只要你实现了IOnUnity3DCall接口和IUnityPlayerContainer接口，你就可以加载和显示Unity端的视图。

默认情况下，继承至UnityPlayerActivity的类就是加载和显示Unity端视图的模块，除非你重写generateIOnUnity3DCallDelegate(@NonNull UnityPlayer unityPlayer,@Nullable Bundle bundle)方法，返回合适的代理，这个可以从initUnityPlayer(@NonNull Bundle bundle) 方法里面直观的看出来。

如果你还想再进一步，实现了IOnUnity3DCall接口和IUnityPlayerContainer接口的代理模块还想让其他模块来显示Unity短的View，那么实现了IOnUnity3DCall接口和IUnityPlayerContainer接口的代理模块就可以在实现IGetUnity3DCall接口，重写generateIOnUnity3DCallDelegate(@NonNull UnityPlayer unityPlayer,@Nullable Bundle bundle)方法，返回合适的代理。当然，可能大多数情况下我们也不需要这么做吧。

基于以上的原理，要让Fragment作为显示Unity短的View的模块的方法就呼之欲出了：

v4包下的Fragment

	/**
	 * Description：The base Unity3D fragment, the Activity holding this Fragment needs to implement the {@link IGetUnity3DCall} interface and implement {@link IGetUnity3DCall#getOnUnity3DCall()}
	 * 方法返回此Fragment的实例
	 * Creator：yankebin
	 * CreatedAt：2018/12/1
	 */
	public abstract class UnityPlayerFragmentV4 extends BaseFragmentV4 implements IOnUnity3DCall, IUnityPlayerContainer {
	    protected UnityPlayer mUnityPlayer; // don't change the name of this variable; referenced from native code
	
	    public void setUnityPlayer(@NonNull UnityPlayer mUnityPlayer) {
	        this.mUnityPlayer = mUnityPlayer;
	    }
	
	    @Override
	    @CallSuper
	    public void onViewCreated(@NonNull Bundle params, @NonNull View contentView) {
	        if (null != mUnityPlayer) {
	            final ViewGroup unityContainer = contentView.findViewById(unityPlayerContainerId());
	            unityContainer.addView(mUnityPlayer);
	            mUnityPlayer.requestFocus();
	        }
	    }
	
	    @Nullable
	    @Override
	    public Context gatContext() {
	        return getActivity();
	    }
	}

app包下的Fragment

	/**
	 * Description：The base Unity3D fragment, the Activity holding this Fragment needs to implement the {@link IGetUnity3DCall} interface and implement {@link IGetUnity3DCall#getOnUnity3DCall()}
	 *   Method returns an instance of this Fragment
	 * Creator：yankebin
	 * CreatedAt：2018/12/1
	 */
	public abstract class UnityPlayerFragment extends BaseFragment implements IOnUnity3DCall, IUnityPlayerContainer {
	    protected UnityPlayer mUnityPlayer; // don't change the name of this variable; referenced from native code
	
	    public void setUnityPlayer(@NonNull UnityPlayer mUnityPlayer) {
	        this.mUnityPlayer = mUnityPlayer;
	    }
	
	    @Override
	    @CallSuper
	    public void onViewCreated(@NonNull Bundle params, @NonNull View contentView) {
	        if (null != mUnityPlayer) {
	            final ViewGroup unityContainer = contentView.findViewById(unityPlayerContainerId());
	            unityContainer.addView(mUnityPlayer);
	            mUnityPlayer.requestFocus();
	        }
	    }
	
	    @Nullable
	    @Override
	    public Context gatContext() {
	        return getActivity();
	    }
	}
	

两者的实现完全一致，只是为了让开发者少封装一种Fragment。我想，当你们看到这里的时候，心中对如何让一个ViewGroup或者Dialog显示Unity端的View的方法已经很清晰了吧。


## 2.2 唠叨的结语

在我的Demo里，让一个Activity加载和控制并且响应Unity做的一个小游戏只需要不到100行代码，看起来是不是已经很简单啦？就像我在开头所说的一样，降低android端的接入复杂度其实是一个仁者见仁智者见智的问题，我所能做到的简化大概就是这个样子，但是我相信这绝对是一个很low的封装，大神们肯定可以用各种各样的设计模式来设计出牛b的架构，所以我只是在这里给出我的一种见解，希望大家轻拍。

还有就是，其实做Unity的难点一直都不在于Android这端，也不在于两端的通信，我做这个项目只适合入门级开发者闲时无事来看一看，扩宽点知识面，了解点新东西，顺便练练自己的文笔，哈哈哈哈哈。

提前透露下吧，后面可能会写几篇关于基于Google chromium内核的Android Chrome浏览器相关的内容，主要涉及源码改造和嵌入ijkplayer实现视频播放的功能（类似腾讯X5内核的多媒体播放功能），希望大家多多支持。

