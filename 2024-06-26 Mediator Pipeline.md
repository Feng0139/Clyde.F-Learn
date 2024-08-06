# 学习内容
## AOP思想
AOP（Aspect Oriented Programming）是一种面相切面编程模式，旨在将其它模块与核心业务进行分离，以提高代码模块化程度。AOP把系统分解为不同的切面。

### OOP编程
假设有一个图书服务，有添加和移除图书两个业务方法，对于业务逻辑核心，还需要对传递的参数进行校验，日志记录，和事物处理，示例代码如下：
``` csharp
public class BookService
{
	public void Add(Book book)
	{
		// 参数校验
		Verification(book);
		// 核心业务
		_dataProvide.Add(book);
		// 日志记录
		_logger.LogInfomation("添加图书: {book.Name}");
	}
	
	public void Remove(Book book)
	{
		// 参数校验
		Verification(book);
		// 核心业务
		_dataProvide.Remove(book);
		// 日志记录
		_logger.LogInfomation("移除图书: {book.Name}");
	}
}
```

通过观察，其中对于参数校验、日志事物等代码，会重复出现在每一个业务方法中。仅仅按照OOP面相对象编程，很难将这些代码进行模块化。

图书服务应该关注自身业务即可，但是却还要在每个业务方法中进行参数校验、记录日志等功能，这些实际上不应该属于图书服务。

### 转换为AOP编程
我们主要目的就是抽离不属于业务逻辑的代码部分，即图书服务代码应该如下：
```
public class BookService
{
	public void Add(Book book)
	{
		// 核心业务
		_dataProvide.Add(book);
	}
	
	public void Remove(Book book)
	{
		// 核心业务
		_dataProvide.Remove(book);
	}
}
```

那么抽离出来的参数校验和日志记录功能，如何在业务方法执行前后调用？答案是[[#Pipeline 管道]]。

## Pipeline 管道
定义一条流水线，其中有很多工序，负责不同的加工。当一个工件从上流下时，会经过一道道工序处理，最终流到末端结束。

那么代码中设计也是类似的，将流水线定义为管道，工序定义为中间件，数据从第一个中间件处理，第二个、第三个... 直到最后的业务方法执行。

Mediator.Net 提供了管道机制，在 Autofac 注册后即可配置一系列中间件，如参数校验、日志记录等。

### 中间件
中间件是一种用于在管道中处理消息的组件。中间件可以在消息到达最终处理程序之前对消息进行操作或检查。每个中间件都实现 `IPipeSpecification` 接口，并定义 `Execute` 方法，该方法包含中间件的逻辑。

核心：
- IPipeSpecification: 中间件规范接口，定义了业务方法的扩展方法，如执行前、执行完成、执行后，验证逻辑以及异常捕获。

#### 规范实现
一个含有日志记录请求数据和结果数据的中间件规范实现。
``` CSharp
public class UnitOfWorkSpecification<TContext> : IPipeSpecification<TContext> where TContext : IContext<IMessage>  
{  
    private readonly ILogger<UnitOfWorkSpecification<TContext>> _logger;  
    private readonly IUnitOfWork _unitOfWork;
    
    public UnitOfWorkSpecification(ILogger<UnitOfWorkSpecification<TContext>> logger, IUnitOfWork unitOfWork)   
    {  
        _logger = logger;  
        _unitOfWork = unitOfWork;  
    }

	// 校验上下文的逻辑方法
    public bool ShouldExecute(TContext context, CancellationToken cancellationToken)  
    {
        return true;  
    }

	// 业务方法执行前
    public Task BeforeExecute(TContext context, CancellationToken cancellationToken)  
    {
        _logger.LogInformation($"{context.Message.GetType().Name} 执行请求 {JsonConvert.SerializeObject(context.Message)}");  
        return Task.CompletedTask;  
    }

	// 业务方法执行中
    public Task Execute(TContext context, CancellationToken cancellationToken)  
    {
        _logger.LogInformation($"{context.Message.GetType().Name} 执行中");  
        return Task.CompletedTask;  
    }

	// 业务方法执行后
    public Task AfterExecute(TContext context, CancellationToken cancellationToken)  
    {
        _logger.LogInformation($"{context.Message.GetType().Name} 执行结果 {JsonConvert.SerializeObject(context.Result)}");  
        return Task.CompletedTask;  
    }

	// 捕获异常
    public Task OnException(Exception ex, TContext context)  
    {
        throw ex;  
    }}
```

#### 定义扩展
编写中间件扩展
```
public static class UnitOfWorkMiddleWare  
{  
    public static void UseUnitOfWork<TContext>(this IPipeConfigurator<TContext> configurator, IUnitOfWork unitOfWork = null) where TContext : IContext<IMessage>  
    {
        if (unitOfWork == null && configurator.DependencyScope == null)  
        {
                throw new DependencyScopeNotConfiguredException(  
                $"{nameof(unitOfWork)} is not provided and IDependencyScope is not configured, Please ensure {nameof(unitOfWork)} is registered properly if you are using IoC container, otherwise please pass {nameof(unitOfWork)} as parameter");  
        }

		// 中间件所需参数
        unitOfWork ??= configurator.DependencyScope.Resolve<IUnitOfWork>();  
        var contextLogger = configurator.DependencyScope.Resolve<ILogger<UnitOfWorkSpecification<TContext>>>(); 

		// 添加到管道
        configurator.AddPipeSpecification(new UnitOfWorkSpecification<TContext>(contextLogger, unitOfWork));  
    }
```

#### 注册配置
配置管道的中间件
```
private void RegisterMediator(ContainerBuilder builder)  
{  
    var mediatorBuilder = new MediatorBuilder();  
  
    mediatorBuilder.RegisterHandlers(_assemblies);  
    mediatorBuilder.ConfigureGlobalReceivePipe(p =>  
    {  
        p.UseUnitOfWork();  <- 此处指定使用中间件
    });  
    builder.RegisterMediator(mediatorBuilder);  
}
```