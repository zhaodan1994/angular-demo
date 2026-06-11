# Java 用例生成器说明

## 1. 项目定位

`java-case-generator` 是一个面向 Data Visualization Java 测试工程的 JSON 转 Java 源码生成工具。

它的主要职责包括：

- 读取生成配置和测试用例 JSON
- 构建 Java 侧测试用例模型
- 解析 DV Option 的具体实现类型
- 生成 Java 测试用例源码
- 生成 Java 测试数据源码
- 将转换失败或生成错误写入 `log.txt`

这个项目不是简单的字符串模板脚本，而是一个完整的 Java 代码生成器。

## 2. 使用技术

这个工程在代码中直接使用了以下技术：

- `Java 8`
- `Maven`
  - 用于构建、依赖管理和 Jar 打包
- `picocli 4.6.1`
  - 用于解析命令行参数 `--config`
- `Gson 2.9.0`
  - 用于 JSON 解析和 JSON 到模型对象的转换
- `Log4j2 2.17.1`
  - 用于控制台日志输出
- `JavaParser 3.24.0`
  - 用于 Java AST 构建和源码打印
- `Java Reflection`
  - 用于反射 DV Option 类型以及 setter 参数类型
- `JarURLConnection` 与 `JarFile`
  - 被 `OptionTypes` 用于扫描 `flexdv.jar` 中的类
- 外部 DV 运行时 Jar
  - `lib/flexdv.jar`
  - 在 `pom.xml` 中以 `system scope` 方式声明

从实现风格上看，这个项目还使用了：

- Builder 模式
- 基于 AST 的代码生成
- 递归文件扫描
- 动态接口到实现类映射
- 基于正则的生成后源码格式修正

## 3. 构建与入口

运行入口位于：

- `src/main/java/.../Program.java`
- `src/main/java/.../TestCaseGeneratorApplication.java`

`pom.xml` 中的关键构建配置包括：

- Java 编译级别：`1.8`
- 主类：`com.grapecity.datavisualization.tests.tools.casegenerator.Program`
- 打包时会把依赖复制到 `target/lib`
- Jar 清单会将 `lib/` 作为运行时 classpath

整体执行流程如下：

1. `Program.main` 创建应用对象，并交给 `picocli` 执行
2. `TestCaseGeneratorApplication` 解析 `--config`
3. `ConfigOptionBuilder` 读取配置并规范化路径
4. 如果目标是 `Java`，先生成数据源码文件
5. 扫描 JSON 测试用例并构建 `ITestCaseModel`
6. 按目标类型过滤测试用例
7. 使用 JavaParser 构建 AST，并写入 `.java` 文件
8. 将转换问题写入 `log.txt`

## 4. 输入与输出

### 配置输入

主要配置结构为：

- `caseSourcePath`
- `dataSourcePath`
- `target.casePath`
- `target.dataPath`
- `target.targetFor`

其中相对路径会通过 `ConfigOptionBuilder` 调用 `FileUtil.getFullPath(...)` 转成绝对路径。

### 测试用例 JSON 输入

从当前代码实现来看，生成器主要读取这些字段：

- `title`
- `description`
- `category`
- `features`
- `testTarget`
- `model.size`
- `model.option`
- `model.runtimeApis.beforeLoad`
- `model.runtimeApis.afterLoad`
- `model.default` 或与异常相关的负载

### 输出结果

工具会生成：

- 输出到 `casePath` 的 Java 测试用例类
- 输出到 `dataPath` 的 Java 数据构建类
- 工作目录下的 `log.txt`

## 5. 核心模块

### 入口与调度

- `Program.java`
  - 输出启动信息、启动 picocli、处理异常
- `TestCaseGeneratorApplication.java`
  - 负责总流程编排、清理规则、目标过滤和日志写入

### 配置与模型构建

- `ConfigOptionBuilder.java`
  - 将 JSON 配置转换为 `ConfigOption`，并展开成完整路径
- `TestCaseModelCollectionBuilder.java`
  - 递归扫描 JSON 文件
- `TestCaseModelBuilder.java`
  - 将 JSON 转换为 `TestCaseModel`
