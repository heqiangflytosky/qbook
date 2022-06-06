本文作为 Dagger2 的进阶篇来介绍一些注解的使用。

## `@Inject`

### 注解构造函数

前面我们学习到了将 `@Inject` 注解在变量上，那么就生成了一个变量的注入器，其实  `@Inject` 还可以注解在构造方法和方法上面。    
先来看一下注解在构造方法方法上。    
值得注意的是同一个类中若有多个构造函数，则 `@Inject` 仅能注解其中一个。    

```
class TestInject {
    @Inject
    public TestInject() {
        
    }
}
```

那么dagger编译器就会生成下面的工厂类：

```
public final class TestInject_Factory implements Factory<TestInject> {
  @Override
  public TestInject get() {
    return newInstance();
  }

  public static TestInject_Factory create() {
    return InstanceHolder.INSTANCE;
  }

  public static TestInject newInstance() {
    return new TestInject();
  }

  private static final class InstanceHolder {
    private static final TestInject_Factory INSTANCE = new TestInject_Factory();
  }
}
```

这个工厂类的生产对象就是 TestInject 对象，提供了两种生产方法：非静态的 `get()` 和静态的 `newInstance()`。而此工厂又是一个单例模式。

再来看下 `Factory` 这个接口。

```
public interface Factory<T> extends Provider<T> {
}
public interface Provider<T> {
    T get();
}
```

`Provider<T>` 的是专门为注入器设计的接口，通过 get 方法返回一个可被注入的对象实例，对象类型为T；而 `Factory<T>` 则是一个未被范围标识的 `Provider` 。

### 注入带参构造函数

我们再来看一个带参数的构造函数。

```
class TestInject2 {
    private String mStr;

    @Inject
    public TestInject2(String str) {
        mStr = str;
    }

    public String getStr() {
        return mStr;
    }
}
```

```
public final class TestInject2_Factory implements Factory<TestInject2> {
  private final Provider<String> strProvider;

  public TestInject2_Factory(Provider<String> strProvider) {
    this.strProvider = strProvider;
  }

  @Override
  public TestInject2 get() {
    return newInstance(strProvider.get());
  }

  public static TestInject2_Factory create(Provider<String> strProvider) {
    return new TestInject2_Factory(strProvider);
  }

  public static TestInject2 newInstance(String str) {
    return new TestInject2(str);
  }
}
```

因为 TestInject2 需要依赖一个 String 对象，所以 TestInject2_Factory 不再是单例的，并且有了一个 `Provider<String>` 作为成员变量提供构造 TestInject2 时所需的参数。    

定义一个对 TestInject2 有依赖的类： 

```
public class OtherTest {
    @Inject
    TestInject2 testInject2;
    @Inject
    TestInject otherTest;

    public TestInject2 getTestInject2() {
        return testInject2;
    }
}
```

为它定义一个 Component，TestInject2 的参数是如何提供的？就通过 `@BindsInstance` 来实现，这个后面再讲。

```
@Component
public interface OtherTestComponent {
    void inject(OtherTest otherTest);

    @Component.Builder
    interface Builder {
        @BindsInstance
        Builder injectStr(String str);

        OtherTestComponent build();
    }
    // 通过在 Component 中添加方法完成依赖对象的注入
    TestInject4 createTestInject4();
}
```

上面是通过变量来完成依赖注入，我们也顺便添加了一个通过在 Component 中添加方法来完成注入单例 `TestInject4` 对象的例子。

```
@MyScope
class TestInject4 {
    @Inject
    public TestInject4() {
        Log.e("Test","TestInject4()"+this);
    }

    public void test(){
        Log.e("Test","TestInject4 test()");
    }
}
```

查看编译代码：    
String 参数通过 Builder 传递给 DaggerOtherTestComponent，然后在构造 TestInject2 时使用。     

