---
layout:     post  
title:      "手写一个简单的 ButterKnife"  
subtitle:   "帮你生成代码"  
date:       2021-05-12 12:00:00  
author:     "GuoHao"  
header-img: "img/spring.jpg"  
catalog: true  
tags:  
    - Android  
    - APT 
    - 三方库 

---

参考工程：/Volumes/T1/As_projs/custom_draw/HenCoderPlus7-main/Apt/app/src/main/java/com/hencoder/apt/MainActivity.java

Github 的路径：https://github.com/rengwuxian/HenCoderPlus7 的 APT 工程

核心思路：
- 编译期，增加一个编译工具，检测到注解就生成辅助类。
    - 辅助类的构造函数内会绑定注解指定的 View。
- 运行期，借助 Binding.bind 函数，用反射的方式找到辅助类，并触发其构造函数。

## 前言

ButterKnife 是一个视图绑定库，采用 APT 计数，帮我们做了 findViewById 等工作，减轻我们的负担，提高效率，加快开发速度。但是目前，Android JetPack 的 ViewBinding、DataBinding 和 Compose 的连续推出，基本说明 ButterKnife 即将成为过去式，但是，其 APT 计数的使用，还是值得学习的。

## 工程概括

- app 模块
    - MainActivity
    - build/.../MainActivityBinding.java
- lib 模块
    - Binding 视图绑定库的主类
- lib-processor 模块
    - 注解处理器
    - src/main/resources/META-INF/xxx.Processor 配置文件
- lib-annotations 模块
    - 注解本身

## 具体实现

用注解标明控件字段，并调用视图绑定库：
位置：Apt/app/src/main/java/com/hencoder/apt/MainActivity.java，即 app 主模块中。

```
public class MainActivity extends AppCompatActivity {
  // 添加注解
  @BindView(R.id.textView) TextView textView;
  @BindView(R.id.layout) ViewGroup layout;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    // 注解初始化
    Binding.bind(this);
    textView.setText("HenCoder");
    layout.setBackgroundColor(Color.CYAN);
  }
}
```

反射绑定辅助类：
位置：Apt/lib/src/main/java/com/hencoder/lib/Binding.java，即在 lib 模块中。

```
// Binding.bind

public class Binding {
  public static void bind(Activity activity) {
    try {
      // 拿到辅助类的类型
      // 辅助类的类名为：原始类 + Binding
      Class bindingClass = Class.forName(activity.getClass().getCanonicalName() + "Binding");
      // 拿到构造函数
      Constructor constructor = bindingClass.getDeclaredConstructor(activity.getClass());
      // 构建一个实例，会触发其构造函数
      constructor.newInstance(activity);
    } catch (ClassNotFoundException e) {
      e.printStackTrace();
    } catch (InstantiationException e) {
      e.printStackTrace();
    } catch (InvocationTargetException e) {
      e.printStackTrace();
    } catch (NoSuchMethodException e) {
      e.printStackTrace();
    } catch (IllegalAccessException e) {
      e.printStackTrace();
    }
  }
}
```

辅助类的实现：
位置：Apt/app/build/generated/ap_generated_sources/debug/out/com/hencoder/apt/MainActivityBinding.java

```
// MainActivityBinding

public class MainActivityBinding {
  public MainActivityBinding(MainActivity activity) {
    activity.textView = activity.findViewById(2131165354);
    activity.layout = activity.findViewById(2131165284);
  }
}
```

## 注解处理器

#### 配置文件

实现注解处理器前，还需要在资源文件表明注解处理器的路径，
即在该文件 src/main/resources/META-INF/services/javax.annotation.processing.Processor 写上如下内容：

```
// 这就是类路径
com.hencoder.lib_processor.BindingProcessor
```

#### 入注解处理器

主工程引入注解处理器的写法：

```
// app 模块的 build.gradle 文件 
...
dependencies {
  implementation fileTree(dir: "libs", include: ["*.jar"])
  ...
  annotationProcessor project(':lib-processor')
}
```

#### 先看小例子

直接操作 filer 生成一个空文件的写法：

![屏幕快照 2021-06-10 上午12.01.37](/img/media/16232372586525/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-06-10%20%E4%B8%8A%E5%8D%8812.01.37.png)


#### 真正的实现

注解处理器的实现：

