# Android APT

随着一些如ButterKnife，dagger等的开源注解框架的流行，APT的概念也越来越被熟知

## APT 介绍

APT的全称为：Android annotation process tool，它是一种处理注解的工具。它对源代码文件进行检测，找出其中的annotation，进行额外的处理。

Annotation处理器在处理Annotation时可以根据源文件中的Annotation生成额外的源文件和其它的文件（文件具体内容由Annotation处理器的编写者决定），APT还会编译生成源文件和原来的源文件，将它们一起生成class文件。简言之：APT可以把注解，在编译时生成代码。

## APT处理要素

* 注解处理器（AbstractProcess）
* 代码处理（JavaPoet）
* 处理器注册（AutoService）
* apt

## APT处理流程

* 定义注解
* 定义注解处理器
* 在处理器中完成处理，通常是生成Java代码
* 注册处理器
* 利用APT完成工作：
1. 扫描注解
2. 根据定义好的处理器处理注解
3. 编译处理生成好的Java文件

## 简单使用方法

1. 新建一个Android项目Test
2. 新建Java Module：“apt-lib”，在apt-lib中添加注解类：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
    public @interface AutoCreate {
}
```
3. 新建Java Module：“apt-process”。在apt-process的gradle文件中添加依赖
```
compile 'com.squareup:javapoet:1.8.0'
compile 'com.google.auto.service:auto-service:1.0-rc2'
compile project(':apt-lib')
```
在apt-process中添加文件：TestProcess

![](/assets/APT_TestProcess.jpeg)

我们计划生成的类为：

```
package com.yuelinghui.test;

import java.lang.String;
import java.lang.System;

public final class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello,JavaPoet!");
    }
}
```

4. 在项目的gradle中添加依赖

```
classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
```
然后在app的gradle中添加依赖：

```
compile project(':apt-lib')
apt project(':apt-process')
```
在app的gradle中还要加上新的plugin：
```
apply plugin: 'com.neenbedankt.android-apt'
```
我们可以在MainActivity上添加注解@AutoCreate，之后Rebuild这个项目

5. 项目Rebuild之后，我们就能在app-build-generated-source-apt-debug中找到我们生成的Java文件：HelloWorld

## 自定义ButterKnife

上面我们已经讲了APT的简单使用，下面我们可以自己定义一个新的类似ButterKnife的工具，修改上面的代码

1. 在apt-lib中我们新建两个注解：AutoCreateActivity和AutoCreateView

* AutoCreateActivity：
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface AutoCreateActivity {
}
```
* AutoCreateView：
```
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AutoCreateView {
    String value();
}
```
2. 修改apt-process中的TestProcess文件
```
@AutoService(Processor.class)
public class TestProcess extends AbstractProcessor {

private Elements mElementUtils;

@Override
public synchronized void init(ProcessingEnvironment processingEnv) {
    super.init(processingEnv);
    mElementUtils = processingEnv.getElementUtils();
}

@Override
public SourceVersion getSupportedSourceVersion() {
    return SourceVersion.RELEASE_7;
}

@Override
public Set<String> getSupportedAnnotationTypes() {
    return Collections.singleton(AutoCreateActivity.class.getCanonicalName());
}

@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {

    Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(AutoCreateActivity.class);
    if (elements != null) {
        for (Element element : elements) {
            TypeElement typeElement = (TypeElement) element;
            List<? extends Element> members = mElementUtils.getAllMembers(typeElement);
            MethodSpec.Builder methodSpecBuilder = MethodSpec.methodBuilder("bindView")
            .addModifiers(Modifier.PUBLIC,Modifier.STATIC)
            .returns(TypeName.VOID)
            .addParameter(ClassName.get(typeElement.asType()),"activity");

            for (Element item : members) {
                AutoCreateView autoCreateView = item.getAnnotation(AutoCreateView.class);
                if (autoCreateView == null) {
                    continue;
                }
                methodSpecBuilder.addStatement(String.format("activity.%s = (%s) activity.findViewById(%s)",item.getSimpleName()
                ,ClassName.get(item.asType()).toString(),autoCreateView.value()));
            }

            TypeSpec typeSpec = TypeSpec.classBuilder("AutoCreate" + element.getSimpleName())
            .superclass(TypeName.get(typeElement.asType()))
            .addModifiers(Modifier.PUBLIC,Modifier.FINAL)
            .addMethod(methodSpecBuilder.build())
            .build();

            JavaFile javaFile = JavaFile.builder(getPackageName(typeElement),typeSpec).build();

            try {
                javaFile.writeTo(processingEnv.getFiler());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
return true;
}

private String getPackageName(TypeElement typeElement) {
    return mElementUtils.getPackageOf(typeElement).getQualifiedName().toString();
}
}
```
3. 在MainAcitivity里修改：

```
@AutoCreateActivity
public class MainActivity extends AppCompatActivity
implements NavigationView.OnNavigationItemSelectedListener {

@AutoCreateView("R.id.fab")
FloatingActionButton fab;
@AutoCreateView("R.id.toolbar")
Toolbar toolbar;
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    AutoCreateMainActivity.bindView(this);
    setSupportActionBar(toolbar);
    fab.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View view) {
        }
    });
```

先在Activity上加上注解，然后在控件声明上加上注解并声明控件id，然后我们Rebuild代码，生成了新的Java文件（AutoCreateMainActivity），然后我们调用这个类的bindView方法。

这样我们的自定义ButterKnife就完成了。这只是一个小应用，只是为了说明APT的工作原理，建议还是使用ButterKnife。
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTM2ODYzMjk3OF19
-->