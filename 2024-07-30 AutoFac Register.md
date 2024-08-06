# 学习内容
## AutoFac 服务注册方式
### 注册 Register
使用Lambda表达式或者委托注册服务的方法之一，允许服务实例化时执行自定义逻辑，基于上下文创建，依赖解析，固定参数传入。
```csharp
// Sample 1
builder.Register(c => new MyService("ExampleService")).As<IMyService>(); // 注册为接口
```

```csharp
// Sample 2
builder.Register(c => 
{
	var context = c.Resolve<IComponentContext>();
	var otherService = context.Resolve<IOtherService>();
	return new MyServiceWithContext("ContextService", otherService);
}).As<IMyService>(); // 注册为接口
```

### 注册类型 RegisterType
最常用的服务注册方法之一，允许指定具体的类型注册为接口和自身。
```csharp
builder.RegisterType<MyService>().As<IMyService>().AsSelf();
```

#### 注册元数据 WithMetadata
`WithMetadata` 方法使得在注册服务时可以附加额外的信息，这些信息可以在解析时使用，帮助进行服务选择和决策。
```csharp
builder.RegisterType<ServiceA>().As<IMyService>().WithMetadata("Category", "A");
builder.RegisterType<ServiceB>().As<IMyService>().WithMetadata("Category", "B");

var services = scope.Resolve<IEnumerable<Meta<IMyService>>>();
var serviceA = services.First(s => (string)s.Metadata["Category"] == "A");
var serviceB = services.First(s => (string)s.Metadata["Category"] == "B");
```

#### 注册生命周期
允许指定服务实例的生命周期，包括单例、每依赖实例、每生命周期范围实例等。
``` csharp
builder.RegisterType<MyService>().As<IMyService>().SingleInstance();
builder.RegisterType<MyService>().As<IMyService>().InstancePerDependency();
builder.RegisterType<MyService>().As<IMyService>().InstancePerLifetimeScope();
```

#### 注册 Keyed 和 Named 服务
`Keyed` 和 `Named` 注册方法提供了一种方式来标识和区分同一个接口的不同实现。这在需要选择特定的实现或有多个实现同一个接口的情况下非常有用。

- Keyed
	```csharp
	// 当 Key 为 String 类型时等价 Named 方法。
	builder.RegisterType<MyServiceA>().Keyed<IMyService>("KeyA");
	builder.RegisterType<MyServiceB>().Keyed<IMyService>("KeyB");
	
	// 手动获取
	var serviceA = scope.ResolveKeyed<IMyService>("KeyA");
	var serviceB = scope.ResolveKeyed<IMyService>("KeyB");
	
	// 注入获取
	public class Demo
	{
		public Demo([KeyFilter("KeyA")] IMyService s1, [KeyFilter("KeyB")] IMyService s2)
		{
			Console.WriteLine(s1.Say());
			Console.WriteLine(s2.Say());
		}
	}
	
	// 需要对注入的服务标记 WithAttributeFiltering 标识
	builder.RegisterType<Demo>().WithAttributeFiltering();
	```

- Named
	```csharp
	builder.RegisterType<MyServiceA>().Named<IMyService>("ServiceA");
	builder.RegisterType<MyServiceB>().Named<IMyService>("ServiceB");
	
	var serviceA = scope.ResolveNamed<IMyService>("ServiceA");
	var serviceB = scope.ResolveNamed<IMyService>("ServiceB");
	```

### 注册实例 RegisterInstance
将一个已存在的实例对象注册为接口和自身。
``` csharp
var myService = new MyService();
builder.RegisterInstance(myService).As<IMyService>().AsSelf();
```

### 注册程序集 RegisterAssemblyType
通过扫描程序集，筛选类后进行统一注册。
```csharp
builder.RegisterAssemblyTypes(Assembly.GetExecutingAssembly())
       .Where(t => t.Name.EndsWith("Service"))
       .AsImplementedInterfaces();
```

### 注册模块 RegisterModule
模块是一种用于封装和组织服务注册逻辑的机制。模块使得服务注册逻辑更加模块化、可重用和易于管理，特别是在大型项目或有多个依赖项时。
```csharp
public class MyModule : Module  
{  
    protected override void Load(ContainerBuilder builder)  
    {
        base.Load(builder);
        // Register Other
    }
}

builder.RegisterModule<MyModule>(); // 注册模块
```

### 注册范型 RegisterGeneric
可以在依赖注入容器中注册并解析泛型服务。这在有多个具有相同结构但不同类型参数的实现时非常有用。

```csharp
public interface IRepository<T>
{
    void Add(T item);
}

public class Repository<T> : IRepository<T>
{
    public void Add(T item)
    {
        Console.WriteLine($"Adding item of type {typeof(T).Name}");
    }
}

builder.RegisterGeneric(typeof(Repository<>)).As(typeof(IRepository<>));

var intRepository = scope.Resolve<IRepository<int>>();
var stringRepository = scope.Resolve<IRepository<string>>();
```