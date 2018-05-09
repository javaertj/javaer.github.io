---
layout:     post
title:      MVP模式探索——Presenter和View解耦的尝试之APT技术
subtitle:   MVP模式探索5
date:       2018-05-09
author:     YANKEBIN
catalog: true
tags:
     - Android
     - APT
     - MVP
     - MMVP
     - 架构模式
---

#前言


在前一篇文章[《MVP模式探索-Presenter和View解耦的尝试》](https://ykbjson.github.io/2018/04/17/MVP%E6%A8%A1%E5%BC%8F%E6%8E%A2%E7%B4%A2-Presenter%E5%92%8CView%E8%A7%A3%E8%80%A6%E7%9A%84%E5%B0%9D%E8%AF%95/)里，我在文章末尾说到了该解耦方式有几个已知的问题，其中一个就是用反射的方式去执行方法会有性能上的损耗，但是可以用APT技术替代反射的方式。后来我去查了一下反射性能损耗的问题，发现反射出现性能损耗是有一定条件的，那就是需要在反复多次执行同一个反射调用时，当反复执行的次数达到一个数量级后，才会和普通调用产生性能差异。然而像MMVP模式里的那些反射执行的方法，并不会在短时间内反复多次调用，所以其实并不存在什么性能上的问题。

关于反射性能问题的文章参考如下
>[Java反射到底慢在哪？](https://www.jianshu.com/p/4e2b49fa8ba1)

但是今天在这里我还是要用APT技术去替代反射的方案，这仅仅是多尝试一种方式和思维去解决同一个问题，也顺便学习一下APT技术和JavaPoet。


当然，在具体实施之前，如果大家对APT技术或JavaPoet还不太了解，就先去充一下电在来看，会顺畅很多。
>[Android APT（编译时代码生成）最佳实践](https://joyrun.github.io/2016/07/19/AptHelloWorld/)

>[Android编译时注解框架5-语法讲解](https://lizhaoxuan.github.io/2016/07/17/apt-Grammar-explanation/)

>[JavaPoet 看这一篇就够了](https://juejin.im/entry/58fefebf8d6d810058a610de)

>[JavaPoet源码](https://github.com/square/javapoet)

#一、技术实施细节


因为替代方案是在原来的代码基础上改的，所以后面的叙述都会以原来的代码为相对参考，所以，如果有什么疑惑的地方，请记得去看我前一篇文章

>[《MVP模式探索-Presenter和View解耦的尝试》](https://ykbjson.github.io/2018/04/17/MVP%E6%A8%A1%E5%BC%8F%E6%8E%A2%E7%B4%A2-Presenter%E5%92%8CView%E8%A7%A3%E8%80%A6%E7%9A%84%E5%B0%9D%E8%AF%95/)


### 1.1 基本说明 

1. 在尝试APT技术的时候，一开始就遇到了一个问题，我们要编写的annotationProcessor是一个Java Library，而我最开始的MMVP moudle是一个Android Library，annotationProcessor需要MMVP moudle里的注解，但是我发现似乎没有办法把一个Android Library作为Java Library的依赖，但是反过来却是可以的，所以我就只好把MMVP moudle里的一些annotationProcessor需要的注解抽离出去，单独建了一个mmvpannotation Java Library，让annotationProcessor moudle和MMVP moudle都依赖mmvpannotation。关于把Android Library作为Java Library的依赖的资料参考

>[为Java Library 添加 Android 注解支持](https://www.jianshu.com/p/fa5163c08452)


2. 原来在使用反射方式的的时候，有些类（比如Presenter类）是不需要使用注解来标识的，但是使用APT技术后，只能通过注解来找到目标类生成对应的代理类解析MMVPAction后，再调用目标类的方法或修改其属性，所这里就新增了了一个注解，MMVPActionProcessor，这个注解在mmvpannotation moudle下

		/**
		 * 包名：com.ykbjson.lib.mmvp.annotation
		 * 描述：MMVPAction处理类需要的注解
		 * 创建者：yankebin
		 * 日期：2018/5/7
		 */
		@Target(ElementType.TYPE)
		@Retention(RetentionPolicy.CLASS)
		public @interface MMVPActionProcessor {
		}


3. 在MMVP项目里，**APT技术在编译时生成的类,需要能够直接调用或通过目标类的实例调用目标类（即添加了MMVPActionProcessor注解的类）的某些方法或者更改目标类的某些属性（这就是和反射的差别之处，直接调用）**，因为IMMVPActionHandler接口是专门负责处理MMVPActin的接口，所以我们就让编译时生成的类也实现IMMVPActionHandler接口，方便处理MMVPAction后调用目标类的方法。

4. 修改之后的项目结构有所变化，新增了mmvpannotation、mmvpcompiler两个moudle

![](https://ykbjson.github.io/blogimage/mvppicture5/project_structure.png)


### 1.2 APT+JavaPoet输出目标类

其实这里真没什么好说的，只要大家去看了我前面说的那几篇文章（APT+JavaPoet），加上我代码里的注释，一定很轻易地就可以看懂了。
我们来看一下这个生成目标类的MMVPAnnotationProcessor

	/**
	 * 包名：com.ykbjson.lib.mmvp.compiler
	 * 描述：mmvp{@link MMVPActionProcessor}注解处理器
	 * 创建者：yankebin
	 * 日期：2018/5/4
	 */
	@AutoService(Processor.class)
	public class MMVPAnnotationProcessor extends AbstractProcessor {
	    private static final String CLASS_NAME_SUFFIX = "_MMVPActionProcessor";
	    private static final String METHOD_NAME_SUFFIX = "_Process";
	    private static final String OVERRIDE_METHOD_NAME_HANDLE_ACTION = "handleAction";
	    private static final String OVERRIDE_METHOD_NAME_GET = "get";
	
	    private Elements elementUtils;
	
	    @Override
	    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
	        parseExecutor(roundEnv);
	        return true;
	    }
	
	    private void parseExecutor(RoundEnvironment roundEnv) {
	        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(MMVPActionProcessor.class);
	        for (Element element : elements) {
	            // 判断是否Class
	            TypeElement typeElement = (TypeElement) element;
	            //要生成的类
	            TypeSpec.Builder typeSpecBuilder = TypeSpec.classBuilder(element.getSimpleName() +
	                    CLASS_NAME_SUFFIX)//类名
	                    .addSuperinterface(ClassName.get("com.ykbjson.lib.mmvp", "IMMVPActionHandler"))//实现的接口
	                    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)//类修饰符
	                    .addField(FieldSpec.builder(TypeName.get(typeElement.asType()), "target", Modifier.PRIVATE).build());//成员变量
	            //生成的类的构造方法
	            MethodSpec.Builder constructorBuilder = MethodSpec.constructorBuilder()
	                    .addModifiers(Modifier.PUBLIC)//方法的修饰符
	                    .addParameter(TypeName.get(typeElement.asType()), "target")//方法的参数
	                    .addStatement("this.$N = $N", "target", "target");//方法的内容
	
	            typeSpecBuilder.addMethod(constructorBuilder.build());//添加到类里
	            //target类目标方法获取和参数解析后调用。这里就是遍历target目标类里的方法，看那哪些方法添加了ActionProcess注解，
	            // 然后，根据收到的Action，根据方法的注解参数，解析action里的参数，然后调用目标类的方法。
	            final List<? extends Element> members = elementUtils.getAllMembers(typeElement);
	            final Map<String, String> methodMap = new LinkedHashMap<>();
	            for (Element item : members) {
	                ActionProcess methodAnnotation = item.getAnnotation(ActionProcess.class);
	                if (methodAnnotation == null) {
	                    continue;
	                }
	
	                final String generatedMethodName = item.getSimpleName().toString() + METHOD_NAME_SUFFIX;
	                //保存方法和注解的关系
	                methodMap.put(methodAnnotation.value(), generatedMethodName);
	                MethodSpec.Builder actionProcessMethodSpecBuilder = MethodSpec.methodBuilder(
	                        generatedMethodName)
	                        .returns(TypeName.BOOLEAN)
	                        .addModifiers(Modifier.PUBLIC);
	
	                //方法必要的唯一参数-MMVPAction
	                actionProcessMethodSpecBuilder.addParameter(ParameterSpec.builder(
	                        ClassName.get("com.ykbjson.lib.mmvp", "MMVPAction"), "action")
	                        .build());
	                //如果当前传入的MMVPAction要执行的方法和当前方法不一致，中断执行
	                CodeBlock codeBlock = CodeBlock.builder().beginControlFlow("if(!\"" + methodAnnotation.value() +
	                        "\".equals(action.getAction().getAction()))")
	                        .addStatement("return false")
	                        .endControlFlow()
	                        .build();
	                actionProcessMethodSpecBuilder.addCode(
	                        codeBlock
	                );
	
	                //获取和处理方法参数列表
	                ExecutableElement method = (ExecutableElement) item;//方法
	                List<? extends VariableElement> parameters = method.getParameters();//方法的参数
	                StringBuilder parametersBuffer = new StringBuilder();
	                //参数集合,需要参数才去解析参数，这下面生成的代码和MMVPArtist里的execute方法里的代码非常类似
	                if (!parameters.isEmpty()) {
	                    actionProcessMethodSpecBuilder.addStatement("$T  paramList= new ArrayList<>()", ArrayList.class);
	                    if (methodAnnotation.needActionParam()) {
	                        if (methodAnnotation.needTransformAction()) {
	                            actionProcessMethodSpecBuilder.addStatement(
	                                    "action = action.transform()"
	                            );
	                        }
	
	                        actionProcessMethodSpecBuilder.addStatement(
	                                "paramList.add(action)"
	                        );
	                    }
	                    if (methodAnnotation.needActionParams()) {
	                        actionProcessMethodSpecBuilder.addStatement(
	                                //这里其实可以使用JavaPoet的beginControlFlow来优雅地实现for循环
	                                "if(null != action.getParams() && !action.getParams().isEmpty()) {\n" +
	                                        "for (String key : action.getParams().keySet()) {\n" +
	                                        "     paramList.add(action.getParam(key));\n" +
	                                        "}" +
	                                        "}"
	                        );
	                    }
	
	                    for (int i = 0; i < parameters.size(); i++) {
	                        parametersBuffer.append(
	                                "(" + parameters.get(i).asType() + ")" + "paramList.get(" + i + ")");
	                        if (i != parameters.size() - 1) {
	                            parametersBuffer.append(",");
	                        }
	                    }
	                }
	                //这里生成的代码类似 target.findViewById(-122443433)
	                actionProcessMethodSpecBuilder.addStatement("target." +
	                        method.getSimpleName().toString() +
	                        "(" + parametersBuffer.toString() + ")");
	
	                actionProcessMethodSpecBuilder.addStatement("return true");
	                typeSpecBuilder.addMethod(actionProcessMethodSpecBuilder.build());
	            }
	            //重载IMMVPActionHandler方法handleAction
	            MethodSpec.Builder overrideMethodSpecBuilder = MethodSpec.methodBuilder(OVERRIDE_METHOD_NAME_HANDLE_ACTION)
	                    .addAnnotation(Override.class)
	                    .addModifiers(Modifier.PUBLIC)
	                    .returns(TypeName.BOOLEAN);
	
	            overrideMethodSpecBuilder.addParameter(ParameterSpec.builder(
	                    ClassName.get("com.ykbjson.lib.mmvp", "MMVPAction"), "action")
	                    .addAnnotation(ClassName.get("android.support.annotation", "NonNull"))
	                    .build());
	            //由于无法预知即将调用的方法，只能把重写的方法全部执行一遍，重写方法里有判断可以避免错误执行,并且只要有某个方法返回了true，后续方法将不再执行.
	            // 或许这样也不比反射执行的风险小吧.
	            int index = 0;
	            StringBuilder resultBuilder = new StringBuilder();
	            for (String key : methodMap.keySet()) {
	                resultBuilder.append(methodMap.get(key) + "(action)" + (index != methodMap.keySet().size() - 1 ? "||" : ""));
	                index++;
	            }
	            overrideMethodSpecBuilder.addStatement("return " + (resultBuilder.length() == 0 ? "false" : resultBuilder.toString()));
	            typeSpecBuilder.addMethod(overrideMethodSpecBuilder.build());
	
	            //重载IMMVPActionHandler方法get
	            overrideMethodSpecBuilder = MethodSpec.methodBuilder(OVERRIDE_METHOD_NAME_GET)
	                    .addAnnotation(Override.class)
	                    .addModifiers(Modifier.PUBLIC)
	                    .returns(ClassName.get("com.ykbjson.lib.mmvp", "IMMVPActionHandler"));
	            overrideMethodSpecBuilder.addStatement("return this");
	            typeSpecBuilder.addMethod(overrideMethodSpecBuilder.build());
	
	            //生成java文件
	            JavaFile javaFile = JavaFile.builder(getPackageName(typeElement), typeSpecBuilder.build())
	                    .addFileComment(" Generated code from MMVP. Do not modify! ")
	                    .build();
	            try {
	                javaFile.writeTo(processingEnv.getFiler());
	            } catch (IOException e) {
	                e.printStackTrace();
	            }
	        }
	    }
	
	    @Override
	    public Set<String> getSupportedAnnotationTypes() {
	        Set<String> supportedAnnotationTypes = new LinkedHashSet<>();
	        supportedAnnotationTypes.add(MMVPActionProcessor.class.getCanonicalName());
	        return supportedAnnotationTypes;
	    }
	
	    private String getPackageName(TypeElement type) {
	        return elementUtils.getPackageOf(type).getQualifiedName().toString();
	    }
	
	    @Override
	    public synchronized void init(ProcessingEnvironment processingEnv) {
	        super.init(processingEnv);
	        elementUtils = processingEnv.getElementUtils();
	    }
	
	    @Override
	    public SourceVersion getSupportedSourceVersion() {
	        return SourceVersion.RELEASE_7;
	    }
	}

这个类里面如果一定要细讲的话，那讲得很多的就是java注解语法基础和JavaPoet使用方法了，但是这并不是本篇文章的讨论范围，也不是大家的难点所在，所以就不赘述了，总之，这里就是在编译时为所有添加了MMVPActionProcessor注解的类都生成一个对应的类，生成的这个类可以直接操作添加了MMVPActionProcessor注解的类里的方法或属性，从而用直接调用的方式替代反射调用的方式。


# 二、效果测试和比对

我们以mmvpsample里的LoginPresenter为例，看一看这个类生成的类到底是如何调用LoginPresenter里的方法的。

LoginPresenter

	/**
	 * 包名：com.ykbjson.app.hvp.login
	 * 描述：登录逻辑处理
	 * 创建者：yankebin
	 * 日期：2018/4/13
	 */
	@MMVPActionProcessor
	public class LoginPresenter implements LoginContract.ILoginPresenter {
	    public static final String ACTION_DO_LOGIN = "LoginPresenter.action.ACTION_DO_LOGIN";
	    public static final String ACTION_NOTIFY_LOGIN_FAILED = "LoginPresenter.action.ACTION_NOTIFY_LOGIN_FAILED";
	    public static final String ACTION_NOTIFY_LOGIN_SUCCESS = "LoginPresenter.action.ACTION_NOTIFY_LOGIN_SUCCESS";
	
	    private LoginRepository repository = new LoginRepository();
	
	    @ActionProcess(value = ACTION_DO_LOGIN, needActionParam = true, needTransformAction = true)
	    @Override
	    public void doLogin(final MMVPAction action) {
	        //参数校验
	        String mobile = action.getParam("mobile");
	        String password = action.getParam("password");
	        if (TextUtils.isEmpty(mobile)) {
	            action.getAction().setAction(ACTION_NOTIFY_LOGIN_FAILED);
	            action.clearParam().putParam("error", "登录账号无效").send();
	            return;
	        }
	        if (TextUtils.isEmpty(password)) {
	            action.getAction().setAction(ACTION_NOTIFY_LOGIN_FAILED);
	            action.clearParam().putParam("error", "登录密码无效").send();
	            return;
	        }
	
	        repository.doLogin(mobile, password, new IMMVPOnDataCallback<LoginResult>() {
	            @Override
	            public void onSuccess(LoginResult data) {
	                action.getAction().setAction(ACTION_NOTIFY_LOGIN_SUCCESS);
	                action.clearParam().putParam("loginResult", data).send();
	            }
	
	            @Override
	            public void onError(String msg) {
	                action.getAction().setAction(ACTION_NOTIFY_LOGIN_FAILED);
	                action.clearParam().putParam("error", msg).send();
	            }
	        });
	    }
	
	    @Override
	    public boolean handleAction(@NonNull MMVPAction action) {
	        switch (action.getAction().getAction()) {
	            case ACTION_DO_LOGIN:
	                doLogin(action.transform());
	                break;
	        }
	
	        return true;
	    }
	
	    @NonNull
	    @Override
	    public IMMVPActionHandler get() {
	        return this;
	    }
	}	

只需要在LoginPresenter上加上MMVPActionProcessor注解，rebuild project，然后到mmvpsample/build/generated/source/debug(或release，根据你build的环境)/LoginPresenter所在包路径/目录下，会有一个生成的类：

LoginPresenter\_MMVPActionProcessor

	//  Generated code from MMVP. Do not modify! 
	package com.ykbjson.app.hvp.login;
	
	import android.support.annotation.NonNull;
	import com.ykbjson.lib.mmvp.IMMVPActionHandler;
	import com.ykbjson.lib.mmvp.MMVPAction;
	import java.lang.Override;
	import java.util.ArrayList;
	
	public final class LoginPresenter_MMVPActionProcessor implements IMMVPActionHandler {
	  private LoginPresenter target;
	
	  public LoginPresenter_MMVPActionProcessor(LoginPresenter target) {
	    this.target = target;
	  }
	
	  public boolean doLogin_Process(MMVPAction action) {
	    if(!"LoginPresenter.action.ACTION_DO_LOGIN".equals(action.getAction().getAction())) {
	      return false;
	    }
	    ArrayList  paramList= new ArrayList<>();
	    action = action.transform();
	    paramList.add(action);
	    target.doLogin((com.ykbjson.lib.mmvp.MMVPAction)paramList.get(0));
	    return true;
	  }
	
	  @Override
	  public boolean handleAction(@NonNull MMVPAction action) {
	    return doLogin_Process(action);
	  }
	
	  @Override
	  public IMMVPActionHandler get() {
	    return this;
	  }
	}

生成这个类的目的，文章开篇就说了，就是替代原来依靠反射方式执行某个方法的方案。原来MMVPArtist的处理逻辑是，在接收到一个MMVPAction后，经过层层查找和解析，如果有符合条件的目标对象需要解析这个MMVPAction，就会执行execute方法

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

通过代码大家可以看到，这里面最终目标类的方法的调用是使用的反射方式，现在我们用APT技术就是想不使用反射的方式也能调用目标类的方法。

当我们有了用APT技术生成的类时，execute方法可以修改如下，这里是重新写了一个方法，以示区分

	 	/**
	     * 执行action里目标类需要执行的方法
	     *
	     * @param action {@link MMVPAction}
	     * @param target 要执行action的类
	     * @return
	     */
	    private static boolean executeByApt(@NonNull MMVPAction action, @NonNull Object target) {
	        IMMVPActionHandler executor = createExecutor(target);
	        return executor.handleAction(action);
	    }
	    
是不是很好奇executor到底是谁？我们接着看一下createExecutor方法

	 	/**
	     * 创建{@link IMMVPActionHandler}
	     *
	     * @param target 当前关联的目标对象
	     * @return
	     */
	    private static IMMVPActionHandler createExecutor(@NonNull Object target) {
	        Class<?> targetClass = target.getClass();
	        if (enableLog) {
	            Log.d(TAG, "Looking up executor for " + targetClass.getName());
	        }
	        Constructor<? extends IMMVPActionHandler> constructor = findExecutorConstructorForClass(targetClass);
	
	        if (constructor == null) {
	            return IMMVPActionHandler.EMPTY;
	        }
	
	        //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
	        try {
	            return constructor.newInstance(target);
	        } catch (IllegalAccessException e) {
	            throw new RuntimeException("Unable to invoke " + constructor, e);
	        } catch (InstantiationException e) {
	            throw new RuntimeException("Unable to invoke " + constructor, e);
	        } catch (InvocationTargetException e) {
	            Throwable cause = e.getCause();
	            if (cause instanceof RuntimeException) {
	                throw (RuntimeException) cause;
	            }
	            if (cause instanceof Error) {
	                throw (Error) cause;
	            }
	            throw new RuntimeException("Unable to create executor instance.", cause);
	        }
	    } 


这里根据target，通过findExecutorConstructorForClass方法，找到了 LoginPresenter\_MMVPActionProcessor 的构造方法，生成了LoginPresenter\_MMVPActionProcessor的实例。

	 	/**
	     * 创建{@link IMMVPActionHandler}
	     *
	     * @param cls 当前关联的目标类
	     * @return
	     */
	    @Nullable
	    @CheckResult
	    @UiThread
	    private static Constructor<? extends IMMVPActionHandler> findExecutorConstructorForClass(Class<?> cls) {
	        Constructor<? extends IMMVPActionHandler> executorCtor = EXECUTORS.get(cls);
	        if (executorCtor != null) {
	            if (enableLog) {
	                Log.d(TAG, "Cached in executor map.");
	            }
	            return executorCtor;
	        }
	        String clsName = cls.getName();
	        if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
	            if (enableLog) {
	                Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
	            }
	            return null;
	        }
	        try {
	            Class<?> executorClass = cls.getClassLoader().loadClass(clsName + "_MMVPActionProcessor");
	            //noinspection unchecked
	            executorCtor = (Constructor<? extends IMMVPActionHandler>) executorClass.getConstructor(cls);
	            if (enableLog) {
	                Log.d(TAG, "HIT: Loaded executor class and constructor.");
	            }
	        } catch (ClassNotFoundException e) {
	            if (enableLog) {
	                Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
	            }
	            executorCtor = findExecutorConstructorForClass(cls.getSuperclass());
	        } catch (NoSuchMethodException e) {
	            throw new RuntimeException("Unable to find executor constructor for " + clsName, e);
	        }
	        EXECUTORS.put(cls, executorCtor);
	        return executorCtor;
	    }


如果大家研究过ButterKnife的源码，看到这里，是否有一种似曾相识的感觉?

现在有了LoginPresenter\_MMVPActionProcessor的实例,然后调用其handleAction方法，我们汇过去看看LoginPresenter\_MMVPActionProcessor的handleAction方法，调用了自身的doLogin\_Process方法，然后再看doLogin\_Process方法，是不是调用了LoginPresenter的doLogin方法？你肯定会很好奇，LoginPresenter\_MMVPActionProcessor怎么会这么智能，直接就知道要调用LoginPresenter的doLogin方法，其实它一点都不智能，如果LoginPresenter有N个方法添加了ActionProcess注解，那么LoginPresenter\_MMVPActionProcessor的handleAction方法就会是这个样子

	  @Override
	  public boolean handleAction(@NonNull MMVPAction action) {
	    return method1(action)||method2(action)||method3(action)||...||methodN(action);
	  }

所以我在MMVPAnnotationProcessor里有这样一句注释**由于无法预知即将调用的方法，只能把重写的方法全部执行一遍，重写方法里有判断可以避免错误执行,并且只要有某个方法返回了true，后续方法将不再执行。或许这样也不比反射执行的风险小吧**。这就是在我已经快要放弃用APT技术解决反射问题时想到的一个笨办法，虽然解决了我遇到的问题，但是我内心是拒绝这样去实现的。

为什么说无法预知即将调用的方法？因为这些代码都是我主动去生成的，即便我知道这里一定有一个handleAction方法，其参数MMVPAction，但是MMVPAction在我面前他就是一个字符串，我无法得到里面的内容，也就无法根据MMVPAction的action精准匹配要执行的方法。

当然，如果了解我MMVP项目的童鞋肯定会发现一个更简单的方法，那就是我的View和Presenter都实现了IMMVPActionHandler接口的，LoginPresenter具体实现了handleAction方法，所以我们的LoginPresenter\_MMVPActionProcessor的handleAction方法就可以简化成这个样子

		  @Override
		  public boolean handleAction(@NonNull MMVPAction action) {
		    return target.handleAction(action);
		  }

甚至连反射和这里的APT技术都不需要，直接在MMVPArtist的handleActionFromView或handleActionFromPresenter方法里，直接调用IMMVPActionHandler的handleAction方法。只是这样做的前提是，Presenter和View都必须正确实现handleAction方法。

关于用APT技术解决反射调用性能损耗的问题，以及对项目的改造，大概就是这个样子啦，如果看文章觉得不清楚很晦涩，那就去文章末尾贴出的项目地址查看源码，或许会好一些吧。如果大家有什么意见或建议，希望各位不吝赐教，感谢！

# 三、结语

今天呢，我们一起探讨了用APT技术解决反射调用性能损耗的问题，虽然结果并不完美，甚至可以说的上是有点牵强，但是，这都不重要，重要的是我们又学会了用一些不同的技术解决相同的问题，用不同的思维方式去思考同一个问题，我想，这应该是值得的。

通过一次次的写博客，我发现一个道理，没有什么东西是一蹴而就的，你们现在看到源码，寥寥数行，可是我都不记得自己改了多少次，才仅仅有这样的结果。这或许是一种磨砺，也或许是一个程序猿的乐趣吧。

最后，当然是所有程序猿都喜欢的代码啦

>[MMVP-github项目地址](https://github.com/ykbjson/TestStandardMvp)






