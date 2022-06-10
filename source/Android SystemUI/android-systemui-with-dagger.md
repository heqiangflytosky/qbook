## 简介

在 SystemUI中，很多对象的使用都是通过依赖注入的方式来实现。    
比如我们需要获取某些对象，可以通过下面的方式使用：    

```
Dependency.get(NavigationModeController.class)
```

Denpency类是SystemUI中主要的依赖实例的管理类：

```
@SysUISingleton
public class Dependency {
    @Inject Lazy<ActivityStarter> mActivityStarter;
    @Inject Lazy<BroadcastDispatcher> mBroadcastDispatcher;
    @Inject Lazy<AsyncSensorManager> mAsyncSensorManager;
    @Inject Lazy<BluetoothController> mBluetoothController;
    @Inject Lazy<LocationController> mLocationController;
    @Inject Lazy<RotationLockController> mRotationLockController;
    @Inject Lazy<NetworkController> mNetworkController;
    @Inject Lazy<ZenModeController> mZenModeController;
    @Inject Lazy<HotspotController> mHotspotController;
    @Inject Lazy<CastController> mCastController;
    @Inject Lazy<FlashlightController> mFlashlightController;
    ......
```

比如我们新建了一个 `QSStatusBarController` 类，如何加入 Dependency 管理呢？

```
//QSStatusBarController.java
@SysUISingleton
public class QSStatusBarController {

    @Inject
    QSStatusBarController(Context context, PrivacyDialogController privacyDialogController, PrivacyItemController privacyItemController, UiEventLogger uiEventLogger, PrivacyLogger privacyLogger) {
        mContext = context;
        ....
    }
```

```
@Inject Lazy<QSStatusBarController> mQSStatusBarController;
mProviders.put(QSStatusBarController.class, mQSStatusBarController::get);
// 获取实例
mController = Dependency.get(QSStatusBarController.class);
```

在这里我们就可能会有点疑问：QSStatusBarController 是何时、如何初始化的呢？它的构造方法参数是如何初始化的呢？答案留到后面介绍。    

比如 NotificationPanelViewController 的获取：    
在 NotificationShadeWindowViewController 中用到了 NotificationPanelViewController 实例，直接通过构造方法参数传入。

```
    @Inject
    public NotificationShadeWindowViewController(
            ......
            NotificationPanelViewController notificationPanelViewController,
            ......) {
        ......
        mNotificationPanelViewController = notificationPanelViewController;
        ......
    }
```

它也是通过 dagger 依赖注入的，由 StatusBarComponent 来提供对象的创建和获取。

```
@Subcomponent(modules = {StatusBarViewModule.class})
@StatusBarComponent.StatusBarScope
public interface StatusBarComponent {
    ......
    @StatusBarScope
    NotificationPanelViewController getNotificationPanelViewController();
```

再比如获取 UI 组件 QSPanel 和 QSContainerImpl 实例：    
在 QSContainerImplController 有用到 QSContainerImpl 对象，它也是通过 QSContainerImplController 构造方法传入 QSContainerImpl 的，其实也是通过 dagger 由 QSFragmentModule 完成依赖注入的。

```
    @Inject
    QSContainerImplController(QSContainerImpl view, QSPanelController qsPanelController,
            QuickStatusBarHeaderController quickStatusBarHeaderController,
            ConfigurationController configurationController) {
        super(view);
        mQsPanelController = qsPanelController;
        mQuickStatusBarHeaderController = quickStatusBarHeaderController;
        mConfigurationController = configurationController;
    }
```

```
@Module
public interface QSFragmentModule {
    ......
    /** */
    @Provides
    static QSPanel provideQSPanel(@RootView View view) {
        return view.findViewById(R.id.quick_settings_panel);
    }

    /** */
    @Provides
    static QSContainerImpl providesQSContainerImpl(@RootView View view) {
        return view.findViewById(R.id.quick_settings_container);
    }
    ......
```

上面的方法都会在 DaggerGlobalRootComponent 中生成对应的 `Provider<T>`。 在 XXX__Factory (比如QSContainerImplController_Factory)方法中通过 XXProvider.get()获取对象。    
SystemUI 中所有的类的注入的依赖的 Provider 都由 DaggerGlobalRootComponent.java 来提供。    
在 SystemUI 中由 Dagger 生成的文件个数有776个之多。所以 Dagger 在 SystemUI 中扮演了重要的角色。   
Dagger 相关的代码在 `com/android/systemui/dagger`、`com/android/keyguard/dagger` 以及各个模块的 dagger 目录中，比如：`com/android/systemui/qs/dagger`。    
Dagger 生成的代码在 `out/soong/.intermediates/frameworks/base/packages/SystemUI/SystemUI-core/android_common/kapt/gen/sources/` 中。   