```
@DaggerGenerated
@SuppressWarnings({
    "unchecked",
    "rawtypes"
})
public final class DaggerOtherTestComponent implements OtherTestComponent {
  private final String injectStr;

  private final DaggerOtherTestComponent otherTestComponent = this;
  // 由于OtherTestComponent提供了返回 TestInject4 的方法，
  // 而且TestInject4是单例，所以提供了 testInject4Provider
  private Provider<TestInject4> testInject4Provider;

  private DaggerOtherTestComponent(String injectStrParam) {
    this.injectStr = injectStrParam;
    initialize(injectStrParam);

  }

  public static OtherTestComponent.Builder builder() {
    return new Builder();
  }

  private TestInject2 testInject2() {
    return new TestInject2(injectStr);
  }

  @SuppressWarnings("unchecked")
  private void initialize(final String injectStrParam) {
    this.testInject4Provider = DoubleCheck.provider(TestInject4_Factory.create());
  }

  @Override
  public void inject(OtherTest otherTest) {
    injectOtherTest(otherTest);
  }

  @Override
  public TestInject4 createTestInject4() {
    return testInject4Provider.get();
  }

  private OtherTest injectOtherTest(OtherTest instance) {
    OtherTest_MembersInjector.injectTestInject2(instance, testInject2());
    OtherTest_MembersInjector.injectOtherTest(instance, new TestInject());
    return instance;
  }

  private static final class Builder implements OtherTestComponent.Builder {
    private String injectStr;

    @Override
    public Builder injectStr(String str) {
      this.injectStr = Preconditions.checkNotNull(str);
      return this;
    }

    @Override
    public OtherTestComponent build() {
      Preconditions.checkBuilderRequirement(injectStr, String.class);
      return new DaggerOtherTestComponent(injectStr);
    }
  }
}
```

我们分别来看一下 TestInject、TestInject2 和单例 TestInject4 注入方式的异同。    
在 injectOtherTest() 方法中，为 OtherTest 对象的两个变量绑定了 TestInject、TestInject2 对象，在`createTestInject4()` 方法中，通过 testInject4Provider 获取单例的 TestInject4 对象。    


代码验证：

```
        OtherTest otherTest = new OtherTest();
        OtherTestComponent otherTestComponent = DaggerOtherTestComponent.builder().injectStr("TestStr").build();
        otherTestComponent.inject(otherTest);
        Log.e("Test",otherTest.getTestInject2().getStr());
        TestInject4 testInject = otherTestComponent.createTestInject4();
        testInject.test();
```

### 通过 Module 注入

上面我们用的方法是Inject构造函数的方法实现注入，用Module的方法也类似，只是要提供一个带参数的 Provides 方法而已。

```
@Module()
public class TestInject2Module {
    @Provides
    public TestInject2 provideTestInject2(String str) {
        return new TestInject2(str);
    }

}
```

把 TestInject2 构造方法的 `@Inject` 注解去掉。

```
class TestInject2 {
    private String mStr;

    //@Inject
    public TestInject2(String str) {
        mStr = str;
    }
...
}
```

下面介绍另外一种提供注入参数的依赖的方法。   
上面的参数注入方法是通过调用一个 `injectStr("TestStr")` 方法来注入参数，现在我们用 Module 来提供这个注入的参数。   
创建一个 Module 类，来提供String参数。   

```
@Module()
public class TestInject2Module {
    @Provides
    public String privideStr(){
        return "TTTTTest";
    }
}
```

OtherTestComponent 中添加 TestInject2Module，去掉 injectStr 方法。   

```
@Component(modules = {TestInject2Module.class})
@MyScope
public interface OtherTestComponent {
    void inject(OtherTest otherTest);

    @Component.Builder
    interface Builder {
//        @BindsInstance
//        Builder injectStr(String str);

        OtherTestComponent build();
    }
}
```



### 注解方法

在来看一下注解在方法上的情况。和前面将的注解到变量上类似，也生成了一个 XX_MembersInjector 类。   
它的使用场景是什么呢？比如我们不想破坏变量的封装性等原因，需要为变量添加 private 访问控制符，前面我们讲过，注入的变量不能为 private的，否则注入不成功。这个问题可以通过 `Inject` 方法来实现。    
我们可以先自定义一个方法，方法名任意，形参为变量对应类型，然后用@Inject标记该方法。例如：   

```
public class Car {
    private CustomEngine engine;

    @Inject
    public void setEngine(CustomEngine engine){
        this. engine = engine;
    }
```

## Lazy

