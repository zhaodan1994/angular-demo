# DotNet 用例生成器说明

## 1. 项目定位

`GrapeCity.DataVisualization.Tests.Tools.TestCaseGenerator` 是一个面向 Data Visualization 测试工程的 JSON 转源码生成工具。

它的主要职责包括：

- 读取测试用例 JSON 和配置文件
- 将 JSON 转换为内存中的测试用例模型
- 从图表运行时程序集解析 Data Visualization Option 的真实类型
- 生成测试用例源码和测试数据源码
- 将生成过程中的错误写入 `log.txt`

从当前代码实现来看，这个工具的核心仍然是 C# 源码生成。虽然它支持按 `DotNet`、`Java`、`JS`、`NuGet` 等目标做分支处理，但最终构建源码单元时仍然是基于 Roslyn 语法树，并输出 `.cs` 文件。

## 2. 使用技术

这个工程在代码中直接使用了以下技术：

- `C#` 与 `.NET 5`（`netcoreapp5.0`）
- `Microsoft.Extensions.CommandLineUtils`
  - 用于构建命令行入口并解析 `--config`
- `Microsoft.Extensions.Logging` 与 `Microsoft.Extensions.Logging.Console`
  - 用于控制台日志和异常输出
- `System.Text.Json`
  - 用于配置解析和测试用例 JSON 解析
- `Microsoft.CodeAnalysis` 与 `Microsoft.CodeAnalysis.CSharp`
  - 即 Roslyn，用于 AST 构建和源码格式化
- `.NET Reflection`
  - 用于反射 `GrapeCity.DataVisualization.Chart.dll`，动态推断 Option/Property 类型
- 外部 DV 运行时程序集
  - 通过 `pathconfig.json` 加载
  - 默认路径指向 `../../RefDLL/NetCore/GrapeCity.DataVisualization.Chart.dll`

从实现方式上看，这个项目还体现了以下设计和技术特点：

- Builder 模式
- 基于模型的代码生成
- 递归目录扫描
- 基于反射的类型推断

## 3. 运行入口

命令行入口位于：

- `Program.cs`
- `TestCaseGeneratorApplication.cs`

整体执行流程如下：

1. `Program.Main` 输出启动信息并创建应用对象
2. `TestCaseGeneratorApplication` 解析 `-c|--config`
3. 读取配置文件并转换为 `ConfigOption`
4. 如果目标是 `DotNet`，先生成数据源码文件
5. 扫描测试用例 JSON，并构建 `ITestCaseModel`
6. 按目标类型过滤测试用例
7. 将模型转换为 Roslyn `CompilationUnitSyntax`
8. 清理目标目录、写入生成文件，并输出 `log.txt`

## 4. 输入与输出

### 配置输入

核心配置模型包含：

- `sourcePath`
- `target.path`
- `target.for`

配置中的相对路径会通过 `ConfigOptionBuilder` 和 `OptionBuilder<T>` 解析为绝对路径。

该项目还依赖：

- `pathconfig.json`

这个文件提供 `DvAssemblyBuilder` 所需的程序集路径。

### 测试用例 JSON 输入

从当前实现看，生成器主要会读取这些字段：

- `title`
- `description`
- `category`
- `features`
- `testTarget`
- `model.size`
- `model.option`
- `model.actions`
- `model.actions_canvas`
- `model.actions_purecanvas`
- `model.runtimeApis.beforeLoad`
- `model.runtimeApis.afterLoad`
- `model.exceptionInfo`

### 输出结果

工具会输出：

- 生成后的测试用例源码文件
- 生成后的测试数据源码文件
- 记录生成错误的 `log.txt`

输出路径由 JSON 源路径和目标基础路径共同推导得到。

## 5. 核心模块

### 入口与调度

- `Program.cs`
  - 应用入口、异常捕获、控制台日志输出
- `TestCaseGeneratorApplication.cs`
  - 命令行解析、配置加载、生成流程编排、目标目录清理

### 配置与模型构建

- `ConfigOptionBuilder.cs`
  - 构建配置对象并规范化相对路径
- `OptionBuilder.cs`
  - 基于反射的通用 JSON 到对象映射器
- `TestCaseModelCollectionBuilder.cs`
  - 递归扫描目录，收集 JSON 测试用例
- `TestCaseModelBuilder.cs`
  - 分发到具体 `ITestCaseModelBuilder` 实现
- `TestCaseModelBuilderFromJsonModel.cs`
  - 将 JSON 转换为 `TestCaseModel`

### DV Option 类型解析

- `DvAssemblyBuilder.cs`
  - 从 `pathconfig.json` 加载 DV 程序集
- `DvPropertyModelBuilder.cs`
  - 使用反射解析 option 类、接口、枚举、集合元素类型以及具体实现类型

这部分是项目里非常关键的技术能力。它让生成器能够把普通 JSON option 负载转换成强类型的 DV Option 对象创建表达式。

### 源码生成与写入

- `TestCaseSourceFIleModelCollectionBuilder.cs`
  - 按输出路径分组模型，并构建 Roslyn 语法树
- `TestCaseSourceFileWriter.cs`
  - 负责格式化语法树并写入生成文件

## 6. 源码生成方式

源码生成主链路如下：

1. 将 JSON 解析成 `TestCaseModel`
2. 解析测试目标和输出路径
3. 从 DV 程序集中解析 Option 属性类型
4. 用 Roslyn 构建 C# AST 节点
5. 通过 `NormalizeWhitespace` 做格式化
6. 输出最终 `.cs` 文件

生成过程中的几个关键点：

- 使用了 `CompilationUnitSyntax`、`NamespaceDeclarationSyntax`、`ClassDeclarationSyntax`、`MethodDeclarationSyntax`
- 会为 `TestCaseModel`、Runtime API、Action、DV Option 构建对象初始化表达式
- 支持处理多态 DV 类型，例如：
  - transform option
  - overlay option
  - color option
  - value encoding option
  - filter encoding option
  - plot config option
  - plugin argument option
- 不能正确生成的属性会记录到 `_errorDetails`，最后写入 `log.txt`

## 7. 数据文件生成

当 `target.for == "DotNet"` 时，工具还会从 `cases/TestResources` 生成数据源码文件。

流程如下：

1. 从资源目录下的 JSON 构建数据模型
2. 将数据模型转换为源码文件模型
3. 删除原有生成数据目录
4. 重新写入数据源码文件

这说明该生成器不仅会生成测试方法，也会生成强类型的测试数据访问代码。

## 8. 运行命令

当前项目 README 中给出的示例如下：

```bash
dotnet run --config ../../test-case-for-dotnet/GrapeCity.DataVisualization.Tests.TestCasesForDotNet/ucgconfig/ucgconfig.json
dotnet run --config ../../test-case-for-dotnet/GrapeCity.DataVisualization.Tests.TestCasesForDotNet/ucgconfig/ucgconfig_NuGet.json
dotnet run --config ../../test-case-for-java/GrapeCity.DataVisualization.Tests.TestCasesForJava/ucgconfig/ucgconfig.json
dotnet run --config ../../test-case-for-java/GrapeCity.DataVisualization.Tests.TestCasesForJava/ucgconfig/ucgconfig_Maven.json
```

## 9. 维护注意事项

- 项目强依赖 `pathconfig.json` 中配置的 DV 程序集路径
- 生成前会删除目标输出目录，因此配置路径必须准确
- 生成逻辑依赖 DV Option 类型命名规则和反射解析结果
- 当前 `Java` 目标分支仍然是通过 Roslyn/C# 语法树生成，而不是原生 Java AST 生成