在 SystemUI 中，只有一个DaggerXXXComponent 类：DaggerGlobalRootComponent，那是因为其他所有的组件都是 GlobalRootComponent 的子组件。   

下面我们举几个具有代表性 的例子来讲述一下他们的注入流程。   
LocalBluetoothManager 由于是第三方提供的类，因此需要手动创建对象完成注入。   
NotificationPanelViewController 是SystemUI内部的类，构造方法使用 `Inject` 修饰，因此 dagger 可以完成自动创建。   
QSPanel 是 QSContainerImpl UI控件，Android 在加载布局是完成对象的创建，我们需要把该对象提取出来并提供给 dagger。   

```
// DaggerGlobalRootComponent.java
public final class DaggerGlobalRootComponent implements GlobalRootComponent {
    private Provider<LocalBluetoothManager> provideLocalBluetoothControllerProvider;
    ......
    private Provider<NotificationPanelViewController> notificationPanelViewControllerProvider;
    ......
    private final class QSFragmentComponentImpl implements QSFragmentComponent {
        private Provider<QSPanel> provideQSPanelProvider;
        ......
        private Provider<QSContainerImpl> providesQSContainerImplProvider;
    ......
```

然后通过各自的 XXXFactory 类完成对象的构建。

```
// DaggerGlobalRootComponent.java
this.provideLocalBluetoothControllerProvider = DoubleCheck.provider(DependencyProvider_ProvideLocalBluetoothControllerFactory.create(DaggerGlobalRootComponent.this.contextProvider, ......));
this.notificationPanelViewControllerProvider = DoubleCheck.provider(NotificationPanelViewController_Factory.create(getNotificationPanelViewProvider, ......));
this.provideQSPanelProvider = QSFragmentModule_ProvideQSPanelFactory.create(provideRootViewProvider);
this.providesQSContainerImplProvider = QSFragmentModule_ProvidesQSContainerImplFactory.create(provideRootViewProvider);

```

```
//NotificationPanelViewController_Factory.java
  public static NotificationPanelViewController newInstance(NotificationPanelView view,
      Resources resources, Handler handler, LayoutInflater layoutInflater,
      ......
      EmergencyButtonController.Factory emergencyButtonControllerFactory) {
    return new NotificationPanelViewController(view, resources, ...... remoteInputManager, controlsComponent, emergencyButtonControllerFactory);
  }
```

LocalBluetoothManager 是 settingslib 提供的一个类，没法做到 Inject 它的构造方法，因此，在 DependencyProvider 中完成了对象的手动创建。

```
// DependencyProvider_ProvideLocalBluetoothControllerFactory.java
  @Nullable
  public static LocalBluetoothManager provideLocalBluetoothController(Context context,
      Handler bgHandler) {
    return DependencyProvider.provideLocalBluetoothController(context, bgHandler);
  }
// DependencyProvider.java
    @SuppressLint("MissingPermission")
    @SysUISingleton
    @Provides
    @Nullable
    static LocalBluetoothManager provideLocalBluetoothController(Context context,
            @Background Handler bgHandler) {
        return LocalBluetoothManager.create(context, bgHandler, UserHandle.ALL);
    }
```

QSPanel 和 QSContainerImpl 的注入是比较类似的，

```
// QSFragmentModule_ProvideQSPanelFactory.java
  public static QSPanel provideQSPanel(View view) {
    return Preconditions.checkNotNullFromProvides(QSFragmentModule.provideQSPanel(view));
  }
// QSFragmentModule.java
    @Provides
    static QSPanel provideQSPanel(@RootView View view) {
        return view.findViewById(R.id.quick_settings_panel);
    }
```

另外还有一些 SystemUI 系列服务类生成，它们是在 SystemUIService 服务启动是完成注入的，比如：

```
SystemUIService.onCreate()
    SystemUIApplication.startServicesIfNeeded
        ContextComponentResolver.resolveSystemUI
            SystemUIAppComponentFactory.resolve
                StatusBarPhoneModule_ProvideStatusBarFactory.get
                    StatusBarPhoneModule.provideStatusBar
                        new StatusBar
            VolumeUI_Factory.get
                VolumeUI_Factory.newInstance()
                    VolumeUI.init
```