有时我们想注入的依赖在使用时再完成初始化，加快加载速度，就可以使用注入 `Lazy<T>`。只有在调用 Lazy 的 get() 方法时才会初始化依赖实例注入依赖。    

```
class Car {
    @Inject
    Lazy <PetrolEngine> engine;

    public PetrolEngine getEngine() {
        return engine.get();
    }
}
```

## Provider

有时候不仅仅是注入单个实例，我们需要多个实例，这时可以使用注入 `Provider<T>`，每次调用它的 get() 方法都会调用到 `@Inject` 构造函数创建新实例或者 `Module` 的 provide 方法返回实例。   

```
class Car {
    @Inject
    Provider<PetrolEngine> provider;
    public PetrolEngine getEngine2() {
        return provider.get();
    }
}
```

## Qualifier 限定符

在前面的例子中，我们可以提供多种能源类型的发动机，比如：

```
interface Engine {
    public String name();
}
public class PetrolEngine implements Engine{
    //@Inject
    public PetrolEngine() {
        Log.e("Test","PetrolEngine constrctor");
    }

    @Override
    public String name() {
        return "PetrolEngine";
    }
}
public class ElectricEngine implements Engine{
    //@Inject
    public ElectricEngine() {
        Log.e("Test","ElectricEngine constrctor");
    }

    @Override
    public String name() {
        return "ElectricEngine";
    }
}

```

在 EngineModule 中，我们也提供了两个不同的方法来生成不同类型的 Energy。那么Dagger怎么知道要调用哪个方法来生成该类型的 Engine 呢？    
而 `@Qualifier` 注解就是用来解决这个问题，使用注解来确定使用哪种 provide 方法。    
我们可以使用 `@Named` 注解，它是被`Qualifier`修饰的注解。你也可以用自定义的其他 `Qualifier` 注解。    

在 provide 方法上加上 `@Named` 注解，用来区分不同类型的 Energy：

```
@Module
class EngineModule {
    @Provides
    @Named("petrol")
    public Engine provideEngine() {
        return new PetrolEngine();
    }

    @Provides
    @Named("electirc")
    public Engine provideEngine2() {
        return new ElectricEngine();
    }
}
```

需要在 Inject 注入的地方加上` @Named` 注解，表明需要注入的是哪一种 Energy：

```
public class Car {
    @Inject
    @Named("petrol")
    Engine engine;

    @Inject
    @Named("electirc")
    Engine engine2;

    public Engine getEngine() {
        return engine;
    }

    public Engine getEngine2() {
        return engine2;
    }
}
```

## Scope 作用域

Scope 是用来确定注入的实例的生命周期的，如果没有使用 Scope 注解，Component 每次调用 Module 中的 provide 方法或 Inject 构造函数生成的工厂时都会创建一个新的实例，而使用 Scope 后可以复用之前的依赖实例。    
下面先介绍 Scope 的基本概念与原理，再分析 Singleton、Reusable 等作用域。    

### Scope

`@Scope` 是元注解，是用来标注自定义注解的：

```
@Retention(RUNTIME)
@Scope
public @interface MyScope {
}
```

MyScope 就是一个由 Scope 定义的注解，Scope 注解只能标注目标类、`@provide` 方法和 Component。Scope 注解要生效的话，需要同时标注在 Component 和提供依赖实例的 Module 或目标类上。Module 中 provide 方法中的 Scope 注解必须和与之绑定的 Component 的 Scope 注解一样，否则作用域不同会导致编译时会报错。例如，EngineModule 中 provide 方法的 Scope 是 MyScope 的话，CarComponent 的 Scope 必须是 是 MyScope 这样作用域才会生效，而且不能是 `@Singleton` 或其他 Scope 注解，不然编译时 Dagger 2 会报错。    
这里所说的单例，只是在 CarComponent 的生命周期范围内是单例的，如果我们应用中有两个 CarComponent 实例，那么被 MyScope 标注的类也是有两个对象的。    
那么怎么保证 MyScope 在全局范围内是单例的呢？我们只要保证在 Application 生命周期内 CarComponent 是单例的就可以了。    

```
@Module
class EngineModule {
    @Provides
    @MyScope
    @Named("petrol")
    public Engine provideEngine() {
        return new PetrolEngine();
    }
}
```

