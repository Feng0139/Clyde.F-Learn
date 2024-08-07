# 考核项目
## 接入 AutoFac
1. 将 `WebApplication` 更改为 `Host` 通用主机。
```csharp
public static void Main(string[] args)  
{  
    Host.CreateDefaultBuilder(args)  
        .UseServiceProviderFactory(new AutofacServiceProviderFactory())  
        .ConfigureWebHostDefaults(webBuilder =>  
        {  
            webBuilder.UseStartup<Startup>();
        })
        .Build()  
        .Run();  
}
```
2. 配置 `ConfigureWebHostDefaults`，编辑启动类 `Startup`
```csharp
public class Startup
{
    public Startup(IConfiguration configuration)
    {
	    Configuration = configuration;
    }
    public IConfiguration Configuration { get; }
  
    public void ConfigureServices(IServiceCollection services)
    {
	    services.AddAuthorization();
        services.AddEndpointsApiExplorer();
        services.AddSwaggerGen();
    }
    
    public void ConfigureContainer(ContainerBuilder builder)
    {
	    builder.RegisterModule(new PractiseForClydeModule(Configuration, typeof(PractiseForClydeModule).Assembly));
    }
    
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env) 
    {
		if (env.IsDevelopment())  
        {
	        app.UseSwagger();  
            app.UseSwaggerUI();  
        }
		app.UseHttpsRedirection();  
        app.UseAuthorization();  
    }
}
```
3. 实现 `PractiseForClydeModule`
```csharp
public class PractiseForClydeModule : Module  
{  
    private readonly Assembly[] _assemblies;  
    private readonly IConfiguration _configuration;  
  
    public PractiseForClydeModule(IConfiguration configuration, params Assembly[] assemblies)  
    {
	    _assemblies = assemblies;  
        _configuration = configuration;  
    }
    
    protected override void Load(ContainerBuilder builder)  
    {
		RegisterDependency(builder);  
    }

	// 注册服务
    private void RegisterDependency(ContainerBuilder builder)  
    {
		var allServiceTypes = _assemblies.SelectMany(a => a.GetTypes())  
            .Where(t => t.IsClass   
					&& t.GetInterfaces().Any(i   
						=> i == typeof(ITransient)  
						|| i == typeof(ISingleton)  
						|| i == typeof(IScope))  
				)
			.ToList();  
        foreach (var type in allServiceTypes)  
        {
	        if (typeof(IScope).IsAssignableFrom(type))  
                builder.RegisterType(type).AsImplementedInterfaces().InstancePerLifetimeScope();  
            else if (typeof(ISingleton).IsAssignableFrom(type))  
                builder.RegisterType(type).AsImplementedInterfaces().SingleInstance();  
            else  
                builder.RegisterType(type).AsImplementedInterfaces();  
        }
    }
}
```

## 配置 EF ORM
### 安装 NuGet Package
1. `PractiseForClyde.Core` 项目安装如下 `Package` 
	1.  `Microsoft.EntityFrameworkCore`
		1. ORM框架
	2.  `Pomelo.EntityFrameworkCore.MySql`
		1. 提供链接到 Mysql 数据库
	3.  `EFCore.BulkExtensions`
		1.  用于批量操作（插入、更新和删除）的 EF 扩展

### 新建 IEntity 接口
存放于 `Domain` 文件夹，后续每个数据表都继承此接口
```csharp
public interface IEntityBase { }  
  
public interface IEntity : IEntityBase  
{  
    public long Id { get; set; }  
}  
  
public interface IEntity<T> : IEntityBase  
{  
    T Id { get; set; }  
}
```

### 新建 DBContext 类
此类代表数据库实体，需要提供链指定数据库，配置数据表类模型。
`PractiseForClydeDBContext` 类继承 `DbContext` 并存放于 `Data` 文件夹中。
#### 构造函数
注入 `IConfiguration`，获取连接语句。
```csharp
public class PractiseForClydeDBContext : DbContext, ITransient  
{  
    private readonly string _dbConnectionString;
    
	public PractiseForClydeDBContext(IConfiguration configuration)  
	{
		var section = configuration.GetSection("DBConnection:ConnectionString");  
		_dbConnectionString = section.Value;  
	} 
}
```