上面例子中的两个类一个是通过 `@Inject` 构造方法，一个是通过 Module 类的 `@Provides` 方法来提供依赖的。    

另外还有比如Android 四大组件的注入，这些后面会举例介绍。

## 组件关系

WMComponent 和 SysUIComponent 都是 GlobalRootComponent 的子组件。

```
GlobalRootComponent
    WMComponent
    SysUIComponent
        StatusBarComponent
        NotificationRowComponent
        DozeComponent
        ExpandableNotificationRowComponent
        KeyguardBouncerComponent
        NotificationShelfComponent
        FragmentService.FragmentCreator
        QSFragmentComponent
        SectionHeaderControllerSubcomponent
```

## 模块关系

```
GlobalRootComponent
    WMModule
        WMShellModule
            WMShellBaseModule
                WMShellConcurrencyModule
    GlobalModule
        FrameworkServicesModule
        GlobalConcurrencyModule
    SysUISubcomponentModule
        DefaultComponentBinder
            DefaultActivityBinder   Binds Activity
            DefaultBroadcastReceiverBinder  Binds BroadcastReceiver
            DefaultServiceBinder  Binds Service
        DependencyProvider  // 类提供一些实例的手动创建
            NightDisplayListenerModule
        SystemUIBinder
            RecentsModule
            StatusBarModule
                StatusBarPhoneModule
                    StatusBarPhoneDependenciesModule
                StatusBarDependenciesModule
                NotificationsModule
                    NotificationSectionHeadersModule
                NotificationRowModule
            KeyguardModule
                FalsingModule
        SystemUIModule
            AppOpsModule
            AssistModule
            ClockModule
            ControlsModule
            DemoModeModule
            FalsingModule
            LogModule
            PeopleHubModule
            PluginModule
            ScreenshotModule
            SensorModule
            SettingsModule
            SettingsUtilModule
            SmartRepliesInflationModule
            StatusBarPolicyModule
            SysUIConcurrencyModule
            TunerModule
            UserModule
            UtilModule
            VolumeModule
            WalletModule
        SystemUIDefaultModule
            MediaModule
            PowerModule
            QSModule
                MediaModule
                QSFlagsModule
```

## 初始化流程

在 SystemUIFactory 中初始化了 GlobalRootComponent、WMComponent 和 SysUIComponent。   

```
SystemUIFactory.init(Context context, boolean fromTest)
    buildGlobalRootComponent()
        DaggerGlobalRootComponent.builder().context(context).build()
    GlobalRootComponent.getWMComponentBuilder().build()
    GlobalRootComponent.getSysUIComponent().build()
    SysUIComponent.createDependency().start()
```

注意看，在构建 DaggerGlobalRootComponent 时，有一个重要的方法 `context(context)`，为什么这里需要 context 参数呢？    
我们在构造一些类的时候，它们有很多的参数，那么这些参数是如何被初始化的呢？    
这些自动注入的类的参数有什么要求吗？    
通过下面对几个类的构建过程的介绍，会解开这个谜团。    

另外这里顺便提一个问题，前面介绍 Dagger 基础知识的时候，我们在构建 CarComponent 时代码是这些写的：    

```
Car car = new Car();
DaggerCarComponent.create().injectCar(car);
Log.e("Test",car.getEngine().name());
```

我们首先创建了一个 Car 对象，然后通过一个 injectCar 方法传递给了 CarComponent。那么在 SystemUI 构建 GlobalRootComponent 时却没有输入任何对象给它，这是为什么呢？    
首先我们知道 Component 是连接依赖提供方和依赖需求方的桥梁，它就像是给加工厂，来提供依赖对象并绑定到依赖需求方。那么 Car 是依赖需求方，而且 Car 是个普通的类，我们必须要有这样一个对象，Dagger 才会把它需要的依赖对象注入给它。    
那么现在我们换一种写法，不需要手动生成 Car 对象。   

```
// 提供一个Inject修饰的构造方法
public class Car {
    @Inject
    @Named("petrol")
    Engine engine;
    ....

    @Inject
    public Car() {

    }

    ....
}
```

```
// 去掉 injectCar 方法，提供一个生产 Car 对象的方法 provideCar()
@Component(modules = EngineModule.class)
public interface CarComponent {
    //void injectCar(Car car);
    Car provideCar();
}
```