```
@Component(modules = EngineModule.class)
@MyScope
public interface CarComponent {
    void injectCar(Car car);
}

```

在 Car 中定义两个 GasEnergy 变量。

```
class Car {
    ......
    @Inject
    @Named("petrol")
    Lazy <Engine> energy;

    @Inject
    @Named("petrol")
    Lazy <Engine> energy2;
```

使用了 MyScope 后，DaggerCarComponent.initialize 方法的 provideEngineProvider 由 `EngineModule_ProvideEngineFactory.create(engineModuleParam)` 变成了 `DoubleCheck.provider(EngineModule_ProvideEngineFactory.create(engineModuleParam))`。

```
public final class DaggerMainComponent implements MainComponent {
  ......

  @SuppressWarnings("unchecked")
  private void initialize(final EngineModule engineModuleParam) {
    this.provideEngineProvider = DoubleCheck.provider(EngineModule_ProvideEngineFactory.create(engineModuleParam));
  }

  ......
```

而 DoubleCheck 包装的意义在于持有了 PetrolEngine 的实例，而且只会生成一次实例，也就是说：没有用 MyScope 作用域之前，DaggerCarComponent 每次注入依赖都会新建一个 PetrolEngine 实例，而用 MyScope 作用之后，每次注入依赖都只会返回第一次生成的实例。DaggerCarComponent 持有 provideEngineProvider 引用，provideEngineProvider 又持有 instance（即 PetrolEngine 实例）的引用，所以生成 PetrolEngine 对象实例的生命周期就和 Component 一致了，作用域就生效了。

### Singleton

Dagger2 自带的注解其实和我们自定义的 MyScope 是一样的，把上面例子中 `@MyScope` 换成 `@Singleton`，效果是一样的。

### Reusable

上文中的自定义的 `@MyScope` 和 `@Singleton` 都可以使得绑定的 Component 缓存依赖的实例，但是与之绑定 Component 必须有相同的 Scope 标记。假如我只想单纯缓存依赖的实例，可以复用之前的实例，不想关心与之绑定是什么 Component，应该怎么办呢？    
这时就可以使用 `@Reusable` 注解，Reusable 不关心绑定的 Component，Reusable 只需要标记目标类或 provide 方法，不用标记 Component。    
下面先看看使用 Reusable 作用域后，生成的 DaggerCarComponent 的变化：    

```
  @SuppressWarnings("unchecked")
  private void initialize(final EngineModule engineModuleParam) {
    this.provideEngineProvider = SingleCheck.provider(EngineModule_ProvideEngineFactory.create(engineModuleParam));
  }
```

从上面代码可以看出使用 `@Reusable` 后，但是这里是用SingleCheck而不是DoubleCheck。    
我们来看一下 SingleCheck 的实现，发现它不是线程安全的，在多线程情况下可能会生成多个实例。因此说它提供的复用，不是严格的同步单例，只是一个“尽量复用”逻辑。    

```
public final class SingleCheck<T> implements Provider<T> {
    private static final Object UNINITIALIZED = new Object();
    private volatile Provider<T> provider;
    private volatile Object instance;

    private SingleCheck(Provider<T> provider) {
        this.instance = UNINITIALIZED;

        assert provider != null;

        this.provider = provider;
    }

    public T get() {
        Object local = this.instance;
        if (local == UNINITIALIZED) {
            Provider<T> providerReference = this.provider;
            if (providerReference == null) {
                local = this.instance;
            } else {
                local = providerReference.get();
                this.instance = local;
                this.provider = null;
            }
        }

        return local;
    }
    ......
}
```

## BindsInstance

前面我们讲了，可以通过 Module 注解或 Inject 目标类构造函数的方式提供依赖实例，除了这两种方式，Component 还可以在创建 Component 的时候绑定依赖实例，用以注入。这就是@BindsInstance注解的作用，只能在 Component.Builder 中使用。

```
class Car2 {
    @Inject
    String mStr;

    @Inject
    Engine engine;

    public Engine getEngine() {
        return engine;
    }
}
```

```
@Component()
public interface CarComponent2 {
    void inject(Car2 car);
    String getStr();
    @Component.Builder
    interface Builder {
        @BindsInstance
        Builder injectEngine(PetrolEngine energy);
        @BindsInstance
        Builder injectStr(String str);
        CarComponent2 build();
    }
}
```

