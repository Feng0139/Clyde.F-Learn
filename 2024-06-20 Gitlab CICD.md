# 工作开展
1. 了解IS部门守则
2. 下载安装 Docker、Rider、SourceTree 等工作软件
3. 观看IS 内部学习清单

# 学习内容
## CI/CD
自动化流程，开发人员持续迭代内容，工具自动编译测试打包，最终自动部署到平台。
在 `Gitlab` 中由 `.gitlab-ci.yml` 配置文件定义作业顺序、执行条件等。

### 流水线定义
通常一个软件需要经过 Build 、Test 和 Deploy 阶段。如此按步骤的执行作业。

### 基本yml文件定义
一般由 `stages` 起头，定义阶段流程，后续执行流水线时按顺序执行对应的 `Job`.
```
stages:
	build
	test
	deploy
```

随后紧跟阶段制定 `Job` 详细内容，如下：
```
build-job:
	stage: build
	script:
		- echo "Doing build."
		- echo "Done!"
```
- `build-job`: 作业名称
- `stage`: 指定此作业在流水线上的处理阶段
- `script`: 作业的执行命令..

> gitlab-ci.yml 关键字参考: [跳转](https://docs.gitlab.cn/jh/ci/yaml/index.htmlé)

### 比较有用的关键词
rules
```
build-job:
	rules:
		- if: xxx == "xxx_event"
		  when: manual
		  allow_failure: true
```

`when`: 控制流水线的下一步骤执行与否
- `manual`: 仅手动执行
- `on_success`: 直接执行
- `delayed`: 延迟一定时间执行作业
	- 配合使用 `start_in` 指定时长，默认秒单位，无单位需要单引号括中。
		- `start_in`: '5' 
		- `start_in`: 5 seconds
		- `start_in`: 30 minutes
		- `start_in`: 1 day
		- `start_in`: 1 week
- `never`: 无论前期流程如何，都不运行作业。在 `rules`部分中使用
- `always`: 无论前期流程如何，都运行作业

`allow_failure`: 定义作业未完成时，是否可以继续运行流水线
- `true` / `false`

### Runners 执行器
作业的执行容器, 有很多种类型的执行器，常用 `shell`、`docker` 和 `Kubernetes`.