```
// 其实也就是 Dagger 为我们自动生成了 Car 对象
        CarComponent component = DaggerCarComponent.builder().build();

        Log.e("Test",component.provideCar().getEngine().name());
        Log.e("Test",component.provideCar().getEngine2().name());
```

我们来看看Dagger为我们生成了哪些代码：    

```
//DaggerCarComponent.java
  @Override
  public Car provideCar() {
    return injectCar(Car_Factory.newInstance());
  }

  private Car injectCar(Car instance) {
    Car_MembersInjector.injectEngine(instance, EngineModule_ProvideEngineFactory.provideEngine());
    Car_MembersInjector.injectEngine2(instance, new ElectricEngine());
    Car_MembersInjector.injectSetAAB(instance, new AAB());
    return instance;
  }
```

```
//Car_Factory.java
  public static Car newInstance() {
    return new Car();
  }
```

## 介绍 NavigationModeController

NavigationModeController 的实例可以通过下面的方法来获取：

```
Dependency.get(NavigationModeController.class)
```


```
// DaggerGlobalRootComponent.java
this.navigationModeControllerProvider = DoubleCheck.provider(NavigationModeController_Factory.create(DaggerGlobalRootComponent.this.contextProvider, (Provider) deviceProvisionedControllerImplProvider, provideConfigurationControllerProvider, provideUiBackgroundExecutorProvider));
rovider));
this.contextProvider = InstanceFactory.create(contextParam);
```

NavigationModeController 的实例化是在 Dagger 生成类 NavigationModeController_Factory 中实现的。

```
//NavigationModeController_Factory.java

public final class NavigationModeController_Factory implements Factory<NavigationModeController> {
  private final Provider<Context> contextProvider;

  private final Provider<DeviceProvisionedController> deviceProvisionedControllerProvider;

  private final Provider<ConfigurationController> configurationControllerProvider;

  private final Provider<Executor> uiBgExecutorProvider;

  public NavigationModeController_Factory(Provider<Context> contextProvider,
      Provider<DeviceProvisionedController> deviceProvisionedControllerProvider,
      Provider<ConfigurationController> configurationControllerProvider,
      Provider<Executor> uiBgExecutorProvider) {
    this.contextProvider = contextProvider;
    this.deviceProvisionedControllerProvider = deviceProvisionedControllerProvider;
    this.configurationControllerProvider = configurationControllerProvider;
    this.uiBgExecutorProvider = uiBgExecutorProvider;
  }

  @Override
  public NavigationModeController get() {
    return newInstance(contextProvider.get(), deviceProvisionedControllerProvider.get(), configurationControllerProvider.get(), uiBgExecutorProvider.get());
  }

  public static NavigationModeController_Factory create(Provider<Context> contextProvider,
      Provider<DeviceProvisionedController> deviceProvisionedControllerProvider,
      Provider<ConfigurationController> configurationControllerProvider,
      Provider<Executor> uiBgExecutorProvider) {
    return new NavigationModeController_Factory(contextProvider, deviceProvisionedControllerProvider, configurationControllerProvider, uiBgExecutorProvider);
  }

  public static NavigationModeController newInstance(Context context,
      DeviceProvisionedController deviceProvisionedController,
      ConfigurationController configurationController, Executor uiBgExecutor) {
    return new NavigationModeController(context, deviceProvisionedController, configurationController, uiBgExecutor);
  }
}
```

来看一下 NavigationModeController 的构造方法：    

```
@SysUISingleton
public class NavigationModeController implements Dumpable {
    @Inject
    public NavigationModeController(Context context,
            DeviceProvisionedController deviceProvisionedController,
            ConfigurationController configurationController,
            @UiBackground Executor uiBgExecutor) {
```

它也有自己的参数，其实这些参数的初始化和 NavigationModeController 自身初始化是类似的，他们有层层依赖的关系，当这些依赖需要用到时，它们会去检查该对象有没有被初始化。这样就完成了它们自身的初始化。

```
SystemUIAppComponentFactory.instantiateServiceCompat()
    ContextComponentResolver.resolveService()
        ContextComponentResolver.resolve()
            KeyguardService_Factory.get()
                KeyguardModule_NewKeyguardViewMediatorFactory.get()
                    NavigationModeController_Factory.get()
                        NavigationModeController_Factory.newInstance(c)
                            NavigationModeController.<init>()
```
KeyguardService 构造方法对 KeyguardViewMediator 有依赖，而 KeyguardViewMediator 对 NavigationModeController 又有依赖，这样就层层通过依赖注入创建了对象。    

