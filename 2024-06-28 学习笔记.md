# 学习内容
`PractiseForFreya` 项目的 `xunit` 是向 `smart_faq` 项目学习的 = 我学习 `smart_faq` 项目 `xunit` :)

测试类的继承顺序
- TestUtilBase -> TestUtil -> XXX Util
- TestUtilBase -> TestBase -> XXX FixtureBase -> XXX Fixture

| 类名         | 描述                                       |
| ------------ | ------------------------------------------ |
| TestUtilBase | 测试和工具的基类，提供便捷的Run回调方法    |
| TestUtil     | 工具基类                                   |
| TestBase     | Fixture 基类，允许指定测试主题和测试数据库名称 |

smart_faq 项目集成测试 Fixtrue 与昨天学习考核项目的集成测试的 Fixtrue 不同。

区别如下：
- 考核项目是组合成一个集合，共享上下文。
- smart项目每一个集成测试都单独继承 Fixtrue，无法共享上下文。

资料
- [共享上下文](https://xunit.net/docs/shared-context)

## 关于测试无法注入 ILogger 对象
学习项目的测试示例，它的IOC容器是直接构建ContainerBuilder，并不含带Ilogger类型。所以若对象类需要被注入Ilogger会报错

解决办法
1. 安装 `Autofac.Extensions.DependencyInjection` 和 `Microsoft.Extensions.Logging` NuGet包。
2. 对 `containerBuilder` 注入含带 ILogger 的 `serviceCollection`
```
public void RegisterBaseContainer(ContainerBuilder builder, IConfiguration configuration)  
{  
    var serviceCollection = new ServiceCollection();  
    serviceCollection.AddLogging(builder =>  
    {  
        builder.AddConsole();  
    });
    builder.Populate(serviceCollection); 
     
    builder.RegisterModule(new PractiseForClydeModule(configuration, typeof(IntegrationTestFixtureBase).Assembly));  
}
```