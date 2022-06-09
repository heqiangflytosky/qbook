dagger2是用于软件解耦的依赖注入工具，由谷歌自dagger优化而来。dagger的开发者就是大名鼎鼎的square公司，旗下还有retrofit,okhttp,greendao等等知名的开发框架。
Dagger 让我们可以用注解的方式来实现依赖注入。Dagger2使用注解和apt工具来生成注入依赖的模板代码。

## 什么是依赖

依赖是程序中常见的现象，比如类 Car 中用到了 PetrolEnergy 类的实例 engine，那么我们就说 Car 对 PetrolEnergy 有一个依赖。

```
public interface Engine {
    public String name();
}

class PetrolEngine implements Engine{
    public PetrolEngine() {
        Log.e("Test","PetrolEngine constrctor");
    }

    @Override
    public String name() {
        return "ElectricEnergy";
    }
}

class Car {
    Engine engine = new PetrolEngine();
}
```

其实上面的这种写法有几个问题：

 - 类 Car 承担了多余的责任，负责 engine 对象的创建，这必然存在了严重的耦合性。一辆汽车使用哪种能源不是由汽车来决定，而是由汽车制造商来决定，这是汽车制造商的责任。
 - 可扩展性，假设我们想修改引擎为电动机，那么我们必然要修改 Car 这个类，明显不符合开放闭合原则。
 - 不利于单元测试。无法测试不同的 Engine 对 Car 的影响。

## 什么是依赖注入

依赖注入是这样的一种行为，在类 Car 中不主动创建 PetrolEnergy 的对象，而是通过外部传入 PetrolEnergy 对象形式来设置依赖。
依赖注入降低了依赖和被依赖对象之间的耦合，方便扩展和单元测试。
常用的依赖注入有如下三种方式。

### 构造器注入

将需要的依赖作为构造方法的参数传递完成依赖注入。

```
class Car {
  Engine mEngine;
  public Car(Engine engine) {
      mEngine = engine;
  }
}

```

### Setter方法注入

增加setter方法，参数为需要注入的依赖亦可完成依赖注入。

```
class Car {
  Engine mEngine;
      
  public void setEngine(Engine engine) {
      mEngine  = engine;
  }
}

```

### 接口注入

接口注入，就是为依赖注入创建一套接口，依赖作为参数传入，通过调用统一的接口完成对具体实现的依赖注入。

```
interface EngineConsumerInterface {
  public void setEngine(Engine engine);
}
  
class Car implements EngineConsumerInterface {
  Engine mEngine;
      
  public void setEngine(Engine engine) {
      mEngine  = engine;
  }
}

```

接口注入和setter方法注入类似，不同的是接口注入使用了统一的方法来完成注入，而setter方法注入的方法名称相对比较随意。

### 注解注入

Dagger 2 依赖注入框架就是使用 @Inject 完成注入。

```
public class Car {
    @Inject
    Engine engine;

    public Engine getEngine() {
        return engine;
    }
}
```

## Dagger 2

Dagger 2 是 Java 和 Android 下的一个完全静态、编译时生成代码的依赖注入框架，由 Google 维护，早期的版本 Dagger 是由 Square 创建的。

下面是 Dagger 2 的一些资源地址：

Github：https://github.com/google/dagger    
官方文档：https://google.github.io/dagger/    
API：http://google.github.io/dagger/api/latest/    

## Dagger2 的基本使用

下面先来看一个 Dagger2 基本使用的例子。

添加依赖：

```
dependencies {
    implementation 'com.google.dagger:dagger:2.41'
    annotationProcessor 'com.google.dagger:dagger-compiler:2.41'
}
```

### 用到的几个注解

在这个例子中，我们会用到下面几个注解：

 - `@Inject`：1.用来标记依赖使用方：标记在需要注入依赖的变量， Dagger2 会帮助我们初始化。2.用来标记依赖的提供方：通过标记构造函数让 Dagger2 使用（Dagger2 通过 @Inject 可以在需要这个类实例的时候来找到这个构造函数并把相关实例 new 出来），从而提供依赖。3.注解方法，后面介绍。
 - `@Moudle`：依赖提供方，负责提供依赖中所需要的对象。
 - `@Provides`：在 Module 中使用，会根据返回值类型在有此注解的方法中寻找应调用的方法，只能使用在方法上面。
 - `@Component`：依赖注入组件，负责将依赖注入到依赖需求方。