##### 配置项
在 `appsettings.json` 中设置连接语句。
```json
{
	"ConnectionStrings": {  
	  "Default": "server=localhost;userid=root;password=123456;database=db_order;Allow User Variables=True;"  
	}
}
```

#### 重写 OnConfiguring 方法
将方法实现改为连接 MySql 数据库语句，需要指定版本。
```csharp
public class PractiseForClydeDBContext : DbContext, ITransient  
{
	...
	
    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)  
    {
		optionsBuilder.UseMySql(_dbConnectionString, new MySqlServerVersion(new Version(8, 0, 3)));  
    }
}
```

#### 重写 OnModelCreating 方法
通过反射查找继承 `IEntity` 的数据表类，添加到 ORM 模型中
```csharp
public class PractiseForClydeDBContext : DbContext, ITransient  
{
	...
	
	protected override void OnModelCreating(ModelBuilder modelBuilder)  
    {
		Assembly.GetExecutingAssembly().GetTypes()  
            .Where(t=>t.IsClass && typeof(IEntity).IsAssignableFrom(t))  
            .ToList().ForEach(t =>
                {  
					if (modelBuilder.Model.FindEntityType(t) == null)  
                    {
						modelBuilder.Model.AddEntityType(t);  
                    }
	            }
	        );
    } 
}
```

### 新建 Repository 接口和类
#### 定义接口
```csharp
public interface IRepository  
{  
     ValueTask<TEntity?> GetByIdAsync<TEntity>(object id, CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task<List<TEntity>> GetAllAsync<TEntity>(CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task<List<TEntity>> ToListAsync<TEntity>(Expression<Func<TEntity, bool>> predicate,  
        CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task InsertAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task InsertAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default)  
        where TEntity : class, IEntity;  
  
    Task UpdateAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task UpdateAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default)  
        where TEntity : class, IEntity;  
  
    Task DeleteAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task DeleteAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default)  
        where TEntity : class, IEntity;  
  
    Task<int> CountAsync<TEntity>(Expression<Func<TEntity, bool>> predicate,  
        CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task<TEntity?> SingleOrDefaultAsync<TEntity>(Expression<Func<TEntity, bool>> predicate,  
        CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task<TEntity?> FirstOrDefaultAsync<TEntity>(Expression<Func<TEntity, bool>> predicate,  
        CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task<bool> AnyAsync<TEntity>(Expression<Func<TEntity, bool>> predicate,  
        CancellationToken cancellationToken = default) where TEntity : class, IEntity;  
  
    Task<List<T>> SqlQueryAsync<T>(string sql, params object[] parameters) where T : class, IEntity;  
  
    IQueryable<TEntity> Query<TEntity>(Expression<Func<TEntity, bool>>? predicate = null) where TEntity : class, IEntity;  
  
    IQueryable<TEntity> QueryNoTracking<TEntity>(Expression<Func<TEntity, bool>>? predicate = null)  
        where TEntity : class, IEntity;  
  
    DatabaseFacade Database { get; }  
    Task BatchInsertAsync<TEntity>(IList<TEntity> entities) where TEntity : class, IEntity;  
    Task BatchUpdateAsync<TEntity>(IList<TEntity> entities) where TEntity : class, IEntity;  
    Task BatchDeleteAsync<TEntity>(Expression<Func<TEntity,bool>> predicate)  where TEntity : class, IEntity;  
}
```


