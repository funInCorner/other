控制器(Controllers)
===========


在创建完资源类和指定哪些资源应该格式化之后，那么下一步需要做的是创建控制器方法将资源通过RESTful APIS暴露给终端用户。


Yii 提供两个控制器基类来简化你创建RESTful方法:
[[yii\rest\Controller]] 和 [[yii\rest\ActiveController]]。这两个控制器不同之处是后者提供了一系列处理[Active Record](db-active-record.md)资源的缺省方法。所以如果你使用 [Active Record](db-active-record.md)和内置的方法也能满足你的需求，你可以考虑你的控制类继承 [[yii\rest\ActiveController]]，它将允许你用最少的代码创建强大的RESTful APIs。


[[yii\rest\Controller]] 和 [[yii\rest\ActiveController]] 提供以下特点，一些特点将会在下面的几个部分进行详细的描述：


* HTTP 方法验证；
* [内容协商和数据格式化](rest-response-formatting.md)；
* [身份验证](rest-authentication.md)；
* [速率限制](rest-rate-limiting.md)。


[[yii\rest\ActiveController]]  额外的特供了以下特点：

* 一个控制器通常有的方法: `index`, `view`, `create`, `update`, `delete`, `options`;
* 在用户请求资源和操作时进行认证。


## 创建控制器类 <span id="creating-controller"></span>


当创建一个新的控制器类，惯例上在命名一个控制器类使用资源的类型名称和单数形式。例如，为用户的信息服务，那么控制器可以命名为`UserController`。



创建一个新的方法类似于给一个Web 程序创建一个方法。唯一不同的地方它呈现结果不是通过调用`render()`来绘制视图，而RESTful方法是直接返回的数据。 [[yii\rest\Controller::serializer|serializer]] 和 [[yii\web\Response|response object]] 将会处理将原始的数据转换为请求格式。例如，

```php
public function actionView($id)
{
    return User::findOne($id);
}
```


## 过滤器 <span id="filters"></span>

许多RESTful接口的功能是实现部分[过滤器](structure-filters.md)的[[yii\rest\Controller]]提供的。特别是,以下的过滤器将会依序处理：

* [[yii\filters\ContentNegotiator|contentNegotiator]]:支持内容协商，在[Response Formatting](rest-response-formatting.md)部分有讲解；
* [[yii\filters\VerbFilter|verbFilter]]: 支持 HTTP 方法验证;
* [[yii\filters\AuthMethod|authenticator]]: 支持用户身份验证,在[Authentication](rest-authentication.md)部分有讲解;
* [[yii\filters\RateLimiter|rateLimiter]]: 支持速率限制, 在[Rate Limiting](rest-rate-limiting.md)部分有讲解.

这些过滤器名称在[[yii\rest\Controller::behaviors()|behaviors()]] 方法已申明。你可以重写这些方法来配置一个独特的过滤器，禁止一些过滤器，或者 添加你自己的过滤器。例如，如果你想使用HTTP 基础的身份验证，你可以写以下的代码：

```php
use yii\filters\auth\HttpBasicAuth;

public function behaviors()
{
    $behaviors = parent::behaviors();
    $behaviors['authenticator'] = [
        'class' => HttpBasicAuth::className(),
    ];
    return $behaviors;
}
```


## 扩展 `ActiveController` <span id="extending-active-controller"></span>

If your controller class extends from [[yii\rest\ActiveController]], you should set
its [[yii\rest\ActiveController::modelClass||modelClass]] property to be the name of the resource class
that you plan to serve through this controller. The class must extend from [[yii\db\ActiveRecord]].

如果你的控制器类继承了[[yii\rest\ActiveController]]，你计划通过这个控制器来给资源类设置命名你应该设置它的[[yii\rest\ActiveController::modelClass||modelClass]]属性。


### 自定义方法 <span id="customizing-actions"></span>

默认情况下, [[yii\rest\ActiveController]] 提供了以下方法:

* [[yii\rest\IndexAction|index]]: 一页一页的资源列表；
* [[yii\rest\ViewAction|view]]: 返回一个指定资源的细节；
* [[yii\rest\CreateAction|create]]: 创建一个新资源；
* [[yii\rest\UpdateAction|update]]: 更新一个已存在的资源；
* [[yii\rest\DeleteAction|delete]]: 删除指定的资源；
* [[yii\rest\OptionsAction|options]]: 返回当前支持的HTTP方法。

所有这些方法都已通过[[yii\rest\ActiveController::actions()|actions()]]方法定义，你可以重写`actions()`配置这些方法或者禁用一些,如以下所示，

```php
public function actions()
{
    $actions = parent::actions();

    // disable the "delete" and "create" actions
    unset($actions['delete'], $actions['create']);

    // customize the data provider preparation with the "prepareDataProvider()" method
    $actions['index']['prepareDataProvider'] = [$this, 'prepareDataProvider'];

    return $actions;
}

public function prepareDataProvider()
{
    // prepare and return a data provider for the "index" action
}
```

请参个别类方法的注释来学习哪些配置选项是有效的。


### 执行权限检查 <span id="performing-access-check"></span>

当通过RESTful APIs暴露资源时，你经常需要检查当前用户是否有权限访问和操作所请求的资源(resource(s))。这可以通过重写 [[yii\rest\ActiveController::checkAccess()|checkAccess()]] 方法来完成，如下所示，

```php
/**
 * Checks the privilege of the current user.
 *
 * This method should be overridden to check whether the current user has the privilege
 * to run the specified action against the specified data model.
 * If the user does not have access, a [[ForbiddenHttpException]] should be thrown.
 *
 * @param string $action the ID of the action to be executed
 * @param \yii\base\Model $model the model to be accessed. If null, it means no specific model is being accessed.
 * @param array $params additional parameters
 * @throws ForbiddenHttpException if the user does not have access
 */
public function checkAccess($action, $model = null, $params = [])
{
    // check if the user can access $action and $model
    // throw ForbiddenHttpException if access should be denied
}
```

这个 `checkAccess()` 方法将会在[[yii\rest\ActiveController]] 默认的 actions中调用。如果你创建新的 actions并也行执行权限检查，你应该在新 actions中直接调用这个方法。

> 小窍门: 你可以通过使用[Role-Based Access Control (RBAC) component](security-authorization.md)来实现 `checkAccess()`。
