# 学习内容
## ReSpawner的Tips
Respawner 是一个数据库工具，用于自动重新创建和初始化数据表或其他数据库对象，以确保数据库在部署环境中的一致性。

这里主要是记录使用上的注意点：

开发经常执行测试方法，为了方便开发，直接使用数据库的root账号进行测试，root账号拥有所有库的访问权。Respawner 在其中部分作用是清理测试数据库表，获取表逻辑是按账号可访问数据库的范围，这就导致了会清理到其它不相关的数据库表数据。

So. 这里提供两个解决方法：

1. 数据库连接所使用的账号权限进行限制，不要授权未涉及的数据库权限。
2. 代码层面创建 Respawner 对象时传递Option，其中 SchemasToInclude 属性添加需要操纵的数据库名称即可。