```
        Car2  car2 = new Car2();
        CarComponent2 component2 = DaggerCarComponent2.builder().injectEngine(new PetrolEngine()).injectStr("Test").build();
        Log.e("Test",component2.getStr());
        component2.inject(car2);
        car2.getEngine().toString();
        Log.e("Test",car2.mStr);
```


## Binds

`@Binds` 注解和 `@Provides` 注解的功能类似，它两者的不同之处在于，`@Provides` 注解可以提供第三方类和接口的注入，`@Binds` 注解只能提供接口的注入，且只能注解抽象方法。    
现在我们分别用两种方式实现 Car 中的两种 Engine 的注入：    
Car 和 CarComponent 都和前面是一样的。    

```
public class Car {
    @Inject
    @Named("petrol")
    Engine engine;

    @Inject
    @Named("electirc")
    Engine engine2;

    public Engine getEngine() {
        return engine;
    }

    public Engine getEngine2() {
        return engine2;
    }
}
```

```
@Component(modules = EngineModule.class)
public interface CarComponent {
    void injectCar(Car car);
}
```

主要区别在 EngineModule ，EngineModule 为 abstract，`@Binds` 注解的方法为 abstract，`@Provides` 注解的方法为 static。

```
@Module
abstract class EngineModule {
    @Provides
    @Named("petrol")
    public static Engine provideEngine() {
        return new PetrolEngine();
    }

    @Binds
    @Named("electirc")
    public abstract  Engine provideEngine2(ElectricEngine engine);
}
```

ElectricEngine 的构造方法要使用 `@Inject` 注解。

```
public class ElectricEngine implements Engine{
    @Inject
    public ElectricEngine() {
        Log.e("Test","ElectricEngine constrctor");
    }

    @Override
    public String name() {
        return "ElectricEngine";
    }
}
```

## IntoMap 和 IntoSet

可以使用 `@IntoSet` 和 `@IntoMap` 以注入的形式初始化 Set 或者 Map。

```
@Module
abstract class TestMapModule {

    @Binds
    @IntoMap
    @StringKey("electric")
    public abstract Engine provideElectricEngine(ElectricEngine engine);

    @Binds
    @IntoMap
    @StringKey("petrol")
    public abstract Engine providePetrolEngine(PetrolEngine engine);
}
```

`provideXXEngine` 方法返回了 ElectricEngine 和 PetrolEngine，都放到 map 中，它的 key 是 String 类型。另外key还可以是 `IntKey`、`ClassKey` 等。

```
public class OtherTest {

    ....
    @Inject
    Map<String,Engine> mEngineMap;
    ....

}
```

在 OtherTest 类中初始化 mEngineMap。 

## MultiBinds

Dagger通过多元绑定可以将几个对象放入一个集合中，即使它们是由不同的Module提供。集合的组装由Dagger自己完成，所以往代码中注入时，不需要直接依赖于每个对象。    
结合 `@IntoSet` 和 `@IntoMap` 等使用。如果是使用`@IntoMap` 则还要使用 `@StringKey` 或者 `@ClassKey` 来给提供的实例指定Map的键。    

## BindsOptionalOf


## Component的复用

现在我们对 Car 来做一个扩展，增加辅助功能，包含雷达，倒车影像，音响系统等。    
先来提供雷达功能。    

```
public class Radar {
    //@Inject
    public String name() {
        return "Radar";
    }
}
```

```
@Component(modules = AssistModule.class)
public interface AssistComponent {
    void injectRadar(Radar rader);
}
```

```
@Module()
public class AssistModule {
    @Provides
    public Radar privideRader(){
        return new Radar();
    }
}
```

Component 中添加 AssistModule。

```
@Component(modules = {EngineModule2.class,AssistModule.class})
public interface CarComponent2 {
    void injectCar(Car2 car);
}

```

在Car中定义 Radar 变量。

```
public class Car2 {
    @Inject
    CustomEngine engine;

    @Inject
    Radar radar;

    public CustomEngine getEngine() {
        return engine;
    }

    public Radar getRadar() {
        return radar;
    }
}
```

