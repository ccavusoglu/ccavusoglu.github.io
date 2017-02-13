---
title: LibGDX Dependency Injection using Dagger2
layout: single
excerpt: "Configuring Dependency Injection framework Dagger2 for using with LibGDX"
date: "2017-02-10"
category: posts
tags:
 - dagger2
 - di
 - libgdx
published: true
---

Inversion of Control is a generic term which says the control flow of a program is inverted by an external source. Dependency Injection is a form of IoC. Basically it helps us manage our dependencies and object lifecycles in an elegant way. However, when poorly used, it almost like using global singleton objects all over the place which always considered a bad design. For more info about IoC and DI Containers: 
<br/>
<br/>
<https://martinfowler.com/bliki/InversionOfControl.html>
<br/>
<https://martinfowler.com/articles/injection.html>
<br/>
<http://www.jamesshore.com/Blog/Dependency-Injection-Demystified.html>
<br/>

Dagger2 is the successor of Dagger and developed by Google. It can be used both in Java and Android projects. It creates dependency graph at compile-time thus has no performance penalty unlike run-time reflection based DI frameworks.

It's modern and easy to use. Also has no performance drawbacks which all together makes it a great tool for use along with LibGDX.

This post will not cover Dagger2 concepts and usages. For more info:
<br/>
<https://github.com/codepath/android_guides/wiki/Dependency-Injection-with-Dagger-2>

We'll start by modifying root <b>build.gradle</b> file.

Add following dependencies under <b>buildscript</b>:

```groovy
classpath "net.ltgt.gradle:gradle-apt-plugin:0.4"
```

Add dagger dependencies and apply apt plugin needed for Dagger's custom annotations:

```groovy
...

project(":core") {
    apply plugin: "java"
    apply plugin: "net.ltgt.apt"

    dependencies {
        compile "com.badlogicgames.gdx:gdx:$gdxVersion"

        compile "com.google.dagger:dagger:$daggerVersion"
        apt "com.google.dagger:dagger-compiler:$daggerVersion"}
}

...
```

Then we create our Module class and Component interface.

```java
@Module
public class MainModule {
    @Provides
    @Singleton
    MainController provideMainController() {
        return new MainController();
    }
}
```

And MainComponent:

```java
@Singleton
@Component(modules = MainModule.class)
public interface MainComponent {
    void inject(Dagger2Demo applicationListener);
}
```

Now, we need to instantiate our MainComponent, only once.

```java
component = DaggerMainComponent.builder().mainModule(new MainModule()).build();
```

After this point, we can either use property injection or construction injection.

```java
component.inject(this);
```

This line injects the properties annotated with <b>@Inject</b>.

That's all. Make sure to rebuild the project to reference Dagger component.

Whole source code:
<https://github.com/ccavusoglu/LibGDX-Dagger2-Demo>