- `TestCaseModelBuilderContext.java`
  - 在构建过程中保存源路径和目标路径上下文

### DV Option 类型解析

- `DvPropertyModelBuilder.java`
  - 利用反射解析 option setter、参数类型和属性类型
- `OptionTypes.java`
  - 扫描 DV Jar 中的 `IOption` 接口和它们的实现类，并建立接口到实现类的映射关系

这部分是该项目里最关键的技术能力之一。它使生成器能够把通用 JSON option 负载转换为具体的 Java DV Option 创建代码。

### AST 生成与源码写入

- `TestCaseSourceFileModelCollectionBuilder.java`
  - 构建 JavaParser `CompilationUnit`、类、方法和表达式
- `SourceFileWriter.java`
  - 将 AST 输出为源码，并在写入前做正则级别的格式修正

## 6. Java 代码生成方式

测试用例生成主链路如下：

1. 扫描 JSON 文件
2. 将 JSON 转换为 `ITestCaseModel`
3. 按输出源码路径对用例分组
4. 构建 JavaParser AST
5. 将 AST 打印为 Java 源码
6. 对生成文本做格式后处理
7. 写出最终 `.java` 文件

生成过程中的几个关键特征：

- 生成的类继承自 `SvgRenderEngine`
- 每条测试用例会生成一个 `public` 方法
- 每个方法内部都会创建 `TestCaseModel model`
- 方法末尾会调用 `this.renderCase(model)`
- import 是通过代码动态组装的
- 类似 `option.data` 的数据引用会被转换成 `com.dv.data...` 下的 builder 风格方法调用

## 7. 数据文件生成

当 `target.targetFor == "Java"` 时，生成器还会额外生成 Java 数据源码文件。

流程如下：

1. 从配置的 `dataSourcePath` 读取 JSON
2. 将 JSON 转换为数据模型
3. 构建源码文件模型
4. 清理目标数据目录，但保留如 `additionaldata` 之类的保留目录
5. 写入生成后的 Java 数据文件

这说明该项目不仅会生成测试用例代码，也会生成可复用的数据访问/构建代码。

## 8. 关键技术设计

### 8.1 基于反射的类型推断

`DvPropertyModelBuilder` 会通过 setter 方法解析出：

- 属性类型
- setter 方法名
- 属性是否为集合
- 枚举类型
- 接口属性对应的具体实现类

### 8.2 基于 Jar 扫描的 Option 实现类映射

`OptionTypes` 会扫描 `flexdv.jar` 中的类，并构建运行时查找表：

- key：option 接口
- value：具体实现类

这样可以减少手工硬编码接口映射的工作量。

### 8.3 AST 与文本后处理结合

虽然项目使用 JavaParser 生成 Java AST，但 `SourceFileWriter` 仍然会再做一层正则替换，用来修正 import 间距、换行以及部分初始化代码格式。因此最终输出并不只依赖 AST，也依赖后续的字符串后处理。

## 9. 运行命令

当前 README 中给出的示例如下：

```bash
mvn clean package
java -jar ./target/tools-1.0-SNAPSHOT.jar --config "../../javaProject/GrapeCity.DataVisualization.Tests.TestCasesForJava/ucgconfig.json"
java -Xmx1024m -Xms1024m -jar ./target/tools-1.0-SNAPSHOT.jar --config "../../javaProject/GrapeCity.DataVisualization.Tests.TestCasesForJava/ucgconfig.json"
java -jar ./target/tools-1.0-SNAPSHOT.jar --config "../../javaProject/GrapeCity.DataVisualization.Tests.TestCasesForJava/ucgconfig_Maven.json"
```

## 10. 维护注意事项

- 项目依赖本地存在的 `lib/flexdv.jar`
- `pom.xml` 使用了 `systemPath`，因此依赖解析是基于本地文件而不是仓库坐标
- 生成前会主动清理输出目录，所以配置路径必须谨慎确认
- `containsTarget(...)` 目前显式处理了 `Java` 和 `NuGet`
- 最终源码格式部分受 `SourceFileWriter` 中的正则替换影响，改动后建议实际生成验证