这里面有一个非常重要的参数，Context 参数，大部分的类的构造方法都需要这个参数，那么这个参数是怎么注入进来的呢？
我们以 DeviceProvisionedController 对象的创建来看它如何绑定 context 的。

```
  private DaggerGlobalRootComponent(GlobalModule globalModuleParam, Context contextParam) {
    this.context = contextParam;
    initialize(globalModuleParam, contextParam);
  }

```

```
public interface GlobalRootComponent {

    /**
     * Builder for a GlobalRootComponent.
     */
    @Component.Builder
    interface Builder {
        @BindsInstance
        Builder context(Context context);

        GlobalRootComponent build();
    }
```

```
//SystemUIFactory.java
    protected GlobalRootComponent buildGlobalRootComponent(Context context) {
        return DaggerGlobalRootComponent.builder()
                .context(context)
                .build();
    }
```

## 介绍 QSFragmen

先来看看 QSFragmen 的创建流程，由于 QSFragment 的构造方法也是被 `@Inject` 修饰的，因此，它是可以通过注入的方式完成对象的创建：

```
StatusBar.createAndAddWindows()
    StatusBar.makeStatusBarView()
        StatusBar.createDefaultQSFragment()
            FragmentHostManager.get(mNotificationShadeWindowView).create(QSFragment.class)
                ExtensionFragmentManager.instantiateWithInjections()
                    FragmentService.FragmentCreator.createQSFragment()
                        DaggerGlobalRootComponent.FragmentCreatorImpl.createQSFragment()
                            new QSFragment()
```

在 FragmentService 中注入 QSFragment    

```
// FragmentService.java
@SysUISingleton
public class FragmentService implements Dumpable {

    ......
    @Subcomponent
    public interface FragmentCreator {
        /** Factory for creating a FragmentCreator. */
        @Subcomponent.Factory
        interface Factory {
            FragmentCreator build();
        }
        /**
         * Inject a QSFragment.
         */
        QSFragment createQSFragment();

        /** Inject a CollapsedStatusBarFragment. */
        CollapsedStatusBarFragment createCollapsedStatusBarFragment();
    }
```


```
// DaggerGlobalRootComponent.java
      @Override
      public QSFragment createQSFragment() {
        return new QSFragment(SysUIComponentImpl.this.remoteInputQuickSettingsDisablerProvider.get(),......);
      }
```

再来介绍一下 QSFragment 构造方法这么多的参数的创建过程，我们就以 StatusBarStateController 为例来介绍一下。  
StatusBarStateControllerImpl 实现了 StatusBarStateController接口，被 `@SysUISingleton` 标注，构造方法被 `@Inject` 标注，因此它是以单例模式的形式进行注入的。  
当实例化 QSFragment 时，StatusBarStateControllerImpl 还没有实例化，就需要创建 StatusBarStateControllerImpl 对象：

```
    @Inject
    public StatusBarStateControllerImpl(UiEventLogger uiEventLogger) {
        mUiEventLogger = uiEventLogger;
        for (int i = 0; i < HISTORY_SIZE; i++) {
            mHistoricalRecords[i] = new HistoricalState();
        }
    }
```

而创建 StatusBarStateControllerImpl 对象又需要 UiEventLogger ，那么 UiEventLogger 有是哪里提供的呢？UiEventLogger 是framework中提供的一个类，它是通过创建 Module 的形式在 GlobalModule 中完成注入的，而且它也是单例的。

```
@Module(includes = {
        FrameworkServicesModule.class,
        GlobalConcurrencyModule.class})
public class GlobalModule {

    ......

    /** Provides an instance of {@link com.android.internal.logging.UiEventLogger} */
    @Provides
    @Singleton
    static UiEventLogger provideUiEventLogger() {
        return new UiEventLoggerImpl();
    }
```

也是通过这种层层依赖的过程，通过依赖注入创建了对象。

## 介绍 QSContainerImpl

我们再来看一下 QSFragment 中 QSContainerImplController 成员变量是如何生成的。通过它我们来介绍一下 QSContainerImpl （QSFragment的容器布局）是如何完成实例对象的获取的。   
QSContainerImplController 的构造方法使用 `@Inject` 修饰，在 QSFragmentComponent 中，通过 `getQSContainerImplController` 实现对象的注入。

