# Android中使用AbstractProcessor在编译时生成代码

发现这边不错的文章，忍不住转了过来，转自：
http://blog.csdn.net/industriously/article/details/53932425

## 1.概述

在现阶段的Android开发中，注解越来越流行起来，比如ButterKnife，Retrofit，Dragger，EventBus等等都选择使用注解来配置。按照处理时期，注解又分为两种类型，一种是运行时注解，另一种是编译时注解，运行时注解由于性能问题被一些人所诟病。编译时注解的核心依赖APT(Annotation Processing Tools)实现，原理是在某些代码元素上（如类型、函数、字段等）添加注解，在编译时编译器会检查AbstractProcessor的子类，并且调用该类型的process函数，然后将添加了注解的所有元素都传递到process函数中，使得开发人员可以在编译器进行相应的处理，例如，根据注解生成新的Java类，这也就是EventBus，Retrofit，Dragger等开源库的基本原理。 
Java API已经提供了扫描源码并解析注解的框架，你可以继承AbstractProcessor类来提供实现自己的解析注解逻辑。下边我们将学习如何在Android Studio中通过编译时注解生成java文件。

## 2.创建名为processor的module

首先使用Android Studio创建一个Android的project。然后开始创建一个名为processor的java library。 
点击file->new->new module如图