```
package com.hencoder.lib_processor;

import com.squareup.javapoet.ClassName;
import com.squareup.javapoet.JavaFile;
import com.squareup.javapoet.MethodSpec;
import com.squareup.javapoet.TypeSpec;

...

public class BindingProcessor extends AbstractProcessor {
  Filer filer;

  @Override
  public synchronized void init(ProcessingEnvironment processingEnvironment) {
    super.init(processingEnvironment);
    filer = processingEnvironment.getFiler();
  }

  // 引入了注解处理器后，开始编译时，注解处理器会在编译代码前被执行
  // 到时候，该 process 函数就会被调用
  // 参数 set 是被捕获到的带注解的元素的集合
  // 参数 roundEnvironment 回合
  @Override
  public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
    // 遍历捕获到的每个带注解的元素 Activity
    for (Element element : roundEnvironment.getRootElements()) {
      String packageStr = element.getEnclosingElement().toString();
      String classStr = element.getSimpleName().toString();

      // 不直接操作 filer，而是借助 javapoet 来写
      // 生成的文件名为"原文件名 + Binding"
      com.squareup.javapoet.ClassName className =
              com.squareup.javapoet.ClassName.get(packageStr, classStr + "Binding");

      // 拿到一个构造函数的 builder
      com.squareup.javapoet.MethodSpec.Builder constructorBuilder =
              com.squareup.javapoet.MethodSpec.constructorBuilder()
          // public 修饰符
          .addModifiers(Modifier.PUBLIC)
          // 添加名为 activity 的参数
          .addParameter(com.squareup.javapoet.ClassName.get(packageStr, classStr), "activity");

      boolean hasBinding = false;

      // 遍历带注解的元素内的元素，Activity 里的字段，方法等
      // @BindView(R.id.textView) TextView textView; 就会在这里被找到
      for (Element enclosedElement : element.getEnclosedElements()) {
        if (enclosedElement.getKind() == ElementKind.FIELD) {
          BindView bindView = enclosedElement.getAnnotation(BindView.class);
          // 如果获取到 BindView 注解，则为该文件增加一行代码
          if (bindView != null) {
            hasBinding = true;
            // 通过构造函数的 builder，在构造函数中加代码
            constructorBuilder.addStatement("activity.$N = activity.findViewById($L)",
                enclosedElement.getSimpleName(), bindView.value());
            // 加一行 "activity.textView = activity.findViewById(2131165354);" 这样的代码
            // $N 就是 enclosedElement.getSimpleName()
            // $L 就是 bindView.value()
          }
        }
      }

      // 构建一个类
      com.squareup.javapoet.TypeSpec builtClass =com.squareup.javapoet. TypeSpec.classBuilder(className)
          .addModifiers(Modifier.PUBLIC) // public 类型的构造函数
          .addMethod(constructorBuilder.build()) // 编写好的构造函数，添加进去
          .build();

      if (hasBinding) {
        try {
          // 在 packageStr 这个包路径下，生成一个由 builtClass 描述的类
          com.squareup.javapoet.JavaFile.builder(packageStr, builtClass)
              .build().writeTo(filer); // 把编写好的信息，写入到文件中
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
    return false;
  }

  // 本处理器，只处理 BindView 这个注解
  @Override
  public Set<String> getSupportedAnnotationTypes() {
    return Collections.singleton(BindView.class.getCanonicalName());
  }
}
```

这个函数，做了啥？
先遍历一个回合的根元素：roundEnvironment.getRootElements()；即 Activity；
再遍历 Activity 内的元素：element.getEnclosedElements()；
如果找到 BindView 注解修饰的字段，则为其生成一段代码并添加到辅助类的构造函数中；

辅助类的生成和插入代码，依靠 javapoet 类间接的操作 Filer 对象。

## 注解的定义

为了让 lib-processor 和 app 工程都能引用到注解，于是独立写到一个库中，即 lib-annotations 库。

```
@Retention(RetentionPolicy.SOURCE) // 保留的范围
@Target(ElementType.FIELD) // 修饰的目标
public @interface BindView {
  int value();
}
```

注解，其实也是一个接口，类型为 @interface；
@Retention 注解，可以指定保留的范围。可选项：
@Target 注解，可修饰的目标，可以是字段、可以是方法，或类；

### @interface

注解也是一种接口，也可以设置一些方法。
比如：
设置一个方法 id：
![屏幕快照 2021-06-09 下午11.50.43](/img/media/16232372586525/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-06-09%20%E4%B8%8B%E5%8D%8811.50.43.png)
使用注解时，就得用（不用就报错）
![屏幕快照 2021-06-09 下午11.50.24](/img/media/16232372586525/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-06-09%20%E4%B8%8B%E5%8D%8811.50.24.png)

还可以有默认值：
![屏幕快照 2021-06-09 下午11.52.42](/img/media/16232372586525/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-06-09%20%E4%B8%8B%E5%8D%8811.52.42.png)

还有个特殊的方法 value()，在免于写 value ==;
![屏幕快照 2021-06-09 下午11.54.56](/img/media/16232372586525/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-06-09%20%E4%B8%8B%E5%8D%8811.54.56.png)

![屏幕快照 2021-06-09 下午11.54.11](/img/media/16232372586525/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-06-09%20%E4%B8%8B%E5%8D%8811.54.11.png)
"value ==" 可以不写：
![屏幕快照 2021-06-09 下午11.54.23](/img/media/16232372586525/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-06-09%20%E4%B8%8B%E5%8D%8811.54.23.png)

### @Retention

```
// RetentionPolicy.class

public enum RetentionPolicy {
    SOURCE,  // 源码状态下保留，编译为字节码就不见了
    CLASS,   // 保留到字节码，但运行时不可见
    RUNTIME; // 运行时可见，即保留范围最大，事件最长

    private RetentionPolicy() {
    }
}
```

对应的英文解释：
![屏幕快照 2021-06-09 下午11.41.56](/img/media/16232372586525/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202021-06-09%20%E4%B8%8B%E5%8D%8811.41.56.png)

### @Target

表示可修饰的目标，可以是如下的内容：

```
public enum ElementType {
    TYPE,  // 类型
    FIELD, // 字段
    METHOD,// 方法
    PARAMETER,   // 参数
    CONSTRUCTOR, // 构造器
    LOCAL_VARIABLE, // 本地变量
    ANNOTATION_TYPE, // 注解类型
    PACKAGE, // 包
    TYPE_PARAMETER, // 类型的参数？
    TYPE_USE;       // 类型的用户？

    private ElementType() {
    }
}

```

做限制是为了减少报错。

## 小结

回顾几个关键的点：
- 如何引入注解处理器
    - annotationProcessor project(':lib-processor')
- 如何找出带 BindView 注解的字段
    - 从 element.getEnclosedElements() 中找
- 如何往辅助类里写代码
    - 借助 javapoet 类间接的操作 Filer 对象
- 运行时如何引用到编译时生成的辅助类
    - 反射！只要知道辅助类的类名。

这只是简单的 Demo，[真的 ButterKnife 参考 JsonChao 的讲解](https://juejin.cn/post/6844904072491827207)。