```
    @Inject
    QSContainerImplController(QSContainerImpl view, QSPanelController qsPanelController,
            QuickStatusBarHeaderController quickStatusBarHeaderController,
            ConfigurationController configurationController) {
        super(view);
        mQsPanelController = qsPanelController;
        mQuickStatusBarHeaderController = quickStatusBarHeaderController;
        mConfigurationController = configurationController;
    }

public interface QSFragmentComponent {

    /** Factory for building a {@link QSFragmentComponent}. */
    @Subcomponent.Factory
    interface Factory {
        QSFragmentComponent create(@BindsInstance QSFragment qsFragment);
    }

    ......

    /** Construct a {@link QSContainerImplController}. */
    QSContainerImplController getQSContainerImplController();
```

现在我们来看一下 QSContainerImpl 参数是如何注入的，它是通过 QSFragmentModule 的 providesQSContainerImpl(View view) 方法来完成实例的获取，而它的参数view又是通过 provideRootView() 方法注入的：

```
@Module
public interface QSFragmentModule {
    // 提供 RootView 类型的 View 参数
    @Provides
    @RootView
    static View provideRootView(QSFragment qsFragment) {
        return qsFragment.getView();
    }
    
    // 注入 QSContainerImpl
    @Provides
    static QSContainerImpl providesQSContainerImpl(@RootView View view) {
        return view.findViewById(R.id.quick_settings_container);
    }
```


## 介绍 DozeService

首先来看看ContextComponentResolver这个类，它是负责生成一些Android组件相关的类。

```
public class ContextComponentResolver implements ContextComponentHelper {
// 存放和Activity相关的Provider
    private final Map<Class<?>, Provider<Activity>> mActivityCreators;
    //存放和Service相关的类的 Provider
    private final Map<Class<?>, Provider<Service>> mServiceCreators;
    //存放实现SystemUI接口的类的Provider
    private final Map<Class<?>, Provider<SystemUI>> mSystemUICreators;
        //存放实现RecentsImplementation接口类OverviewProxyRecentsImpl的Provider
    private final Map<Class<?>, Provider<RecentsImplementation>> mRecentsCreators;
// 存放BroadcastReceiver的Provider
    private final Map<Class<?>, Provider<BroadcastReceiver>> mBroadcastReceiverCreators;
    
    
    private <T> T resolve(String className, Map<Class<?>, Provider<T>> creators) {
        try {
            Class<?> clazz = Class.forName(className);
            Provider<T> provider = creators.get(clazz);
            return provider == null ? null : provider.get();
        } catch (ClassNotFoundException e) {
            return null;
        }
    }
    
```

上面那些存放各种 Provider的Map时什么时候构建的呢？这里也是通过自动注入实现的。
相关的Module分别是 DefaultActivityBinder，DefaultBroadcastReceiverBinder，DefaultServiceBinder，SystemUIBinder，RecentsModule

```
@Module
public abstract class DefaultServiceBinder {
    /** */
    @Binds
    @IntoMap
    @ClassKey(DozeService.class)
    public abstract Service bindDozeService(DozeService service);
```

DozeService的生成为例：
DozeServic` @Inject`了构造方法，因此Dagger会生成一个DozeService_Factory工程类用于生成DozeService对象。

```
public class DozeService extends DreamService
        implements DozeMachine.Service, RequestDoze, PluginListener<DozeServicePlugin> {
  ......

    @Inject
    public DozeService(DozeComponent.Builder dozeComponentBuilder, PluginManager pluginManager) {
        mDozeComponentBuilder = dozeComponentBuilder;
        setDebug(DEBUG);
        mPluginManager = pluginManager;
    }
```

```
SystemUIAppComponentFactory.instantiateServiceCompat
    ContextComponentResolver.resolveService
        SystemUIAppComponentFactory.resolve
            DozeService_Factory.get
                DozeService_Factory.newInstance
```

来看一下 DozeService_Factory.newInstance 方法。

```
public final class DozeService_Factory implements Factory<DozeService> {
  private final Provider<DozeComponent.Builder> dozeComponentBuilderProvider;

  private final Provider<PluginManager> pluginManagerProvider;

  public DozeService_Factory(Provider<DozeComponent.Builder> dozeComponentBuilderProvider,
      Provider<PluginManager> pluginManagerProvider) {
    this.dozeComponentBuilderProvider = dozeComponentBuilderProvider;
    this.pluginManagerProvider = pluginManagerProvider;
  }

  ......

  public static DozeService newInstance(DozeComponent.Builder dozeComponentBuilder,
      PluginManager pluginManager) {
    return new DozeService(dozeComponentBuilder, pluginManager);
  }
}
```