注意：我们既可以使用 `@Injec` 构造方法，也可以使用`@Moudle` 和 `@Provides` 来标记依赖的提供方，另外还可以通过 `@BindsInstance` 的方式。我们可以根据不同的使用场景来选择合适的注入方式。    
`@Injec` 构造方法适用于我们有依赖提供方的源代码的情况，而使用 `@Moudle` 和 `@Provides` 适用于第三方库，因为我们无法再需要使用的类上添加 `@Inject`。

### 标注需要注入的依赖

使用 `@Inject` Car 类中的 energy 变量，标注需要注入的依赖。

```
public class Car {
    @Inject
    Engine engine;

    public Engine getEngine() {
        return engine;
    }
}
```

在 `build/generated/source/apt/debug/` 目录下生成成员变量注入器 `Car_MembersInjector.java` 类。

```
@QualifierMetadata
@DaggerGenerated
@SuppressWarnings({
    "unchecked",
    "rawtypes"
})
public final class Car_MembersInjector implements MembersInjector<Car> {
  private final Provider<Engine> engineProvider;

  public Car_MembersInjector(Provider<Engine> engineProvider) {
    this.engineProvider = engineProvider;
  }

  public static MembersInjector<Car> create(Provider<Engine> engineProvider) {
    return new Car_MembersInjector(engineProvider);
  }

  @Override
  public void injectMembers(Car instance) {
    injectEngine(instance, engineProvider.get());
  }

  @InjectedFieldSignature("com.example.heqiang.testsomething.dagger.car.Car.engine")
  public static void injectEngine(Object instance, Object engine) {
    ((Car) instance).engine = (Engine) engine;
  }
}
```

实现了 `injectEngine` 方法供 DaggerCarComponent 调用。    
从 `injectEngine` 方法中可以看到注入依赖的代码是 `((Car) instance).engine = (Engine) engine`，所以 @Inject 标注的成员属性不能是 private 的，不然无法注入。    

### Module

如果我们使用了 `@Inject` 在 PetrolEnergy 的构造方法上标注了依赖的提供方：

```
class PetrolEngine implements Engine{
    @Inject
    public PetrolEngine() {
        Log.e("Test","PetrolEngine constrctor");
    }

    @Override
    public String name() {
        return "PetrolEngine";
    }
}
``` 

那么这里就可以不用提供 Module 类了。如果两种方法都使用了，那么就优先使用 Module 类。

添加一个 Module 类，用于提供依赖中所需要的对象，需要用到 `@Module` 和 `@Provider` 注解：

```
@Module
class EngineModule {
    @Provides
    public Engine provideEngine() {
        return new PetrolEngine();
    }
}
```

Module 类编译时生成了两个类：`EngineModule_ProvideEngineFactory` 和 `EngineModule_Proxy`。    
`EngineModule_ProvideEngineFactory` 是由注解 `@Provides` 生成，`EngineModule_Proxy`由注解 `@Module` 生成。

```
@ScopeMetadata
@QualifierMetadata
@DaggerGenerated
@SuppressWarnings({
    "unchecked",
    "rawtypes"
})
public final class EngineModule_ProvideEngineFactory implements Factory<Engine> {
  private final EngineModule module;

  public EngineModule_ProvideEngineFactory(EngineModule module) {
    this.module = module;
  }

  @Override
  public Engine get() {
    return provideEngine(module);
  }

  public static EngineModule_ProvideEngineFactory create(EngineModule module) {
    return new EngineModule_ProvideEngineFactory(module);
  }

  public static Engine provideEngine(EngineModule instance) {
    return Preconditions.checkNotNullFromProvides(instance.provideEngine());
  }
}
```

```
@DaggerGenerated
@SuppressWarnings({
    "unchecked",
    "rawtypes"
})
public final class EngineModule_Proxy {
  private EngineModule_Proxy() {
  }

  public static EngineModule newInstance() {
    return new EngineModule();
  }
}
```

上面完成两步，通过 Dagger 2 生成的代码代码可以知道，生成了 Car 的成员属性注入类和 PetrolEnergy 的工厂类，接下来需要的就是新建工厂实例并调用成员属性注入类完成 car 的实例注入。完成这个过程的桥梁就是dagger.Component。

### Component 桥梁

添加一个 Component，表示给哪些类注入哪些对象，比如 Car 类需要用到 PetrolEnergy 对象，那么就可以新建一个接口，在接口上添加注解 @Component，把 EngineModule 赋上去，在接口下新增一个注入方法，把需要使用该对象的类作为参数传入进来，当然还可以将其他对象提供出去：

