---
layout:     post
title:      ButterKnife源码解读
subtitle:
date:       2019-12-16
author:     parting_soul
header-img: img/butterknife_bg.jpg
catalog: true
tags:
    - Android
    - APT
    - 源码分析
---

[TOC]

# 一. 概述

[ButterKnife](https://github.com/JakeWharton/butterknife) 是一个依赖注入框架，主要用于绑定View、一些View的事件等等，可以大大减少findViewById以及设置View事件监听器的代码，并且框架的性能相比于传统写法也没有什么太大的损耗。

# 二. 简单使用

ButterKnife的用法十分简单

导入依赖

```groovy
android {
  ...
  // Butterknife requires Java 8.
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}

dependencies {
  implementation 'com.jakewharton:butterknife:10.2.0'
  annotationProcessor 'com.jakewharton:butterknife-compiler:10.2.0'
}
```

在代码中使用

在需要注入值的属性或者方法上使用对应的注解修饰，然后在onCreate中启动注解处理工具，然后在onDestroy中移除对属性的绑定。

```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.cl)
    ConstraintLayout cl;

    @BindView(R.id.bt)
    Button bt;

    @BindString(R.string.app_name)
    String appName;

    @BindDrawable(R.drawable.ic_launcher_background)
    Drawable icon;

    @BindColor(R.color.colorAccent)
    int color;

    private Unbinder mUnbinder;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mUnbinder = ButterKnife.bind(this);
    }

    @OnClick(R.id.bt)
    void onClick() {
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mUnbinder.unbind();
    }
}
```

# 三. 源码分析

我们将使用上方的例子来讲解ButterKnife，我们先来看看我们在例子中做了哪些操作

- 通过BindView将布局中对应id的布局绑定到对应的View
- 通过BindString将一个资源字符串绑定到一个String类型的属性
- 通过BindDrawable将一个资源类型的图片绑定到一个Drawable的属性
- 通过BindColor将一个颜色绑定到一个int类型的属性
- 通过OnClick为R.id.bt对用的控件设置了点击事件的方法回调

```java
public class MainActivity extends AppCompatActivity {

    @BindView(R.id.cl)
    ConstraintLayout cl;

    @BindView(R.id.bt)
    Button bt;

    @BindString(R.string.app_name)
    String appName;

    @BindDrawable(R.drawable.ic_launcher_background)
    Drawable icon;

    @BindColor(R.color.colorAccent)
    int color;

    private Unbinder mUnbinder;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mUnbinder = ButterKnife.bind(this);
    }

    @OnClick(R.id.bt)
    void onClick() {
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mUnbinder.unbind();
    }
}
```

## 3.1 ViewBinding 类

上述代码使用了ButterKnife注解，在编译时就会被系统扫描到，然后在同级的包名目录下会生成一个[类名]_ViewBinding的一个类。

![image-20191216103225889](https://i.postimg.cc/Njs0zt7B/image.png)

现在我们来看生成的MainActivity_ViewBinding类，这个类代码十分简单。

- 属性： 有一个MainActivity的属性，用于持有对MainActivity对象的引用。
- 构造方法：两个构造方法，在构造方法内对MainActivity中使用注解修饰的属性进行赋值或为指定的控件设置监听器

```java
public class MainActivity_ViewBinding implements Unbinder {
  private MainActivity target;

  private View view7f070042;

  @UiThread
  public MainActivity_ViewBinding(MainActivity target) {
    this(target, target.getWindow().getDecorView());
  }

  @UiThread
  public MainActivity_ViewBinding(final MainActivity target, View source) {
    this.target = target;

    View view;

    // 找到id为R.id.cl的控件，然后将其赋值给MainActivity中的名为cl的属性，也就是使用BindView修饰，注解值为R.id.cl的控件
    target.cl = Utils.findRequiredViewAsType(source, R.id.cl, "field 'cl'", ConstraintLayout.class);
    // 同理 找到id为.id.bt的控件，将其赋值给MainActivity中的名为bt的属性
    view = Utils.findRequiredView(source, R.id.bt, "field 'bt' and method 'onClick'");
    target.bt = Utils.castView(view, R.id.bt, "field 'bt'", Button.class);
    view7f070042 = view;
    //为R.id.bt的控件设置监听器
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.onClick();
      }
    });

    Context context = source.getContext();
    Resources res = context.getResources();
    // 将R.color.colorAccent对应的颜色赋值给MainActivity中名为color的属性
    target.color = ContextCompat.getColor(context, R.color.colorAccent);
    // 将R.drawable.ic_launcher_background对应的图片赋值给MainActivity中名为icon的属性
    target.icon = ContextCompat.getDrawable(context, R.drawable.ic_launcher_background);
    // 将R.string.app_name 对应的字符串值赋值给MainActivity中名为appName的属性
    target.appName = res.getString(R.string.app_name);
  }

  @Override
  @CallSuper
  public void unbind() {
    MainActivity target = this.target;
    if (target == null) throw new IllegalStateException("Bindings already cleared.");
    this.target = null;

    // 移除被BindView修饰的View控件的引用
    target.cl = null;
    target.bt = null;
    // 移除监听器
    view7f070042.setOnClickListener(null);
    view7f070042 = null;
  }
}
```

这里以id为R.id.bt的控件为例进行讲解：

可以看到这边通过Utils这个工具类去获取这个id的控件，findRequiredView有三个参数，第一个source为DecorView对象，是MainActivity的顶级容器，第二个参数是需要获取控件的id，第三个参数为描述信息

```java
public MainActivity_ViewBinding(final MainActivity target, View source) {
 		...
    view = Utils.findRequiredView(source, R.id.bt, "field 'bt' and method 'onClick'");
}
```

我们来看看Utils工具类，不难发现，最终还是通过findViewById从source(这里是MainActivity的顶层View)获取到了控件，然后将其返回。

```java
public static <T> T findRequiredViewAsType(View source, @IdRes int id, String who,
                                           Class<T> cls) {
  View view = findRequiredView(source, id, who);
  return castView(view, id, who, cls);
}

public static View findRequiredView(View source, @IdRes int id, String who) {
  	//可以看到最终还是通过findViewById从source中找到View
    View view = source.findViewById(id);
    if (view != null) {
      return view;
    }
   ...
}
```

我们再回到MainActivity_ViewBinding的构造方法中，由于在例子中我们在属性名为bt的按钮设置了BindView的注解，所以这里将找到的控件赋值给MainActivity的bt属性，这里由于是直接赋值，因此被注解修饰的属性访问权限不能为private。

由于在例子中我们还给R.id.bt的控件设置了点击事件的回调方法，所以这边找到R.id.bt的控件之后，还需为其设置点击事件，并在回调方法中回调MainActivity的onClick方法。因此bt对应的控件被点击，MainActivity的onClick会被回调。

```java
public MainActivity_ViewBinding(final MainActivity target, View source) {
 		...
    //获取id为R.id.bt的View
    view = Utils.findRequiredView(source, R.id.bt, "field 'bt' and method 'onClick'");
  	target.bt = Utils.castView(view, R.id.bt, "field 'bt'", Button.class);
    view7f070042 = view;
    //为R.id.bt的控件设置监听器
    view.setOnClickListener(new DebouncingOnClickListener() {
      @Override
      public void doClick(View p0) {
        target.onClick();
      }
    });
}
```

最后再讲下unbind这个方法，这个方法主要是用于移除之前绑定控件或者监听器的引用，在Activity销毁前调用，用于及时回收不必要的内存。

```java
@Override
@CallSuper
public void unbind() {
  MainActivity target = this.target;
  if (target == null) throw new IllegalStateException("Bindings already cleared.");
  this.target = null;

  // 移除被BindView修饰的View控件的引用
  target.cl = null;
  target.bt = null;
  // 移除监听器
  view7f070042.setOnClickListener(null);
  view7f070042 = null;
}
```

好了，MainActivity_ViewBinding类算是讲解完了，这里做一个小结

- Activity或者Fragment中有使用BindView等注解，在项目编译时，ButterKnife会在同级包目录下生成[类型]_ViewBinding类(一个类对应一个ViewBinding类)
- 在生成的类的构造方法中会对目标类中所有被注解修饰的属性进行值的注入，若存在监听器相关的注解，则同时为控件创建事件监听器，并回调目标类对应的方法

## 3.2 ButterKnife.bind

看到这，相信大家心里都有疑问，虽然生成了这样一个类，它在构造方法中完成了值的注入，但是也没看见ButterKnife调用啊。

我们再回到例子，看看是不是还漏了什么东西。对的，想必你也发现了，我们在onCreate方法中将MainActivity对象注入到了ButterKnife中，我们来看看这里做了什么操作。

```java
public class MainActivity extends AppCompatActivity {
		@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mUnbinder = ButterKnife.bind(this);
    }
}
```

ButterKnife这个类的代码也比较少

```java
public final class ButterKnife {
  private ButterKnife() {
    throw new AssertionError("No instances.");
  }

  private static final String TAG = "ButterKnife";
  private static boolean debug = false;

  // 用于缓存_ViewBinding类的构造方法，避免每次通过反射去获取
  @VisibleForTesting
  static final Map<Class<?>, Constructor<? extends Unbinder>> BINDINGS = new LinkedHashMap<>();

  /** Control whether debug logging is enabled. */
  public static void setDebug(boolean debug) {
    ButterKnife.debug = debug;
  }


  // Activity注入
  @NonNull @UiThread
  public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return bind(target, sourceView);
  }

  // View注入，例如自定义View中使用注解
  @NonNull @UiThread
  public static Unbinder bind(@NonNull View target) {
    return bind(target, target);
  }


  // 对话框注入
  @NonNull @UiThread
  public static Unbinder bind(@NonNull Dialog target) {
    View sourceView = target.getWindow().getDecorView();
    return bind(target, sourceView);
  }


  // Activity注入
  @NonNull @UiThread
  public static Unbinder bind(@NonNull Object target, @NonNull Activity source) {
    View sourceView = source.getWindow().getDecorView();
    return bind(target, sourceView);
  }

  // 对话框注入
  @NonNull @UiThread
  public static Unbinder bind(@NonNull Object target, @NonNull Dialog source) {
    View sourceView = source.getWindow().getDecorView();
    return bind(target, sourceView);
  }

  @NonNull @UiThread
  public static Unbinder bind(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);
    if (constructor == null) {
      return Unbinder.EMPTY;
    }
    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      return constructor.newInstance(target, source);
    } catch (Exception e) {
        ...
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
  }

  // 寻找_ViewBinding类的构造方法
  @Nullable @CheckResult @UiThread
  private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    // 缓存中存在该构造方法直接返回
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null || BINDINGS.containsKey(cls)) {
      return bindingCtor;
    }

    // 过滤系统类
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")
        || clsName.startsWith("androidx.")) {
      return null;
    }
    try {
      // 加载ViewBinding类到内存
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      // 获取ViewBinding类的构造方法
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
    } catch (Exception e) {
        ...
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    // 获取成功后把狗杂方法进行缓存
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
  }
}
```

这里继续使用上述的例子进行讲解

在MainActivity的onCreate中调用了bind方法，将Activity注入到ButterKnife，可以看到最终调用了两个参数的bind方法，这边target就是MainActivity对象，source为MainActivity的DecorView。

可以看到在bind方法中根据targetClass(这里是指MainActivity)去寻找_ViewBinding类的构造方法

```java
// Activity注入
@NonNull @UiThread
public static Unbinder bind(@NonNull Activity target) {
  // 得到顶层容器DecorView
  View sourceView = target.getWindow().getDecorView();
  return bind(target, sourceView);
}

@NonNull @UiThread
public static Unbinder bind(@NonNull Object target, @NonNull View source) {
  Class<?> targetClass = target.getClass();
  // 寻找_ViewBinding的构造方法
  Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);
  if (constructor == null) {
    return Unbinder.EMPTY;
  }
  try {
    return constructor.newInstance(target, source);
  } catch (Exception e) {
    ...
      throw new RuntimeException("Unable to create binding instance.", cause);
  }
}
```

接下来来看看findBindingConstructorForClass方法的实现

- 首先根据targetClass去缓存中寻找是否存在targetClass对用的_ViewBinding类的构造方法，缓存中存在就直接返回
- 缓存中不存在，过滤一些系统的类后，根据targetClass的得到对应 _ViewBinding的全类名，将该_ViewBinding类通过ClassLoader加载到内存，然后通过Class去获取类对应的构造方法
- 获取到构造方法之后将构造方法加入到缓存Map中，然后返回该构造方法

```java
// 寻找_ViewBinding类的构造方法
  @Nullable @CheckResult @UiThread
  private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    // 缓存中存在该构造方法直接返回
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null || BINDINGS.containsKey(cls)) {
      return bindingCtor;
    }

    // 过滤系统类
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")
        || clsName.startsWith("androidx.")) {
      return null;
    }
    try {
      // 加载ViewBinding类到内存
      Class<?> bindingClass = cls.getClassLoader().loadClass(clsName + "_ViewBinding");
      // 获取ViewBinding类的构造方法
      bindingCtor = (Constructor<? extends Unbinder>) bindingClass.getConstructor(cls, View.class);
    } catch (Exception e) {
        ...
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    // 获取成功后把构造方法进行缓存
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
  }
```

​		现在再回到bind方法中，通过findBindingConstructorForClass方法拿到了ViewBinding类的构造方法，然后通过构造方法创建了一个ViewBinding的实例(这里是MainActivity_ViewBinding类的实例)。

​		我们在前一个小结中已经提到，MainActivity中的值注入和监听器的绑定是在MainActivity_ViewBinding的构造方法中完成的，而这个MainActivity_ViewBinding实例就是在ButterKnife.bind方法被调用时创建的，此时整个依赖注入流程就已经完成了。

```java
 public static Unbinder bind(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);
    if (constructor == null) {
      return Unbinder.EMPTY;
    }
    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      return constructor.newInstance(target, source);
    } catch (Exception e) {
        ...
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
  }
```

小结： 通过ButterKnife的bind方法生成了一个ViewBinding类的实例，然后在ViewBinding类中进行了被注解修饰属性值的注入以及事件监听回调方法的绑定。

## 3.3 注解处理器

​	是不是恍然大悟，原来ButterKnife帮我们做了这些事情啊，其实ButterKnife帮我们做的事情远不止这些，前面我们提到只要你在Activity或者Fragment等地方正确的使用了ButterKnife的注解，在编译时ButterKnife就会为这些使用了注解的Activity或者Fragment生成对应的ViewBinding类，那么这种操作是如何实现的呢，这就要用到APT(Annotation Processing Tool)技术了。

​	不了解APT的同学可以戳这里 [APT入门](https://partingsoul.github.io/2019/11/26/APT技术/)

---

### 3.3.1 整体流程

​	注解处理器在编译时可以获取到注解的相关信息，例如被注解修饰的类元素，字段元素等等，它在ButterKnife中的作用就是通过根据注解的相关信息，在事先定义好的模板类中填充代码，最终将这个类信息输出到文件中。

​	首先看一下注解处理的核心类 **ButterKnifeProcessor**，该类在编译时会被系统扫描到，从而执行注解处理的代码。

支持的注解类型

```java
//支持的监听器注解类型
private static final List<Class<? extends Annotation>> LISTENERS = Arrays.asList(//
  OnCheckedChanged.class, //
  OnClick.class, //
  OnEditorAction.class, //
  OnFocusChange.class, //
  OnItemClick.class, //
  OnItemLongClick.class, //
  OnItemSelected.class, //
  OnLongClick.class, //
  OnPageChange.class, //
  OnTextChanged.class, //
  OnTouch.class //
);

private Set<Class<? extends Annotation>> getSupportedAnnotations() {
  Set<Class<? extends Annotation>> annotations = new LinkedHashSet<>();
  annotations.add(BindAnim.class);
  annotations.add(BindArray.class);
  annotations.add(BindBitmap.class);
  annotations.add(BindBool.class);
  annotations.add(BindColor.class);
  annotations.add(BindDimen.class);
  annotations.add(BindDrawable.class);
  annotations.add(BindFloat.class);
  annotations.add(BindFont.class);
  annotations.add(BindInt.class);
  annotations.add(BindString.class);
  annotations.add(BindView.class);
  annotations.add(BindViews.class);
  annotations.addAll(LISTENERS);
  return annotations;
}

@Override
public Set<String> getSupportedAnnotationTypes() {
    Set<String> types = new LinkedHashSet<>();
    for (Class<? extends Annotation> annotation : getSupportedAnnotations()) {
      types.add(annotation.getCanonicalName());
    }
    return types;
}

```

process方法为注解处理的核心方法，该方法主要做了两个事情

- 寻找并且解析被注解修饰的元素，返回一个类元素与它对应Viewbinding类信息的一个Map
- 遍历所有的ViewBinding类信息类，为每个被注解修饰的类元素创建一个ViewBinding类

```java
@Override
public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
  //寻找并且解析被注解修饰的元素，返回一个类元素与它对应viewbinding类信息的一个Map
  Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

  for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
    TypeElement typeElement = entry.getKey();
    BindingSet binding = entry.getValue();
    //生成被注解修饰类的viewbinding类Java代码
    JavaFile javaFile = binding.brewJava(sdk, debuggable);
    try {
      javaFile.writeTo(filer);
    } catch (IOException e) {
      error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
    }
  }

  return false;
}
```

### 3.3.2 注解编译器项目结构

在分析细节前，首先看一下ButterKnife注解编译器的项目结构

![ButterKnife注解编译器项目结构](https://i.postimg.cc/x1M9vTBg/butterknife-compiler.png)

每个类或接口的作用：

| 类/接口                    | 作用                                                         |
| :------------------------- | :----------------------------------------------------------- |
| BindingSet                 | ViewBinding类内部所有的信息(属性，方法等)，一个类对应一个BindingSet，同时具有生成模板代码的能力 |
| ButterKnifeProcessor       | 注解处理器                                                   |
| FieldAnimationBinding      | 被注解修饰的动画属性信息                                     |
| FieldCollectionViewBinding | 被BindViews修饰的控件组属性信息                              |
| FieldDrawableBinding       | 被注解修饰的Drawable属性信息(drawbable对应的id，属性的名称等) |
| FieldResourceBinding       | 被注解修饰的资源属性信息(这里的资源包括bitmap,dimen,color,string等) |
| FieldTypefaceBinding       | 被注解修饰的字体属性的信息（属性的名字，字体的id，字体的风格） |
| FieldViewBinding           | 被注解修饰的控件属性的信息(控件名、类型、是否必须)           |
| Id                         | 资源id信息封装，包括id的值、id的代码                         |
| MemberViewBinding          | 被注解修饰成员信息接口(成员包括属性和方法)                   |
| MethodViewBinding          | 被注解修饰方法的信息(方法名、方法的参数、返回值)             |
| Parameter                  | 监听器方法形参的封装（形参的类型，形参的位置）               |
| ResourceBinding            | 资源绑定信息接口                                             |
| ViewBinding                | 控件绑定信息，存储了被BindView修饰控件的信息以及该控件的事件监听信息，一个View对应一个ViewBinding |

从上可知 ButterKnife注解编译器中的类可以分为三类

- 注解处理器

- 用于保存**被注解修饰属性或者方法的信息**描述类
- BindingSet 存储了一个ViewBinding类所有元素信息，并且提供生成模板代码的能力

### 3.3.3 process方法

#### 1. 解析注解获得ViewBinding类信息集合

我们现在开始分析process方法的细节，在process中首先调用了findAndParseTargets方法，去解析所有被ButterKnife修饰的注解，最终返回一个Map，Map的Key存储被ButterKnife注解修饰成员的类，value保存了ViewBinding类相关信息。

```java
  //寻找并且解析被注解修饰的元素，返回一个类元素与它对应viewbinding类信息的一个Map
  Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);
```

进入findAndParseTargets方法，这个方法主要有两个作用：

- 解析各种ButterKnife注解
- 为每一个被注解修饰成员元素的父元素(类元素)生成对应的ViewBinding类结构信息(BindingSet)

这边先从解析注解开始，以解析BindView注解为例

```java
private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {
    Map<TypeElement, BindingSet.Builder> builderMap = new LinkedHashMap<>();
    // 用于保存被注解修饰成员元素的父元素(这里是类元素)，ButterKnife注解是用于修饰字段或者方法的
    Set<TypeElement> erasedTargetNames = new LinkedHashSet<>();


    // ... 省略处理其他处理注解的代码

    // 遍历所有被BindView修饰的元素
    for (Element element : env.getElementsAnnotatedWith(BindView.class)) {
        try {
          	// 解析该元素以及注解
            parseBindView(element, builderMap, erasedTargetNames);
        } catch (Exception e) {
            logParsingError(element, BindView.class, e);
        }
    }

    ...
    return bindingMap;
}
```

parseBindView方法主要是用于解析BindView注解，得到被注解修饰属性的信息，然后将其保存起来

执行流程：

1. 检查注解修饰的合法性(包括被注解修饰属性的访问权限以及注解修饰的类合法性)

   ```java
   private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap, Set<TypeElement> erasedTargetNames) {
           //得到当前字段元素的父元素，也就是类元素
           TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

           //判断元素的访问权限的合法性以及是否判断在系统的api类中使用了ButterKnife注解
           boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element)
                   || isBindingInWrongPackage(BindView.class, element);private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
                                  Set<TypeElement> erasedTargetNames) {

           // 如果注解修饰的是泛型
           TypeMirror elementType = element.asType();
           if (elementType.getKind() == TypeKind.TYPEVAR) {
               /**
                *  找到泛型类型的上界 例如
                * @BindView(R.id.xx)
                * T view;
                * 其中T的类型为 ? extend View
                */
               TypeVariable typeVariable = (TypeVariable) elementType;
               elementType = typeVariable.getUpperBound();
           }
           // 得到类元素的全名
           Name qualifiedName = enclosingElement.getQualifiedName();
           // 得到属性的名字
           Name simpleName = element.getSimpleName();
           if (!isSubtypeOfType(elementType, VIEW_TYPE) && !isInterface(elementType)) {
               // 若当前类不是View类的子类并且类型元素不是接口
               if (elementType.getKind() == TypeKind.ERROR) {
                   note(element, "@%s field with unresolved type (%s) "
                                   + "must elsewhere be generated as a View or interface. (%s.%s)",
                           BindView.class.getSimpleName(), elementType, qualifiedName, simpleName);
               } else {
                   error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
                           BindView.class.getSimpleName(), qualifiedName, simpleName);
                   hasError = true;
               }
           }

           // 存在错误则直接返回
           if (hasError) {
               return;
           }
   		....
   }
   ```

   访问权限检查: 属性或者方法的访问权限不能为private或者static

   ```java
   // 判断访问权限是否合法
       private boolean isInaccessibleViaGeneratedCode(Class<? extends Annotation> annotationClass,
                                                   String targetThing, Element element) {
           boolean hasError = false;
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

           //属性或者方法的访问权限不能是private或者static
           Set<Modifier> modifiers = element.getModifiers();
           if (modifiers.contains(PRIVATE) || modifiers.contains(STATIC)) {
               error(element, "@%s %s must not be private or static. (%s.%s)",
                       annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                       element.getSimpleName());
               hasError = true;
           }

           // 元素的父元素必须为类
           if (enclosingElement.getKind() != CLASS) {
               error(enclosingElement, "@%s %s may only be contained in classes. (%s.%s)",
                       annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                       element.getSimpleName());
               hasError = true;
           }

           // 类元素不能为private
           if (enclosingElement.getModifiers().contains(PRIVATE)) {
               error(enclosingElement, "@%s %s may not be contained in private classes. (%s.%s)",
                       annotationClass.getSimpleName(), targetThing, enclosingElement.getQualifiedName(),
                       element.getSimpleName());
               hasError = true;
           }

           return hasError;
       }
   ```

   注解修饰属性对应的类合法性： 被注解修饰的属性对应的类不能为系统类

   ```java
   // 是否绑定的类不合法，无法绑定系统类
       private boolean isBindingInWrongPackage(Class<? extends Annotation> annotationClass,
                                            Element element) {
           TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
        String qualifiedName = enclosingElement.getQualifiedName().toString();

           //只要被注解修饰属性对应的类为系统相关类则为不合法
           if (qualifiedName.startsWith("android.")) {
               error(element, "@%s-annotated class incorrectly in Android framework package. (%s)",
                       annotationClass.getSimpleName(), qualifiedName);
               return true;
           }
           if (qualifiedName.startsWith("java.")) {
               error(element, "@%s-annotated class incorrectly in Java framework package. (%s)",
                       annotationClass.getSimpleName(), qualifiedName);
               return true;
           }

           return false;
       }
   ```

   注解修饰的属性类型合法性：被BindView修饰的元素需要是View的子类或者接口

   ```java
   // 得到类元素的全名
   Name qualifiedName = enclosingElement.getQualifiedName();
// 得到属性的名字
   Name simpleName = element.getSimpleName();
if (!isSubtypeOfType(elementType, VIEW_TYPE) && !isInterface(elementType)) {
     // 若当前类不是View类的子类并且类型元素不是接口
     if (elementType.getKind() == TypeKind.ERROR) {
       note(element, "@%s field with unresolved type (%s) "
            + "must elsewhere be generated as a View or interface. (%s.%s)",
            BindView.class.getSimpleName(), elementType, qualifiedName, simpleName);
     } else {
       error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
             BindView.class.getSimpleName(), qualifiedName, simpleName);
       hasError = true;
     }
   }
   ```

2. 将被注解的属性信息保存

对注解的合法性检查完之后，就是要取出注解里的布局id以及对应属性的信息，将其保存起来

- 在缓存中寻找当前属性对应的类是否已经存在，若存在则校验对应的id是否重复绑定；若不存在，创建当前属性对应类的BindingSet
- 将布局id以及被注解修饰属性的信息包装成FieldViewBinding，并将该包装类加入到BindingSet中
- 将被注解修饰属性的父元素(也就是类元素)保存起来

```java
// 得到BindView内的id值
int id = element.getAnnotation(BindView.class).value();
//在缓存中取出BindingSet
BindingSet.Builder builder = builderMap.get(enclosingElement);
// 将id值封装成Id对象
Id resourceId = elementToId(element, BindView.class, id);
if (builder != null) {
  //已经存在类信息包装类
  String existingBindingName = builder.findExistingBindingName(resourceId);
  if (existingBindingName != null) {
    // 该资源id已经被绑定过了
    error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
          BindView.class.getSimpleName(), id, existingBindingName,
          enclosingElement.getQualifiedName(), element.getSimpleName());
    return;
  }
} else {
  //不存在则创建
  builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
}

String name = simpleName.toString();
TypeName type = TypeName.get(elementType);
// 判断字段的是是否能为空
boolean required = isFieldRequired(element);
// 将字段信息添加到类信息包装类中
builder.addField(resourceId, new FieldViewBinding(name, type, required));

//将被注解的元素的类元素保存起来
erasedTargetNames.add(enclosingElement);
```

完整代码

```java
private void parseBindView(Element element, Map<TypeElement, BindingSet.Builder> builderMap,
                               Set<TypeElement> erasedTargetNames) {
        //得到当前字段元素的上级类元素
        TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();

        //判断元素的访问权限的合法性以及是否判断在系统的api类中使用了ButterKnife注解
        boolean hasError = isInaccessibleViaGeneratedCode(BindView.class, "fields", element)
                || isBindingInWrongPackage(BindView.class, element);

        // 如果注解修饰的是泛型
        TypeMirror elementType = element.asType();
        if (elementType.getKind() == TypeKind.TYPEVAR) {
            /**
             *  找到泛型类型的上界 例如
             * @BindView(R.id.xx)
             * T view;
             * 其中T的类型为 ? extend View
             */
            TypeVariable typeVariable = (TypeVariable) elementType;
            elementType = typeVariable.getUpperBound();
        }
        // 得到类元素的全名
        Name qualifiedName = enclosingElement.getQualifiedName();
        // 得到属性的名字
        Name simpleName = element.getSimpleName();
        if (!isSubtypeOfType(elementType, VIEW_TYPE) && !isInterface(elementType)) {
            // 若当前类不是View类的子类并且类型元素不是接口
            if (elementType.getKind() == TypeKind.ERROR) {
                note(element, "@%s field with unresolved type (%s) "
                                + "must elsewhere be generated as a View or interface. (%s.%s)",
                        BindView.class.getSimpleName(), elementType, qualifiedName, simpleName);
            } else {
                error(element, "@%s fields must extend from View or be an interface. (%s.%s)",
                        BindView.class.getSimpleName(), qualifiedName, simpleName);
                hasError = true;
            }
        }

        // 存在错误则直接返回
        if (hasError) {
            return;
        }

        // 得到BindView内的id值
        int id = element.getAnnotation(BindView.class).value();
        //在缓存中取出BindingSet
        BindingSet.Builder builder = builderMap.get(enclosingElement);
        // 将id值封装成Id对象
        Id resourceId = elementToId(element, BindView.class, id);
        if (builder != null) {
            //已经存在类信息包装类
            String existingBindingName = builder.findExistingBindingName(resourceId);
            if (existingBindingName != null) {
                //之前已经添加了字段信息
                error(element, "Attempt to use @%s for an already bound ID %d on '%s'. (%s.%s)",
                        BindView.class.getSimpleName(), id, existingBindingName,
                        enclosingElement.getQualifiedName(), element.getSimpleName());
                return;
            }
        } else {
            //不存在则创建
            builder = getOrCreateBindingBuilder(builderMap, enclosingElement);
        }

        String name = simpleName.toString();
        TypeName type = TypeName.get(elementType);
        // 判断字段的是是否能为空
        boolean required = isFieldRequired(element);
        // 将字段信息添加到类信息包装类中
        builder.addField(resourceId, new FieldViewBinding(name, type, required));

        //将被注解的元素的类元素保存起来
        erasedTargetNames.add(enclosingElement);
}
```

小结：

- 解析BindView注解首先会校验注解的合法性，依次是权限合法性，类合法性，类型合法性，不合法则返回
- 判断缓存中是否存在属性对应类的BindingSet（ViewBinding描述信息），存在则需要判断id是否被重复绑定，不存在则创建
- 将BindView的值以及对应修饰属性的信息封装成FieldViewBinding加入到BindingSet中



解析完所有的注解之后，会得到一个Map，这个Map的Key为被ButterKnife注解修饰的类元素，Value为ViewBinding类信息包装类。在使用ButterKnife时，我们为了不用每次都书写ButterKnife.bind方法，通常会把这些代码放在BaseActivity中，或者在父类中绑定一些公有的属性，也就是存在父类和子类同时使用ButterKnife注解的情况。因此我们生成的ViewBinding类也会存在一些继承关系，所以需要为每一个ViewBinding类确定其继承关系，找到它可能存在的父类。

- 找到所有存在属性被注解修饰的父类元素
- 为每一个元素和存在的父元素建立对应关系，最终得到所有的ViewBinding信息集合

```java
 private Map<TypeElement, BindingSet> findAndParseTargets(RoundEnvironment env) {

    ...
    //解析处理所有ButterKnife注解

// 找到所有存在属性被注解修饰的父类元素
    Map<TypeElement, ClasspathBindingSet> classpathBindings =
            findAllSupertypeBindings(builderMap, erasedTargetNames);

    //把类信息包装集合放入队列中
    Deque<Map.Entry<TypeElement, BindingSet.Builder>> entries =
            new ArrayDeque<>(builderMap.entrySet());
    // 一个类对应一个类信息包装类，也就是每一个使用了ButterKnife注解的类对应viewBinding类的信息
    Map<TypeElement, BindingSet> bindingMap = new LinkedHashMap<>();
    while (!entries.isEmpty()) {
        //队列头出列
        Map.Entry<TypeElement, BindingSet.Builder> entry = entries.removeFirst();

        TypeElement type = entry.getKey();
        BindingSet.Builder builder = entry.getValue();

        //查找当前类是否存在父类被注解修饰
        TypeElement parentType = findParentType(type, erasedTargetNames, classpathBindings.keySet());
        if (parentType == null) {
            //当前类不存在父类使用了ButterKnife注解，直接保存当前类元素和对应类信息包装类
            bindingMap.put(type, builder.build());
        } else {
            // 存在父类使用了ButterKnife注解
            BindingInformationProvider parentBinding = bindingMap.get(parentType);
            if (parentBinding == null) {
                parentBinding = classpathBindings.get(parentType);
            }
            if (parentBinding != null) {
                //为当前viewBinding类设置父类信息
                builder.setParent(parentBinding);
                bindingMap.put(type, builder.build());
            } else {
                // 存在父类使用了注解的情况，但父类还未被处理，重新将元素加入队列尾部，延后处理
                entries.addLast(entry);
            }
        }
    }

    ...
    return bindingMap;
}
```

#### 2. 生成Java代码

在解析处理注解后得到了所有ViewBinding信息集合，现在需要将这些所有的ViewBinding生成具体的Java代码。

可以看到代码中遍历生成的集合，调用BindingSet的brewJava生成Java代码

```java
@Override
public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env) {
  //寻找并且解析被注解修饰的元素，返回一个类元素与它对应viewbinding类信息的一个Map
  Map<TypeElement, BindingSet> bindingMap = findAndParseTargets(env);

  for (Map.Entry<TypeElement, BindingSet> entry : bindingMap.entrySet()) {
    TypeElement typeElement = entry.getKey();
    BindingSet binding = entry.getValue();
    //生成被注解修饰类的viewbinding类Java代码
    JavaFile javaFile = binding.brewJava(sdk, debuggable);
    try {
      // 将Java类输出到文件
      javaFile.writeTo(filer);
    } catch (IOException e) {
      error(typeElement, "Unable to write binding for type %s: %s", typeElement, e.getMessage());
    }
  }

  return false;
}
```

进入brewJava方法，这里使用了javapoet框架来生成代码，createType方法用于生成类信息

```java
JavaFile brewJava(int sdk, boolean debuggable) {
  TypeSpec bindingConfiguration = createType(sdk, debuggable);
  return JavaFile.builder(bindingClassName.packageName(), bindingConfiguration)
    .addFileComment("Generated code from Butter Knife. Do not modify!")
    .build();
}
```

进入createType方法

执行流程：

1. 创建类结构，若存在父类则继承父类，不存在则实现接口
2. 判断是否需要添加成员属性target字段，该字段在使用了BindView或者事件相关注解时target会被添加
3. 根据注入的类型添加只有target形参的构造方法
4. 判断创建的构造方法中是否需要添加View形参，若没有，则默认添加一个T_ViewBinding(T target, View source)两个参数的构造方法
5. 添加最终进行依赖注入和事件绑定的构造方法
6. 添加unbind方法

```java
private TypeSpec createType(int sdk, boolean debuggable) {
  //1. 创建类
  TypeSpec.Builder result = TypeSpec.classBuilder(bindingClassName.simpleName())
    .addModifiers(PUBLIC)
    .addOriginatingElement(enclosingElement);
  if (isFinal) {
    result.addModifiers(FINAL);
  }

  if (parentBinding != null) {
    //存在父类添加了ButterKnife注解，生成ViewBinding类同样继承父类对应的ViewBinding
    result.superclass(parentBinding.getBindingClassName());
  } else {
    // 不存在父类，实现Unbinder接口
    result.addSuperinterface(UNBINDER);
  }

  //2. 是否在ViewBinding中添加target属性，target是注入类的应用，在使用了BindView或者事件相关注解时target会被添加
  if (hasTargetField()) {
    result.addField(targetTypeName, "target", PRIVATE);
  }

  //3. 创建只有target形参的构造方法
  if (isView) {
    result.addMethod(createBindingConstructorForView());
  } else if (isActivity) {
    result.addMethod(createBindingConstructorForActivity());
  } else if (isDialog) {
    result.addMethod(createBindingConstructorForDialog());
  }

  if (!constructorNeedsView()) {
    // 4. 如果一个参数的构造方法中不要用到view,创建一个两个参数的构造方法，一个参数为target,另一个为view
    // T_ViewBinding(T target, View source),例如类没有使用BindView和事件相关注解，此时构造方法中不需要用到View，此时会自动创建这个两个参数的构造方法
    result.addMethod(createBindingViewDelegateConstructor());
  }
  //5. 添加具体绑定属性，方法的构造方法
  result.addMethod(createBindingConstructor(sdk, debuggable));

  //6. 添加unbind方法
  if (hasViewBindings() || parentBinding == null) {
    result.addMethod(createBindingUnbindMethod(result));
  }

  return result.build();
}
```

我们这边着重看下第5个步骤，创建了一个最終的构造方法，其他构造方法都会调用该构造方法。该构造方法是真正进行依赖注入和事件绑定的方法。

执行流程

1. 首先添加target形参，若存在事件绑定，则target参数需要用final修饰，因为事件监听器以匿名类方法设置
2. 根据绑定参数情况选择第二个参数的类型，若只是资源绑定，则第二个参数为Context，若存在控件或者方法绑定则第二个参数为View
3. 如果存在父类，则添加调用父类构造方法的代码
4. 若有控件或者方法绑定，target为成员属性，则需要给该target属性赋值
5. 存在控件绑定，进行控件值注入以及事件绑定
6. 存在资源绑定，则进行资源值注入

```java
private MethodSpec createBindingConstructor(int sdk, boolean debuggable) {
  MethodSpec.Builder constructor = MethodSpec.constructorBuilder()
    .addAnnotation(UI_THREAD)
    .addModifiers(PUBLIC);

  //1. 添加target形参
  if (hasMethodBindings()) {
    constructor.addParameter(targetTypeName, "target", FINAL);
  } else {
    constructor.addParameter(targetTypeName, "target");
  }

  //2. 添加第二个形参
  if (constructorNeedsView()) {
    // 添加View形参
    constructor.addParameter(VIEW, "source");
  } else {
    constructor.addParameter(CONTEXT, "context");
  }

  if (hasUnqualifiedResourceBindings()) {
    // Aapt can change IDs out from underneath us, just suppress since all will work at runtime.
    constructor.addAnnotation(AnnotationSpec.builder(SuppressWarnings.class)
                              .addMember("value", "$S", "ResourceType")
                              .build());
  }

  if (hasOnTouchMethodBindings()) {
    constructor.addAnnotation(AnnotationSpec.builder(SUPPRESS_LINT)
                              .addMember("value", "$S", "ClickableViewAccessibility")
                              .build());
  }

  // 3. 是否存在父类
  if (parentBinding != null) {
    //被注解类的类存在父类页使用了ButterKnife注解
    if (parentBinding.constructorNeedsView()) {
      //调用父类的构造方法
      constructor.addStatement("super(target, source)");
    } else if (constructorNeedsView()) {
      constructor.addStatement("super(target, source.getContext())");
    } else {
      constructor.addStatement("super(target, context)");
    }
    constructor.addCode("\n");
  }

   //4.添加target属性的赋值
  if (hasTargetField()) {
    constructor.addStatement("this.target = target");
    constructor.addCode("\n");
  }

  // 5. 控件和事件绑定
  if (hasViewBindings()) {
    if (hasViewLocal()) {
      //是否要创建View的局部变量，一般有给控件设置监听器注解时，会生成该局部变量
      constructor.addStatement("$T view", VIEW);
    }
    // 添加View控件的绑定代码
    for (ViewBinding binding : viewBindings) {
      addViewBinding(constructor, binding, debuggable);
    }
    // 添加使用BindViews绑定控件的代码
    for (FieldCollectionViewBinding binding : collectionBindings) {
      constructor.addStatement("$L", binding.render(debuggable));
    }

    if (!resourceBindings.isEmpty()) {
      constructor.addCode("\n");
    }
  }

  // 6. 资源绑定
  if (!resourceBindings.isEmpty()) {
    if (constructorNeedsView()) {
      constructor.addStatement("$T context = source.getContext()", CONTEXT);
    }
    if (hasResourceBindingsNeedingResource(sdk)) {
      constructor.addStatement("$T res = context.getResources()", RESOURCES);
    }
    for (ResourceBinding binding : resourceBindings) {
      constructor.addStatement("$L", binding.render(sdk));
    }
  }

  return constructor.build();
}
```

这边同样看第5个步骤，对控件和事件的绑定，这边遍历所有需要进行注入的控件信息，为每一个注入编写代码

```java
// 添加View控件的绑定代码
for (ViewBinding binding : viewBindings) {
  addViewBinding(constructor, binding, debuggable);
}
```

进入addViewBinding方法，这边分了两种情况

- 只存在控件的绑定
- 存在控件绑定和方法的绑定

对控件的绑定主要是通过编写findViewById的代码

```java
private void addViewBinding(MethodSpec.Builder result, ViewBinding binding, boolean debuggable) {
  if (binding.isSingleFieldBinding()) {
    //若只有属性的绑定，没有方法的绑定
    FieldViewBinding fieldBinding = requireNonNull(binding.getFieldBinding());

    // 对target中被注解修饰的属性进行赋值

    CodeBlock.Builder builder = CodeBlock.builder()
      .add("target.$L = ", fieldBinding.getName());

    boolean requiresCast = requiresCast(fieldBinding.getType());
    //debug && (requecast || isRequired)
    if (!debuggable || (!requiresCast && !fieldBinding.isRequired())) {
      //如果非debug状态或者字段不需要进行强转和字段并且不是必需赋值
      if (requiresCast) {
        //需要向下造型
        builder.add("($T) ", fieldBinding.getType());
      }
      builder.add("source.findViewById($L)", binding.getId().code);
    } else {
      //debug模式并且需要强转或者是参数是必需赋值的
      //  Utils.findRequiredViewAsType(source,[id], "描述", T)
      builder.add("$T.find", UTILS);
      builder.add(fieldBinding.isRequired() ? "RequiredView" : "OptionalView");
      if (requiresCast) {
        builder.add("AsType");
      }
      builder.add("(source, $L", binding.getId().code);
      if (fieldBinding.isRequired() || requiresCast) {
        //添加描述
        builder.add(", $S", asHumanDescription(singletonList(fieldBinding)));
      }
      if (requiresCast) {
        //控件的类型
        builder.add(", $T.class", fieldBinding.getRawType());
      }
      builder.add(")");
    }
    result.addStatement("$L", builder.build());
    return;
  }

  // View属性有值绑定并且有事件绑定
  List<MemberViewBinding> requiredBindings = binding.getRequiredBindings();
  // 给局部view变量赋值
  if (!debuggable || requiredBindings.isEmpty()) {
    result.addStatement("view = source.findViewById($L)", binding.getId().code);
  } else if (!binding.isBoundToRoot()) {
    result.addStatement("view = $T.findRequiredView(source, $L, $S)", UTILS,
                        binding.getId().code, asHumanDescription(requiredBindings));
  }

  // 添加属性绑定
  addFieldBinding(result, binding, debuggable);
  //添加事件绑定
  addMethodBindings(result, binding, debuggable);
}
```

接下来看看事件方法的绑定，这边代码比较多，我们挑主要的讲

- 一个View可能会设置多个事件监听，所以第一个for循环是遍历当前View需要注册的所有事件，为每一个事件创建一个对应的事件匿名类，然后将事件监听器绑定到控件。
- 每一个事件监听器可能会有多个回调方法，因此第第二层for循环就是给匿名事件类添加所有的回调方法
- 在target类中，同一个控件的id可以多次绑定在不同的方法上，因此第三层for循环就是在事件监听器回调方法中去回调每一个target中当前viewid绑定的方法
- target中事件绑定的方法可能会有多个形参，所以第四层循环是为了给调用target的事件绑定方法传入的所有参数

```java
private void addMethodBindings(MethodSpec.Builder result, ViewBinding binding,
                               boolean debuggable) {
    Map<ListenerClass, Map<ListenerMethod, Set<MethodViewBinding>>> classMethodBindings =
            binding.getMethodBindings();
    if (classMethodBindings.isEmpty()) {
        return;
    }

    // 控件是否可以为空，为空添加判断代码
    boolean needsNullChecked = binding.getRequiredBindings().isEmpty();
    if (needsNullChecked) {
        result.beginControlFlow("if (view != null)");
    }

    // 对存在事件监听的控件需要将控件设置为成员变量，以便在unbind及时移除控件的监听器
    String fieldName = "viewSource";
    String bindName = "source";
    if (!binding.isBoundToRoot()) {
        fieldName = "view" + Integer.toHexString(binding.getId().value);
        bindName = "view";
    }
    // 给成员变量赋值
    result.addStatement("$L = $N", fieldName, bindName);

    //1. 为当前view设置事件监听，遍历所有设置的监听方法
    for (Map.Entry<ListenerClass, Map<ListenerMethod, Set<MethodViewBinding>>> e
            : classMethodBindings.entrySet()) {
        ListenerClass listener = e.getKey();
        Map<ListenerMethod, Set<MethodViewBinding>> methodBindings = e.getValue();

        //创建一个事件匿名类
        TypeSpec.Builder callback = TypeSpec.anonymousClassBuilder("")
                .superclass(ClassName.bestGuess(listener.type()));

        //2. 给事件匿名类的所有回调方法绑定方法
        for (ListenerMethod method : getListenerMethods(listener)) {

            // 事件监听器接口需要实现的方法
            MethodSpec.Builder callbackMethod = MethodSpec.methodBuilder(method.name())
                    .addAnnotation(Override.class)
                    .addModifiers(PUBLIC)
                    .returns(bestGuess(method.returnType()));
            // 添加方法形参
            String[] parameterTypes = method.parameters();
            for (int i = 0, count = parameterTypes.length; i < count; i++) {
                callbackMethod.addParameter(bestGuess(parameterTypes[i]), "p" + i);
            }

            boolean hasReturnValue = false;
            CodeBlock.Builder builder = CodeBlock.builder();
            //得到需要绑定的方法
            Set<MethodViewBinding> methodViewBindings = methodBindings.get(method);
            if (methodViewBindings != null) {
                //3. 在接口实现方法中调用需要target中对用的回调方法
                for (MethodViewBinding methodBinding : methodViewBindings) {
                    if (methodBinding.hasReturnValue()) {
                        hasReturnValue = true;
                        builder.add("return "); // TODO what about multiple methods?
                    }
                    builder.add("target.$L(", methodBinding.getName());
                    List<Parameter> parameters = methodBinding.getParameters();
                    String[] listenerParameters = method.parameters();
                    //4. 添加方法参数
                    for (int i = 0, count = parameters.size(); i < count; i++) {
                        if (i > 0) {
                            builder.add(", ");
                        }

                        Parameter parameter = parameters.get(i);
                        int listenerPosition = parameter.getListenerPosition();

                        if (parameter.requiresCast(listenerParameters[listenerPosition])) {
                            if (debuggable) {
                                builder.add("$T.castParam(p$L, $S, $L, $S, $L, $T.class)", UTILS,
                                        listenerPosition, method.name(), listenerPosition, methodBinding.getName(), i,
                                        parameter.getType());
                            } else {
                                builder.add("($T) p$L", parameter.getType(), listenerPosition);
                            }
                        } else {
                            builder.add("p$L", listenerPosition);
                        }
                    }
                    builder.add(");\n");
                }
            }

            if (!"void".equals(method.returnType()) && !hasReturnValue) {
                builder.add("return $L;\n", method.defaultReturn());
            }

            callbackMethod.addCode(builder.build());
            callback.addMethod(callbackMethod.build());
        }

        boolean requiresRemoval = listener.remover().length() != 0;
        String listenerField = null;
        if (requiresRemoval) {
            TypeName listenerClassName = bestGuess(listener.type());
            listenerField = fieldName + ((ClassName) listenerClassName).simpleName();
            result.addStatement("$L = $L", listenerField, callback.build());
        }

         //给控件设置事件监听器
        String targetType = listener.targetType();
        if (!VIEW_TYPE.equals(targetType)) {
            //不是view,则需要强转为指定类型
            result.addStatement("(($T) $N).$L($L)", bestGuess(targetType), bindName,
                    listener.setter(), requiresRemoval ? listenerField : callback.build());
        } else {
            result.addStatement("$N.$L($L)", bindName, listener.setter(),
                    requiresRemoval ? listenerField : callback.build());
        }
    }

    if (needsNullChecked) {
        result.endControlFlow();
    }
}
```

至此BindView注解解析以及对应ViewBinding类代码生成整个流程就这样完成了。