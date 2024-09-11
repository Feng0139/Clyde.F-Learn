# NuGet Package
## 创建 Package 步骤
1. 准备好C#项目文件，确保能够正常编译。然后在 `.csproj` 文件中配置包信息。
2. 修改 `.csproj` 文件

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    ...

	<!-- 新增内容 -->
    <Version>1.0.0</Version>
    <Authors>Your Name</Authors>
    <Description>Package Description</Description>
    <PackageTags>tag1;tag2</PackageTags>
    <PackageLicenseExpression>MIT</PackageLicenseExpression>
    <RepositoryUrl>https://github.com/your-repo</RepositoryUrl>
    
  </PropertyGroup>

</Project>
```

- `Version`: 版本信息，可以添加后缀标识 `-alpha` 、`-beta`、 `-rc`来表示包的不同阶段。
- `Authors`:  作者。
- `Description`: 包的描述内容。
- `PackageTags`: 关键词、属性标签，有助于搜索和理解包的作用。
- `PackageLicenseExpression`: 当前包的许可证类型。
- `RepositoryUrl`: 项目存储库网址。

3. 生成包
使用终端，切换到项目根目录下执行此命令来生成对应包：

```
dotnet pack
```

这将生成 `.nupkg` 文件，通常在 `bin/Debug` 或 `bin/Release` 目录下。

## 发布到托管源
如果要将包发布到 `NuGet.org`源，需要一个 `API` 密钥，可以在 `NuGet.org`的账户设置中生成一个。

然后使用此命令进行发布:

```
dotnet nuget push MyPackage.nupkg -k ApiKey -s https://api.nuget.org/v3/index.json
```

- `-k`: 填写对应的密钥。
- `-s`: 对应包管理源的发布地址。如若不指定将默认推送到配置中设置的源。

以下是其它托管源
- `Github`

```
https://nuget.pkg.github.com/your-username/index.json
```

- `Azure`

```
https://pkgs.dev.azure.com/your-organization/packaging/your-feed-name/nuget/v3/index.json
```


# 自建托管源
## BaGet
1. 拉取最新BaGet镜像

```
docker pull loicsharma/baget
```

2. 运行BaGet容器
```
docker run -d \
  --name baget \
  -e "Baget__Database__Type=Sqlite" \
  -e "Baget__Search__Type=Database" \
  -p 5000:80 \
  loicsharma/baget:latest
```

3. 访问网站浏览

托管网址: [http://localhost:5000](http://localhost:5000/)

上传指南: [http://localhost:5000/upload](http://localhost:5000/upload)