```
// 注意这里，如果不用 EngineModule 类提供依赖的话，这里就可以不写 `modules = EngineModule.class`
@Component(modules = EngineModule.class)
public interface CarComponent {
    void injectCar(Car car);
}
```

生成 `DaggerCarComponent` 类。

```
@DaggerGenerated
@SuppressWarnings({
    "unchecked",
    "rawtypes"
})
public final class DaggerCarComponent implements CarComponent {
  private final EngineModule engineModule;

  private final DaggerCarComponent carComponent = this;

  private DaggerCarComponent(EngineModule engineModuleParam) {
    this.engineModule = engineModuleParam;

  }

  public static Builder builder() {
    return new Builder();
  }

  public static CarComponent create() {
    return new Builder().build();
  }

  @Override
  public void injectCar(Car car) {
    injectCar2(car);
  }

  private Car injectCar2(Car instance) {
    Car_MembersInjector.injectEngine(instance, EngineModule_ProvideEngineFactory.provideEngine(engineModule));
    return instance;
  }

  public static final class Builder {
    private EngineModule engineModule;

    private Builder() {
    }

    public Builder engineModule(EngineModule engineModule) {
      this.engineModule = Preconditions.checkNotNull(engineModule);
      return this;
    }

    public CarComponent build() {
      if (engineModule == null) {
        this.engineModule = new EngineModule();
      }
      return new DaggerCarComponent(engineModule);
    }
  }
}
```

如果我们没有提供 Module 中类方法，而是使用 PetrolEngine 中通过 `@Inject` 标注一个构造方法来作为依赖的提供方，那么 dagger 会在 `injectCar2()` 方法中调用 `Car_MembersInjector.injectEngine(instance, new PetrolEngine());` 来注入 PetrolEngine 实例。

然后调用 `injectCar` 方法就完成了注入。

```
        Car car = new Car();
        DaggerCarComponent.create().injectCar(car);
        Log.e("Test",car.getEngine().name());
```

另外，如果是使用了 `@Module` 来提供依赖，还可以使用下面的依赖注入方法，可以指定 Module 类：

```
        DaggerCarComponent.builder().engineModule(new EngineModule()).build().injectCar(car);
```

至此我们就完成了一个简单的依赖注入的程序。

### Another

现在介绍一下我们使用`@Inject`目标类构造方法的方式来实现依赖注入。    
注意，使用这种方式时依赖使用方和依赖提供方的类必须要保持一致，上面的在Car中声明接口 Engine 是编译不过的。   
那么这里我们先抛出个疑问，那如果在Car里面无法声明变量为 Engine 类型，那么岂不是解决不了解耦的问题么？这个问题等介绍 `@Binds` 修饰符时就会有答案了。

```
class CustomEngine{
    @Inject
    public CustomEngine() {
        Log.e("Test","CustomEngine constrctor");
    }

    public String name() {
        return "CustomEngine";
    }
}
```

```
public class Car2 {
    @Inject
    CustomEngine engine;

    public CustomEngine getEngine() {
        return engine;
    }
}
```

```
@Component()
public interface CarComponent2 {
    void injectCar(Car2 car);
}

```

```
        Car2 car = new Car2();
        DaggerCarComponent2.create().injectCar(car);
        Log.e("Test",car.getEngine().name());
```

## 注入流程

在上面代码中，进行依赖注入是通过 `DaggerMainComponent.create().injectCar(this)` 进行的，那么我们就通过这个方法来看一下 Dagger2 是如果实现依赖注入的。   

```
DaggerCarComponent.create().injectCar(this)
    DaggerCarComponent.create()
        new Builder().build()  // Builder 是 DaggerCarComponent 内部类，负责生成 DaggerCarComponent
            new EngineModule()
            new DaggerCarComponent(engineModule)
    DaggerCarComponent.injectCar(Car car)
        DaggerCarComponent.injectCar2(car) // injectCar过程中完成GasEnergy初始化并赋值给变量
            EngineModule_ProvideEngineFactory.provideEngine(engineModule)
                EngineModule.provideEngine()
                    new GasEnergy()
            Car_MembersInjector.injectEnergy()
                ((Car) instance).energy = (GasEnergy) energy // 完成注入
```

Module 类是通过 provideXXX 来提供注入的实例对象。