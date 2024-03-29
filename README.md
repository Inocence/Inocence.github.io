# Yii2与Laravel生命周期的理解

## Yii2生命周期的理解

### 流程分析

#### yii\BaseYii

这个是Yii的基础类，都是一些静态属性和静态方法，其中最重要的属性就是$app主体对象和$container容器对象，
$container容器对象保存了通过BaseYii::createObject创建的对象

#### **yii\web\Application** extends \yii\base\Application

- 设置初始化组件
- 实现handleRequest方法，自定义处理逻辑
- 保存controller实例

#### **yii\base\Application** extends Module

- 初始化组件
- 注册exception handler

#### **yii\base\Controller** extends Component

- 调用beforaction&afteraction，创建yii\base\InlineAction，并执action

#### **yii\base\InlineAction** extends Action

- 将业务逻辑action构造为一个InlineAction并执行，这样的好处是可以定制各种action的

#### **abstract class yii\base\Module** extends ServiceLocator

Module的意思是包含MVC的子应用
- 管理module实例
- 执行mvc通用逻辑

#### **yii\di\ServiceLocator** extends Component

- 管理组件

#### **yii\base\Component** extends BaseObject

- 管理行为和事件

#### **yii\base\Behavior** extends BaseObject

- 管理事件

#### **yii\base\BaseObject** implements Configurable

- 配置对象的属性值
- 调用子类init方法

#### events()方法

- key为事件名，value为执行方法的数组
- 在执行trigger的时候，会根据key去找这里对应的方法

### 执行流程分析

- \yii\base\Application实例化并将$this主体对象挂载在BaseYii::$app上，并设置module实例，然后自定义$config数据，注册errorhandler，配置主体对象$this
- Component::__construct($config)触发yii\base\BaseObject调用主体对象的init方法（个人觉得这里静态方法调用了动态的对象很不规范），init方法会调用bootstrap方法，开始初始化组件
- \yii\base\Application的run()触发yii\web\Application的handleRequest()方法执行
- yii\web\Application的handleRequest()方法触发yii\base\Application的runAction()方法执行
- yii\base\Application的runAction()方法触发yii\base\Controller的runAction()方法执行
- yii\base\Controller的runAction()方法触发yii\base\Controller的beforeAction()方法执行
- yii\base\Controller的beforeAction()方法触发yii\base\Component的__call()方法执行，此时加载yii\base\Controller子类的behavior()方法，behavior()会加载events()方法
- 后面就是执行action了，不重要了。。。

### 一些高级概念的理解

#### 什么是container容器？

容器就是单例对象，它的属性中包含了很多已经实例化的对象，对象分为单例和非单例，在取出对象的时候有区别，取出单例对象的时候直接返回，
取出非单例对象的时候，回去重新创建

#### 什么是DI(依赖注入)？

在YII中，依赖注入主要用在服务定位器ServiceLocator中，比如在调用某个组件的时候Yii::$app->email，会去检查$container容器中是否存在这个对象，
如果不存在则创建该对象，并存入容器中

#### 什么是behavior行为？

每个controller可以关联多个behavior，通过behavior()方法配置，每个behavior其实本质上就是event事件的集合，通过events()方法配置

#### 什么是event事件

事件其实就是一个key-value的数组，key为事件名，value为事件的方法。这样在调用的时候，就只需要执行trigger()方法，
传入事件名而不用再去执行具体的事件方法了，是的代码更加整洁易于维护

## Laravel生命周期的理解

### 流程分析

#### app.php

- 实例化application对象
- 注册Kernel，Handler单例

主要代码：
```php
$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);
```

#### index.php

- 实例化注册Kernel对象
- 实例化request对象

```php
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);

$response->send();

$kernel->terminate($request, $response);
```

#### App\Http\Kernel extends Illuminate\Foundation\Http\Kernel

- 定义默认middleware

```php
protected $routeMiddleware = [
    'auth' => \Illuminate\Auth\Middleware\Authenticate::class,
    'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
    'bindings' => \Illuminate\Routing\Middleware\SubstituteBindings::class,
    'can' => \Illuminate\Auth\Middleware\Authorize::class,
    'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
];
```

#### Illuminate\Foundation\Http\Kernel

此类是组合$app，$router，$request进行处理的地方，也是业务部分真正启动的地方

- 定义默认bootstrapper

```php
protected $bootstrappers = [
    \Illuminate\Foundation\Bootstrap\LoadEnvironmentVariables::class,
    \Illuminate\Foundation\Bootstrap\LoadConfiguration::class,
    \Illuminate\Foundation\Bootstrap\HandleExceptions::class,
    \Illuminate\Foundation\Bootstrap\RegisterFacades::class,
    \Illuminate\Foundation\Bootstrap\RegisterProviders::class,
    \Illuminate\Foundation\Bootstrap\BootProviders::class,
];
```

#### \Illuminate\Foundation\Bootstrap\RegisterProviders

- 调用application的注册方法

```php
public function bootstrap(Application $app)
{
    $app->registerConfiguredProviders();
}
```

#### registerConfiguredProviders方法

- 根据config配置(app.providers)，批量载入providers

```php
public function registerConfiguredProviders()
{
    $providers = Collection::make($this->config['app.providers'])
                    ->partition(function ($provider) {
                        return Str::startsWith($provider, 'Illuminate\\');
                    });

    $providers->splice(1, 0, [$this->make(PackageManifest::class)->providers()]);

    (new ProviderRepository($this, new Filesystem, $this->getCachedServicesPath()))
                ->load($providers->collapse()->toArray());
}
```

### 核心思想

### middleware

- middleware中的":"表示分隔别名和参数","表示分隔多个参数
- middleware通过pipeline的栈进行遍历
- middlewarePriority表示高优先级middleware
- App\Http\Kernel中的middleware表示全局middleware会首先在pipeline中进行处理
- $this->dispatchToRouter()会将$middleware，$middlewareGroups，$routeMiddleware聚合起来处理在pipeline中进行处理

### guard

- 表示认证方式，如session或者token

#### 构造对象

- singleton() 在容器bindings属性中注册为单例模式
- Container->make() 创建对象，并判断在bindings是否注册为单例
- Container->instance() 直接注入单例对象
- Container::$instance 为储存单例的全局变量

#### 数组取值方式

- 实现了ArrayAccess接口的offsetGet方法，通过make获取对象

#### 常用路径

- 在application中

## 总结

其实这些高级概念并不复杂，不要为名字吓到了，多看源码多理解，这里推荐[xdebug](https://xdebug.org/)去追踪代码的执行，因为框架源码中经常用到魔术方法和设计模式，
导致很难直观的看出执行的流程