![enter image description here](http://img.blog.csdn.net/20161229215228171)

我们需要创建一个非Android的library，注意一定要选择Java Library

![enter image description here](http://img.blog.csdn.net/20161229215408243)

![enter image description here](http://img.blog.csdn.net/20161229215521619)

## 3.兼容性配置

由于Android目前不是完全支持Java 8的语言特性，会导致编译出错。这里将项目的源和目标兼容性值保留为 Java 7。 
打开app模块下的build.gradle

![enter image description here](http://img.blog.csdn.net/20161230094702814)

在android标签下添加 compile options

```gradle
compileOptions {
   sourceCompatibility JavaVersion.VERSION_1_7
   targetCompatibility JavaVersion.VERSION_1_7
}
```

如图 

![enter image description here](http://img.blog.csdn.net/20161230095834772)

然后打开processor library的build.gradle 

![enter image description here](http://img.blog.csdn.net/20161230100024147)

添加

```gradle
sourceCompatibility = 1.7
targetCompatibility = 1.7
```

![enter image description here](http://img.blog.csdn.net/20161230100954700)

## 4.创建Annotation

在processor模块下创建一个注解类 
![enter image description here](http://img.blog.csdn.net/20161230101123232)
命名为CustomAnnotation
![enter image description here](http://img.blog.csdn.net/20161230101236374)
具体内容如下
![enter image description here](http://img.blog.csdn.net/20161230110048597)
## 5.创建注解处理器

Processor继承自AbstractProcessor类，@SupportedAnnotationTypes中填写待处理的注解全称，@SupportedSourceVersion表示处理的JAVA版本。

```java
@SupportedAnnotationTypes(“<待处理注解类路径>”)
@SupportedSourceVersion(SourceVersion.RELEASE_7)
```
![enter image description here](http://img.blog.csdn.net/20161230112529280)
在注解类上右键选择copy reference即可快速的获得类的全称 
![enter image description here](http://img.blog.csdn.net/20161230141551648)
实现AbstractProcessor的方法 
![enter image description here](http://img.blog.csdn.net/20161230144820904)
编译时候将会执行process方法
![enter image description here](http://img.blog.csdn.net/20161230144852483)

```java
 @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        StringBuilder builder = new StringBuilder()
                .append("package com.yuntao.annotationprocessor.generated;\n\n")
                .append("public class GeneratedClass {\n\n") // open class
                .append("\tpublic String getMessage() {\n") // open method
                .append("\t\treturn \"");


        // for each javax.lang.model.element.Element annotated with the CustomAnnotation
        for (Element element : roundEnv.getElementsAnnotatedWith(CustomAnnotation.class)) {
            String objectType = element.getSimpleName().toString();


            // this is appending to the return statement
            builder.append(objectType).append(" says hello!\\n");
        }


        builder.append("\";\n") // end return
                .append("\t}\n") // close method
                .append("}\n"); // close class


        try { // write the file
            JavaFileObject source = processingEnv.getFiler().createSourceFile("com.yuntao.annotationprocessor.generated.GeneratedClass");


            Writer writer = source.openWriter();
            writer.write(builder.toString());
            writer.flush();
            writer.close();
        } catch (IOException e) {
            // Note: calling e.printStackTrace() will print IO errors
            // that occur from the file already existing after its first run, this is normal
        }


        return true;
    }
```

此处调用process方法生成了一个Java类文件，该类包含一个getMessage方法，该方法会返回一个字符串，其中字符串包含被@CustomAnnotation修饰的类的名称。这里主要在process方法中获取注解修饰类的名称。先看下生成的代码如下。 

![enter image description here](http://img.blog.csdn.net/20161230150411553)

注意：由于这个文件是在build过程中创建的，所以只有build成功之后才可以查看到它。对应该类生成在以下目录中：

```java
app/build/generated/source/apt/debug/<package>/GeneratedClass.java
```
这里便于演示我们只是简单的生成了一个Java源文件，当然如果要生成更复杂的文件，可以利用第三方工具，例如javapoet

## 6.创建resource

创建好注解处理器后，我们需要告诉编译器在编译的时候使用哪个注解处理器，这里就需要创建javax.annotation.processing.Processor文件

在processor模块下，main目录中创建一个resources文件夹，然后下边在创建META-INF/services，最后里边一个javax.annotation.processing.Processor文件，如下：

![enter image description here](http://img.blog.csdn.net/20161230151120118) 

在此文件中写入注解处理器的类全称

```gradle
com.yuntao.annotationprocessor.processor.CustomAnnotationProcessor
```

## 7.添加android-apt

在project下的build.gradle中添加apt插件 

![enter image description here](http://img.blog.csdn.net/20161230151149664)

添加依赖

```gradle
classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
```

![enter image description here](http://img.blog.csdn.net/20161230151233899)

然后在app中的build.gradle添加

```gradle
apply plugin: 'com.neenbedankt.android-apt'
```

![enter image description here](http://img.blog.csdn.net/20161230151739265)

## 8.设置build的依赖

这一节主要讲的是把processor模块中的注解，注解处理器编译生成一个jar，然后把这个jar包复制到app模块下。然后让app依赖引用这个jar。 
首先配置app的build.gradle依赖项，添加jar依赖，看最后一行。

```gradle
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.0.1'
    testCompile 'junit:junit:4.12'
    compile files('libs/processor.jar')
}
```
然后我们编写一个gradle task，把生成的jar文件复制到app/libs目录中。

```gradle
task processorTask(type: Exec) {
    commandLine 'cp', '../processor/build/libs/processor.jar', 'libs/'
}
```

最后我们创建task的依赖顺序，app:preBuild依赖我们写的processorTask, processorTask依赖 :processor:build: 
意思就是processor build完成后把jar复制到app，然后在执行app:preBuild。

```gradle
processorTask.dependsOn(':processor:build')
preBuild.dependsOn(processorTask)
```

最后配置完的build.gradle文件如下。 

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483081833_8464.png)

执行下编译命令/gradlew :app:clean :app:build，看task执行顺序 

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483081927_6141.png)

同时在app/libs目录下可以查看到生成的jar 

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483081968_6383.png)

## 9.使用注解

我们在MainActivity的类与onCreate方法上使用@CustomAnnotation 

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483082054_1724.png)

现在加上注解之后还没有生成我们需要的java文件，需要rebuild下才会生成，可以在Android Studio中选择Build>Rebuild或者在终端执行

```
/gradlew :app:clean :app:build
```

然后我们可以在下述位置查看到生成的Java文件

app/build/generated/source/apt/debug/package/GeneratedClass.java
![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483082156_1285.png)
![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483082213_2699.png)
## 10.验证能否使用

在MainActivity的onCreate方法弹窗，显示下生成的GeneratedClass的getMessage方法返回的信息。代码如下

```java
private void showAnnotationMessage() {
        GeneratedClass generatedClass = new GeneratedClass();
        String message = generatedClass.getMessage();
        // android.support.v7.app.AlertDialog
        new AlertDialog.Builder(this)
                .setPositiveButton("Ok", null)
                .setTitle("Annotation Processor Messages")
                .setMessage(message)
                .show();
    }
```

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483082816_6002.png)

运行后结果
![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483082876_7928.png)

## 11.AnnotationProcessor中调试代码

一般在写代码的时候调试是不可避免的，最后一节我们讲下如何调试。 
首先代码中对process()方法设置代码断点。

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483082951_3321.png)

设置gradle daemon端口和JVM参数。把下面两行加入到你的gradle.properties文件。
![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483082904_4880.png)

```
org.gradle.daemon=true
org.gradle.jvmargs=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```
在命令行中运行gradle daemon来启动守护线程。

```
gradle --daemon
```

在Android Studio建立Remote Debugger

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483083146_8434.png)

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483083071_5291.png)

Remote Debugger 配置，我们在这里使用默认设置。IP:localhost，端口:5005。 

![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483083354_3049.png)

然后点击下图按钮运行，它就会连接到daemon线程中了。 
![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483083421_1807.png)

最后我们用gradle命令来运行构建。

```
gradle clean assembleDebug
```
既然我们已经启动了守护线程，Remote Debugger将触发断点并挂起构建运行。 
![enter image description here](http://img.my.csdn.net/uploads/201612/30/1483083422_4695.png)

最后总结下本编文章，主要讲了AbstractProcessor的使用，在Android Studio构建过程创建Java文件，同时使用他的例子，最后补充了一下如何调试Processor的方法。