#### 实现接口
```csharp
public class EFRepository : IRepository, ITransient  
{  
    private readonly PractiseForClydeDBContext _dbContext;  
    public EFRepository(PractiseForClydeDBContext dbContext)  
    {
	    _dbContext = dbContext;  
    }
    
    public ValueTask<TEntity?> GetByIdAsync<TEntity>(object id, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        return _dbContext.FindAsync<TEntity>(new[] { id }, cancellationToken);  
    }  
    
    public Task<List<TEntity>> GetAllAsync<TEntity>(CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        return _dbContext.Set<TEntity>().ToListAsync(cancellationToken);  
    }  
    
    public Task<List<TEntity>> ToListAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        return _dbContext.Set<TEntity>().Where(predicate).ToListAsync(cancellationToken);  
    }  
    
    public async Task InsertAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        await _dbContext.AddAsync(entity, cancellationToken).ConfigureAwait(false);  
        _dbContext.ShouldSaveChanges = true;  
    }  
    
    public async Task InsertAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        await _dbContext.AddRangeAsync(entities, cancellationToken).ConfigureAwait(false);  
        _dbContext.ShouldSaveChanges = true;  
    }  
    
    public Task UpdateAsync<TEntity>(TEntity entity, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        _dbContext.Update(entity);  
        _dbContext.ShouldSaveChanges = true;  
        return Task.CompletedTask;  
    }  
    
    public Task UpdateAllAsync<TEntity>(IEnumerable<TEntity> entities, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        _dbContext.UpdateRange(entities);  
        _dbContext.ShouldSaveChanges = true;  
        return Task.CompletedTask;  
    }  
    
    public Task DeleteAsync<TEntity>(TEntity entity,  
        CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        _dbContext.Remove(entity);  
        _dbContext.ShouldSaveChanges = true;  
        return Task.CompletedTask;  
    }  
    
    public Task DeleteAllAsync<TEntity>(IEnumerable<TEntity> entities,  
        CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        _dbContext.RemoveRange(entities);  
        _dbContext.ShouldSaveChanges = true;  
        return Task.CompletedTask;  
    }  
    
    public Task<int> CountAsync<TEntity>(Expression<Func<TEntity, bool>> predicate,  
        CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        return _dbContext.Set<TEntity>().CountAsync(predicate, cancellationToken);  
    }  
    
    public Task<TEntity?> SingleOrDefaultAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        return _dbContext.Set<TEntity>().SingleOrDefaultAsync(predicate, cancellationToken);  
    }  
    
    public Task<TEntity?> FirstOrDefaultAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        return _dbContext.Set<TEntity>().FirstOrDefaultAsync(predicate, cancellationToken);  
    }  
    
    public Task<bool> AnyAsync<TEntity>(Expression<Func<TEntity, bool>> predicate, CancellationToken cancellationToken = default) where TEntity : class, IEntity  
    {  
        return _dbContext.Set<TEntity>().AnyAsync(predicate, cancellationToken);  
    }  
    
    public async Task<List<TEntity>> SqlQueryAsync<TEntity>(string sql, params object[] parameters) where TEntity : class, IEntity  
    {  
        return await _dbContext.Set<TEntity>().FromSqlRaw(sql, parameters).ToListAsync();  
    }  
    
    public IQueryable<TEntity> Query<TEntity>(Expression<Func<TEntity, bool>>? predicate = null) where TEntity : class, IEntity  
    {  
        return predicate == null ? _dbContext.Set<TEntity>() : _dbContext.Set<TEntity>().Where(predicate);  
    }  
    
    public IQueryable<TEntity> QueryNoTracking<TEntity>(Expression<Func<TEntity, bool>>? predicate = null) where TEntity : class, IEntity  
    {  
        return predicate == null  
            ? _dbContext.Set<TEntity>().AsNoTracking()  
            : _dbContext.Set<TEntity>().AsNoTracking().Where(predicate);  
    }  
    
    public DatabaseFacade Database => _dbContext.Database;  
    public async Task BatchInsertAsync<TEntity>(IList<TEntity> entities) where TEntity : class, IEntity  
    {  
        await _dbContext.BulkInsertAsync<TEntity>(entities).ConfigureAwait(false);  
    }  
    
    public async Task BatchUpdateAsync<TEntity>(IList<TEntity> entities) where TEntity : class, IEntity  
    {  
        await _dbContext.BulkUpdateAsync<TEntity>(entities).ConfigureAwait(false);  
    }  
    
    public async Task BatchDeleteAsync<TEntity>(Expression<Func<TEntity, bool>> predicate) where TEntity : class, IEntity  
    {  
        await _dbContext.Set<TEntity>().Where(predicate).ExecuteDeleteAsync().ConfigureAwait(false);  
    }}
```