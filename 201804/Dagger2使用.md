# 什么是Dagger2
 Dagger2 是一款使用在Java和Android上的依赖注入的开源库。
# 配置Dagger2
在app目录下的builde.prop文件中增加以下配置

```
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.google.dagger:dagger:2.14.1'
    annotationProcessor 'com.google.dagger:dagger-compiler:2.14.1'
}

```

#Dagger2 基础
#### Inject标签
注入，该注解标示地方表示需要通过DI框架来注入实例。Inject有三种方式，分别是构造方法注入、属性变量注入、普通方法注入。申明了Inject之后，会从注入框架中通过Component去查找需要注入的类实例，然后注入进来。

#### Component
使用该标签作为桥梁，使Module和Scope联系在一起.
#### SubComponent 
和上面的类似，相当于子父类继承关系。不需要在被依赖的Component显示提供依赖，不需要使用更多的DaggerXXXXComponent对象来创建依赖，斤需要在被依赖Component中直接调用预先留好的方法即可
#### Scope标签
管理Dagger中的实例的生命周期。ApplicationScope对应Application，因为一个APP里面大部分情况下只有个Application，所以在使用ApplicationScpoe的时候还需要加上@Singleton标签。ActivityScope对应每个Activity，类似的还有ServiceScope等
#### Provide
该标签用在Module中的方法中，申明方法的是可以提供被注入的类实例，一般进行初始化操作。
#### Qualifier
如果类实例创建有多种相同的方式，需要通过标签tag来区分，并在注入的时候通过标签来区分。

#使用介绍
#### Module
使用@Module标注类，标识他为供应端。

## 1.1 处理实例需求端
在实例需求中，使用@Inject标注进行实例化操作

```
public class MainActivity extends Activity {
    @Inject
    BasePresenter presenter;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

## 基本使用流程
使用@Module增加一个SaladModule类，用于管理所有依赖

a) 使用@Provides，为每一个依赖写一个provideXxx方法

 // 按照格式写就好了，
    // 返回值（被依赖的类类型）
    // 方法名（provideXxx必须以provide开头，后面随意）
    @Provides
    public Pear providePear(){
        return new Pear();//返回这个类的对象
    }
 
使用@Component增加一个SaladComponent接口

a) 指明要在那些Module里寻找依赖

@Component(modules = {SaladModule.class})//
1
b) 为所有的依赖添加一个方法

   //注意：下面这三个方法，返回值必须是从上面指定的依赖库SaladModule.class中取得的对象
    //注意：而方法名不一致也行，但是方便阅读，建议一致，因为它主要是根据返回值类型来找依赖的

    Pear providePear();
 
c) 提供一个供目标类使用的注入方法

  //注意：下面的这个方法，表示要将以上的三个依赖注入到某个类中
    //这里我们把上面的三个依赖注入到Salad中
    //因为我们要做沙拉
    void inject(Salad salad);
 
目标类使用依赖注入

a) 使用@Inject声明属性变量，表示注入这个依赖 
b) 调用saladComponent的inject，注入依赖

 // DaggerSaladComponent编译时才会产生这个类，
        // 所以编译前这里报错不要着急(或者现在你先build一下)
        SaladComponent saladComponent = DaggerSaladComponent.create();
        saladComponent.inject(this);//将saladComponent所连接的SaladModule中管理的依赖注入本类中
1
2
3
4
或者调用下面方法，也可以

   DaggerSaladComponent.builder()
 .saladModule(new SaladModule())
 .build()
 .inject(thi