### Component.dependencies

如果一个 Component 依赖其他 Compoent 公开的依赖实例，用 Component 中的dependencies声明。

来定义一个 Car3，它依赖于 Car2 中 Engine 的实现。

```
public class Car3 {
    @Inject
    CustomEngine engine;

    public CustomEngine getEngine() {
        return engine;
    }
}

```

使用 dependencies 表示 CarComponent3 依赖CarComponent2的实现。

```
@Component(dependencies = CarComponent2.class)
public interface CarComponent3 {
    void injectCar(Car3 car);
}
```

那么在 CarComponent2 中我们需要将被依赖的内容显示地提供出来：    

```
@Component(modules = {EngineModule2.class,AssistModule.class})
public interface CarComponent2 {
    CustomEngine provideEngine();
}
```

这样，虽然在 CarComponent3 中没有 Module 来提供创建 CustomEngine，但是可以依赖 CarComponent2 中的 Module 提供 CustomEngine 的创建。

这里需要将 carComponent2 实例手动传入到 carComponent3。

```
        Car2 car2 = new Car2();
        CarComponent2 carComponent2 = DaggerCarComponent2.create();
        carComponent2.injectCar(car2);
        Log.e("Test",car2.getEngine().name());
        Log.e("Test",car2.getRadar().name());

        Car3 car3 = new Car3();
        DaggerCarComponent3.builder().carComponent2(carComponent2).build().injectCar(car3);
        Log.e("Test",car3.getEngine().name());
```

### Subcomponent & Module.subcomponents

如果一个 Component 继承（也可以叫扩展）某 Component 提供更多的依赖，可以用SubComponent 实现。

现在我们定义一个 Bus 类来介绍 Subcomponent 的使用。Bus 类不仅有自己的新的属性 Wheel，而且还有 Car 中的属性，这个时候可以使用 Subcomponent。

首先定义 Bus 类。

```
public class Bus {
    @Inject
    CustomEngine engine;

    @Inject
    Radar radar;

    @Inject
    Wheel wheel;
    
    @Inject
    String str; // 测试Bus对于Car中注入的String的使用
    
    // 为了自动注入bus对象
    @Inject
    public Bus() {

    }

    public CustomEngine getEngine() {
        return engine;
    }

    public Radar getRadar() {
        return radar;
    }

    public Wheel getWheel() {
        return wheel;
    }
}
```

```
// 使用 Subcomponent 来标记 BusComponent，表示它是某个 Component 的 Subcomponent
// Bus 类新增加了 Wheel 属性，所以这里要加上 WheelModule
@Subcomponent(modules = WheelModule.class)
public interface BusComponent {
    void injectBus(Bus bus);
    void injectWheel(Wheel wheel);
    // Subcomponent 必须显示声明 Subcomponent.Builder
    @Subcomponent.Builder
    interface Builder{
        BusComponent build();
    }
    // 添加一个方法，来获取Bus对象，要求构造方法必须 @Inject
    Bus getBus();
}
```

```
public class Wheel {
    public String name() {
        return "Wheel";
    }
}
```

```
@Module()
public class WheelModule {
    @Provides
    public Wheel privideWheel(){
        return new Wheel();
    }
}
```

怎么表明一个 SubComponent 是属于哪个 parent Component 的呢？只需要在 parent Component 依赖的 Module 中的 subcomponents 加上 SubComponent 的 class，然后就可以在 parent Component 中请求 SubComponent.Builder。

因为 CarComponent2 依赖了 EngineModule2.class 和 AssistModule.class，那么这两个 Module 中都要使用 subcomponents 来指定。

```
@Module(subcomponents = BusComponent.class)
public class EngineModule2 {
    @Provides
    public CustomEngine provideEngine() {
        return new CustomEngine();
    }
}
```

```
@Module(subcomponents = BusComponent.class)
public class AssistModule {
    @Provides
    public Radar privideRader(){
        return new Radar();
    }
}
```

CarComponent2 中针对上面的例子又添加了对String的注入。

```
@Component(modules = {EngineModule2.class,AssistModule.class})
public interface CarComponent2 {
    void injectCar(Car2 car);

    @Component.Builder
    interface Builder {
        @BindsInstance
        Builder inject(String str);

        CarComponent2 build();
    }
    // 请求 BusComponent
    BusComponent.Builder busComponent();
}
```

