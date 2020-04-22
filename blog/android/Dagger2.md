Dagger2快速入门

#前言
 　关于Dagger2的介绍在这里就不多说了，如果对Dagger2的用途和要解决的问题不了解的，可以点[这里](http://www.jianshu.com/p/cd2c1c9f68d4)。这有三篇文章可以让你对Dagger2有一个比较深刻的理解。但是看了这三篇文章并不能让你知道如何使用Dagger2，网上也有一些Dagger2的例子，但是大部分都跟MVP，Retrofit，RxJava一起使用。对于很多没接触过这些框架和开源项目的人来说，无疑造成了一定的困难，这篇文章的就是要把这些开源框架什么的都摒弃掉，直接使用Dagger2来完成一个简单的程序。从而让想学习Dagger2的人有一个初步的认识。
#1.需求是什么?
　 我们要实现的程序很简单，就是把android自带的Toast类稍微封装一下，然后注入到BaseActivity当中，这样新建的Activity继承了BaseActivity就可以直接使用我们封装好的Toast了。
#2.实现过程
　1.首先我们在androi studio中新建一个android项目。新建完成以后，我们要使用Dagger2就要配置相关的参数。
  　 在app\build.gradle中添加如下依赖
```
   apt'com.google.dagger:dagger-compiler:2.0.2'//指定注解处理器
   compile'com.google.dagger:dagger:2.0.2'//dagger公用api
   provided'org.glassfish:javax.annotation:10.0-b28'//添加android缺失的部分javax注解
```
   并且在这个文件的开头声明我们要使用apt
```
   apply plugin:'com.neenbedankt.android-apt'
```
   在项目根目录下的build.gradle中添加如下代码
```
   classpath'com.neenbedankt.gradle.plugins:android-apt:1.8'
```
　2.完成配置以后，我们就可以开始写代码了。首先建立一个ToastUtil类，这个类是对Toast类进行简单的封装。
```
package com.zzy.daggertoast.utils;

import android.content.Context;
import android.widget.Toast;

/**
 * Created by adm on 2016/4/18.
 */
public class ToastUtil
{
    public Context context;

    public ToastUtil(Context context)
    {
        this.context=context;
    }

    public void showToast(String text)
    {
        Toast.makeText(context,text,Toast.LENGTH_SHORT).show();
    }
}

```
　以后使用ToastUtil的showToast方法就可以显示Toast了。
　有了ToastUtil以后我们就要使用它，所以我们现在新建一个类BaseActivity继承AppCompatActivity。在这个类中声明ToastUtil为成员变量。然后我们不采用New的方式初始化，而是采用Dagger2的方式，也就是使用注解@inject。

```
public abstract class BaseActivity extends AppCompatActivity
{
   @inject
   public ToastUtil toast;
   @Override
   protected void onCreate(Bundle savedInstanceState)
   {
     super.onCreate(savedInstanceState);
     setContextView(getLayoutId());
     toast.showToast("BaseActivity onCreate...");
     afterCreate();
   }
//让子类复写以下方法
public abstract int getLayoutId();
public abstract void afterCreate();
}
```
以上代码要是运行的话肯定会报空指针错误，因为我们虽然在这里使用了@inject注解，但是我们的对象并没有被注入进来。现在我们的BaseActivity依赖ToastUtils这个类，那么我们接下来就要为这个依赖提供初始化操作。
新建一个类叫做AppModule：
```
@Module
public class AppModule
{
    public Context context;

    public AppModule(Context context)
    {
        this.context=context;
    }

    @Provides
    ToastUtil provideToastUtil()
    {
        return new ToastUtil(context);
    }
}
```
　首先我们应该声明这个类是一个@Module,这个类中有一个provideToastUtil方法，没错，这个方法就是提供ToastUtil对象的。生成的这个对象会赋值到BaseActivity中，记得在方法上面添加@Provides。
　好了，现在我们也有了提供ToastUtil对象的方法了。那么我们的ToastUtil对象如何注入到BaseActivity中呢？肯定是需要一个注入器才行。于是我们新建了一个接口AppComponent：
```
@Component(modules = {AppModule.class})
public interface AppComponent
{
    void inject(BaseActivity activity);
}
```
我们要把这个接口声明为@Component,并且告诉它modules的值是AppModule.class，这样BaseActivity中的依赖就会到这个AppModule类中找提供依赖初始化的方法。当我们编译程序以后，我们一开始配置的apt会为我们生成AppComponent这个接口的子类，名字为DaggerAppComponent,这个类我们可以在android Studio 目录app\build\generated\source\apt\你的包名中找到。以下是DaggerAppComponent的代码：
```
@Generated("dagger.internal.codegen.ComponentProcessor")
public final class DaggerAppComponent implements AppComponent {
  private Provider<ToastUtil> provideToastUtilProvider;
  private MembersInjector<BaseActivity> baseActivityMembersInjector;

  private DaggerAppComponent(Builder builder) {  
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {  
    return new Builder();
  }

  private void initialize(final Builder builder) {  
    this.provideToastUtilProvider = AppModule_ProvideToastUtilFactory.create(builder.appModule);
    this.baseActivityMembersInjector = BaseActivity_MembersInjector.create((MembersInjector) MembersInjectors.noOp(), provideToastUtilProvider);
  }

  @Override
  public void inject(BaseActivity activity) {  
    baseActivityMembersInjector.injectMembers(activity);
  }

  public static final class Builder {
    private AppModule appModule;
  
    private Builder() {  
    }
  
    public AppComponent build() {  
      if (appModule == null) {
        throw new IllegalStateException("appModule must be set");
      }
      return new DaggerAppComponent(this);
    }
  
    public Builder appModule(AppModule appModule) {  
      if (appModule == null) {
        throw new NullPointerException("appModule");
      }
      this.appModule = appModule;
      return this;
    }
  }
}


```
这就是我们注入器的真正实现。有了这些以后，我们就要开始在BaseActivity中进行注入了。现在修改我们的BaseActivity类：
```
public abstract class BaseActivity extends AppCompatActivity
{
    @Inject
    public ToastUtil toast;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        DaggerAppComponent.builder().appModule(new AppModule(this)).build().inject(this);
        toast.showToast("BaseActivity onCreate....");
        setContentView(getLayoutId());
        afterCreate();
    }

    public abstract int getLayoutId();
    public abstract void afterCreate();
}
```
可以看到我只是添加了一行代码:
```
DaggerAppComponent.builder().appModule(new AppModule(this)).build().inject(this);
```
添加完这行代码以后，我们就完成了注入，具体是怎么注入进来的，可以对照apt帮我们生成的代码。
接着我们在用android studio帮我们自动生成的MainActivity继承BaseActivity，然后再实现，getLayoutId(),afterCreate()这两个方法：
```
public class MainActivity extends BaseActivity
{
  public int getLayoutId()
  {
    return R.layout.activity_main;
  }

  public void afterCreate()
  {
    toast.showToast("MainActivity afterCreate...");
  }
}
```
Ok,到目前为止我们就完成了所有的代码，运行一下就可以看到Toast弹出来啦。

#3.后续
　本项目并没有讲解Dagger2的介绍，原理等，只是抛弃所有的框架，开源项目来直接使用Dagger2,让大家有一个直观的认识，所以在看本文章时，最好是有一点初步的知识。由于本文写的仓促，肯定存在很多问题，希望可以抛砖引玉，让初学者对Dagger2有更深一步的认识。
   　[源码地址](https://github.com/121880399/Dagge2Sample)
　转载请标明出处。