SubComponent 编译时不会生成 DaggerXXComponent，需要通过 parent Component 的获取 SubComponent.Builder 方法获取 SubComponent 实例。

```
        Car2 car2 = new Car2();
        CarComponent2 carComponent2 = DaggerCarComponent2.create();
        carComponent2.injectCar(car2);
        Log.e("Test",car2.getEngine().name());
        Log.e("Test",car2.getRadar().name());

        BusComponent busComponent = carComponent2.busComponent().build();
        Bus bus = busComponent.getBus();

        Log.e("Test",bus.getEngine().name());
        Log.e("Test",bus.getRadar().name());
        Log.e("Test",bus.getWheel().name());
        Log.e("Test",bus.str);
```

dependencies 和 SubComponent 的对比：
相同点：

 - 两者都能复用其他 Component 的依赖
 - 有依赖关系和继承关系的 Component 不能有相同的 Scope

区别：

 - 依赖关系中被依赖的 Component 必须显式地提供公开依赖实例的接口，而 SubComponent 默认继承 parent Component 的依赖。
 - 依赖关系会生成两个独立的 DaggerXXComponent 类，而 SubComponent 子组件是依赖于父组件才能进行工作的，它并不会被独立的编译成注入代码，所以不会生成独立的 DaggerXXComponent 类，而是通过内部类的方式来实现子组件接口。

### Component.Factory

## Module.includes

就是实现了Module的一种组合关系。    
比如我们又实现了汽车的 Media 和 Navigation 等辅助设备，而且为他们各自建立了 XXModule。    

```
public class Media {
    public String name() {
        return "Media";
    }
}

@Module()
public class MediaModule {
    @Provides
    public Media privideMedia(){
        return new Media();
    }
}

public class Navigation {
    public String name() {
        return "Navigation";
    }
}

@Module()
public class NavigationMoudle {
    @Provides
    public Navigation privideNavigation(){
        return new Navigation();
    }
}
```

然后在 Car 中建立对它们的依赖：

```
public class Car4 {
    @Inject
    CustomEngine engine;

    @Inject
    Radar radar;

    @Inject
    Wheel wheel;

    @Inject
    Media media;

    @Inject
    Navigation navigation;
}
```

Car4Component 依赖 WheelModule.class 和 AssistModule2.class。

```
@Component (modules = {WheelModule.class,AssistModule2.class})
public interface Car4Component {
    void injectCar(Car4 car);
    void injectEngine(CustomEngine engine);
    void injectWheel(Wheel wheel);
    void injectMedia(Media media);
    void injectNavigation(Navigation navigation);
}
```

```
// AssistModule2 包含了 MediaModule.class 和 NavigationMoudle.class

@Module(includes = {MediaModule.class, NavigationMoudle.class})
public class AssistModule2 {
    @Provides
    public Radar privideRader(){
        return new Radar();
    }
}
```

```
        Car4 car4 = new Car4();
        DaggerCar4Component.create().injectCar(car4);
        Log.e("Test",car4.engine.name());
        Log.e("Test",car4.radar.name());
        Log.e("Test",car4.wheel.name());
        Log.e("Test",car4.media.name());
        Log.e("Test",car4.navigation.name());
    }
```

Module.includes 的实现其实和 Component.modules 的实现是一样的，比如上面我们不用 Module.includes：

```
// 去掉 includes
@Module(/*includes = {MediaModule.class, NavigationMoudle.class}*/)
public class AssistModule2 {
    @Provides
    public Radar privideRader(){
        return new Radar();
    }
}
```

在 Car4Component 中包含各个 Module：

```
//这里新增了 MediaModule.class, NavigationMoudle.class，不再包含在 AssistModule2 中。
@Component (modules = {WheelModule.class,AssistModule2.class,MediaModule.class, NavigationMoudle.class})
public interface Car4Component {
    void injectCar(Car4 car);
    void injectEngine(CustomEngine engine);
    void injectWheel(Wheel wheel);
    void injectMedia(Media media);
    void injectNavigation(Navigation navigation);
}
```

两者实现的效果是一样的。    
