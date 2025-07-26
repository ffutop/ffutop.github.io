---
title: "告别 Spring Boot 慢启动：GraalVM 原生镜像迁移实践"
date: 2025-07-11
author: ffutop
tags: 
  - Spring Boot
  - Spring
  - GraalVM
  - Native Image
  - Java
  - Cloud Native
  - Migration
categories: 
  - 案例分析
---

随着云原生和 Serverless 架构的兴起，应用的启动速度、内存占用和打包体积变得至关重要。Spring Boot 3.x 结合 GraalVM Native Image 技术，为 Java 应用提供了“原生编译”这一颠覆性选项，能够创造出启动如闪电、资源消耗极低的轻量级可执行文件。本文将作为一份详细的案例分析与经验总结，系统性地阐述一个典型的 Spring Boot 2.7.2 应用，如何一步步迁移到 Spring Boot 3.x，并最终打包为高性能的 GraalVM 原生可执行文件。我们将深入探讨迁移过程中的关键变更、常见陷阱、解决方案以及最终的性能收益。

如需了解 GraalVM 相关知识，可参考 [剖析 GraalVM Native Image 技术](./2025-07-11-GraalVM-native-image)。

*声明：本文为了更加聚焦地介绍 GraalVM 原生镜像迁移实践，提供的源码与脚本并不严格遵循最佳工程实践，但会更利于阅读*

## 迁移实践

### Java 版本升级

顽固的 JDK 8 已经在公司生存了近 10 年，首先需要升级到支持的 Native Image 的 JDK 版本。

这里选择 JDK 21，使用 [GraalVM Community](https://sdkman.io/jdks/graalce) 分发版本。

```bash
# 安装 SDKMAN
curl -s "https://get.sdkman.io" | bash

# 安装 GraalVM Community 21.0.2
sdk install java 21.0.2-graalce
```

### Spring Boot 版本升级

简单起见，直接将依赖的父 POM 升级到 Spring Boot 3.5.3

```diff
<project>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
+		<version>3.5.3</version>
-		<version>2.7.2</version>
		<relativePath/>
	</parent>
</project>
```

同时为了编译 Native Image，需要在 POM 中添加 GraalVM Native Image 插件。

```xml
<project>
  <build>
    <plugins>
      <!-- 引入 Spring Boot Maven 插件 -->
      <!-- 当声明 mvn -Pnative 时，会引入由 父 POM 
        spring-boot-starter-parent 管理的 
        pluginManagement 声明 -->
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>3.5.3</version>
      </plugin>
      <!-- 引入 Graalvm Native Maven 插件-->
      <!-- 当声明 mvn -Pnative 时，会引入由 父 POM 
        spring-boot-starter-parent 管理的 
        pluginManagement 声明 -->
      <!-- 但仍需主动声明在 package 阶段挂载上 compile-no-fork 目标 -->
      <plugin>
        <groupId>org.graalvm.buildtools</groupId>
        <artifactId>native-maven-plugin</artifactId>
        <version>0.10.6</version>
        <executions>
          <execution>
            <id>build-native</id>
            <goals>
              <goal>compile-no-fork</goal>
            </goals>
            <phase>package</phase>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

由于 Java 版本和 Spring Boot 大版本升级，需要同步进行一些不兼容的变更。包括：

1. 依赖的 Spring Framework 需要全面升级到 6.x 版本（如有在原 `pom.xml` 主动声明了低版本号的，下同）

2. 依赖的 Spring Security 升级到 6.x 版本，移除了 `antMatcher` 改用 `securityMatcher`。

3. 因 Java 版本升级，Java EE 也需升级到 Jakarta EE 版本。相关的 `javax.*` 代码都需要调整为 `jakarta.*`。

4. 依赖的三方库，也需升级到与 Spring Boot 3.x 适配的版本以上。可参考 Spring Boot 托管依赖项 [最新版本的托管依赖项（请注意时效性）](https://docs.spring.io/spring-boot/appendix/dependency-versions/coordinates.html) 或 [Spring Boot 3.0.x 的托管依赖性](https://docs.spring.io/spring-boot/docs/3.0.x/reference/html/dependency-versions.html#appendix.dependency-versions)

另有更多的变更，请移步 [Spring Boot 3.0 Migration Guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)

### Spring Security 变更

Spring Security 6.x 版本的变更可参考 [Migrating to 6.0](https://docs.spring.io/spring-security/reference/6.0/migration/index.html)。

因此前已经紧跟版本，我们直接受到的影响，只是移除 `antMatcher` 改用 `securityMatcher`。

```diff
@Bean
public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity httpSecurity) throws Exception {
    httpSecurity
-            .antMatcher("/actuator/**")
+            .securityMatcher("/actuator/**")
            .authenticationProvider(new AnonymousAuthenticationProvider("ANONYMOUS"))
            .authorizeHttpRequests((requests) -> requests.anyRequest().permitAll());
    return httpSecurity.build();
```

### JDBC Driver & HikariCP 变更

由于原生镜像需要静态编译，原 `PostgreSQL JDBC Driver` 需要由 `runtime` 改为默认的 `compile` 依赖。否则因编译期不存在 Drvier，最终原生镜像将不会引入相关代码。

```diff
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
- <scope>runtime</scope>
</dependency>
```

同时，虽然公共的元数据仓库提供了关于 HikariCP 的元数据信息。但是，因配置方式不同（例如下列配置），完全借助 Spring Boot 的机制将无法引入 `HikariConfig`。 

```java
@Bean("demoDataSource")
@ConfigurationProperties(prefix = "demo.datasource")
public DataSource demoDataSource() {
    return DataSourceBuilder.create().build();
}
```

为了解决问题，我们需要手动引入 `HikariConfig`。本示例借助 Spring AOT 机制以编码方式主动声明了 `HikariConfig` 反射元数据信息。

```java
@Configuration
@MapperScan(value = {"com.ffutop.demo.mapper"}, sqlSessionFactoryRef = "demoSqlSessionFactory")
@ImportRuntimeHints(DataSourceConfiguration.DataSourceConfigurationRuntimeHints.class)
public class DataSourceConfiguration {

  static class DataSourceConfigurationRuntimeHints implements RuntimeHintsRegistrar {

    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
      hints.reflection().registerType(com.zaxxer.hikari.HikariConfig.class, MemberCategory.values());
    }
  }
}
```

### MyBatis Spring Boot Starter 升级

MyBatis Spring Boot Starter 需要从 2.x 升级到 3.x，来适应 Spring Boot 的升级。

但为适应 GraalVM 原生镜像的编译，还需要额外添加 AOT 功能。

[MyBatis Spring Native](https://github.com/mybatis/spring-native) 期望提供开箱可用的 AOT 功能，但该项目似乎已经陷入停滞，几无更新提交。

而 MyBatis Spring Boot Starter 提供了一份 [Wiki](https://github.com/mybatis/spring-boot-starter/wiki/Quick-Start-for-building-native-image) 说明如何手动创建 [MyBatisNativeConfiguration](https://github.com/mybatis/spring-boot-starter/wiki/MyBatisNativeConfiguration.java) 支持 Native Image 编译。不过这份 Java 代码似乎仍有[缺陷](https://github.com/baomidou/mybatis-plus/issues/5826)，还需进一步增补对元数据的注册方可解决。

如果编译运行原生可执行文件得到下列错误，请移步并拷贝 [MyBatisNativeConfiguration BugFree Version](#MyBatisNativeConfiguration.java)


```plain
Application run failed
java.lang.ExceptionInInitializerError
        at org.mybatis.spring.mapper.MapperScannerConfigurer.postProcessBeanDefinitionRegistry(MapperScannerConfigurer.java:387)
        at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors(PostProcessorRegistrationDelegate.java:349)
        at org.springframework.context.support.PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(PostProcessorRegistrationDelegate.java:148)
        at org.springframework.context.support.AbstractApplicationContext.invokeBeanFactoryPostProcessors(AbstractApplicationContext.java:791)
        at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:609)
        at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:146)
        at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:752)
        at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:439)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:318)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1361)
        at org.springframework.boot.SpringApplication.run(SpringApplication.java:1350)
        at com.ffutop.demo.DemoServiceApplication.main(DemoServiceApplication.java:10)
        at java.base@21.0.2/java.lang.invoke.LambdaForm$DMH/sa346b79c.invokeStaticInit(LambdaForm$DMH)
Caused by: org.apache.ibatis.logging.LogException: Error creating logger for logger org.mybatis.spring.mapper.ClassPathMapperScanner.  Cause: java.lang.NullPointerException
        at org.apache.ibatis.logging.LogFactory.getLog(LogFactory.java:56)
        at org.apache.ibatis.logging.LogFactory.getLog(LogFactory.java:49)
        at org.mybatis.logging.LoggerFactory.getLogger(LoggerFactory.java:32)
        at org.mybatis.spring.mapper.ClassPathMapperScanner.<clinit>(ClassPathMapperScanner.java:65)
        ... 13 more
Caused by: java.lang.NullPointerException
        at org.apache.ibatis.logging.LogFactory.getLog(LogFactory.java:54)
        ... 16 more
```

### Flyway 跨版本升级

| Spring Boot | Flyway |
| --- | --- |
| 2.7.2 | 8.5.13 |
| 3.5.3 | 11.7.2 |

为与 Spring Boot 3.x 适配，需要将 Flyway 版本升级到 11.7.2 。

在跨大版本升级过程中，[Flyway 10](https://documentation.red-gate.com/fd/release-notes-for-flyway-engine-179732572.html#ReleaseNotesforFlywayEngine-Flyway10.0.0(2023-10-31)) 引入了一个重大不兼容变更——将数据库支持模块化，移出了 Flyway 核心库。

> Modularized database support in Flyway to allow greater flexibility. This includes; DB2, Derby, HSQLDB, Informix, PostgreSQL, CockroachDB, Redshift, SAP HANA, Snowflake and Sybase ASE. See Database Support page for your database for module dependency. If you are including Flyway in your project, either as a dependency or via the maven and gradle plugins please include the respective database module in your project configuration.

因此需要额外增加一个依赖（我们使用的 postgresql，其他数据库类似）

```xml
<dependency>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

如果未声明该依赖，你将收获如下错误：

```plain
Caused by: org.flywaydb.core.api.FlywayException: Unsupported Database: PostgreSQL 15.7
  at org.flywaydb.core.internal.database.DatabaseTypeRegister.lambda$getDatabaseTypeForConnection$7(DatabaseTypeRegister.java:134)
  at java.base/java.util.Optional.orElseThrow(Optional.java:403)
  // ... something omitted ...
```

## 构建、测试与运行

### 原生编译

在迁移过程中，我们已经将 `spring-boot:process-aot`, `native:add-reachability-metadata`, `native:compile-no-fork` 等插件添加到了项目中。直接在命令行执行 `mvn -Pnative package` 或借助 IDE 在启用 `native` Profile 的情况下执行 package 阶段即可完成编译。

这里推荐借助 `native` Profile 来区别是否进行原生编译，在禁用 `native` Profile 的情况下执行 package 阶段，可以很方便地得到传统的 JAR 包。

编译过程的八个阶段可以参考 [Native Image Build Output](https://www.graalvm.org/latest/reference-manual/native-image/overview/BuildOutput/)。由于是原生编译，整个过程相较于传统 JAR 构建，会花费非常长的额外的时间，一般以分钟记。对于较大的工程，甚至可能需要十几分钟的编译过程。


```plain
GraalVM Native Image: Generating 'demo-service' (executable)...
========================================================================================================================
[1/8] Initializing...                                                                                    (8.6s @ 0.23GB)
 Java version: 21.0.2+13, vendor version: GraalVM CE 21.0.2+13.1
 Graal compiler: optimization level: 2, target machine: x86-64-v3
 C compiler: cc (apple, x86_64, 17.0.0)
 Garbage collector: Serial GC (max heap size: 80% of RAM)
 2 user-specific feature(s):
 - com.oracle.svm.thirdparty.gson.GsonFeature
 - org.springframework.aot.nativex.feature.PreComputeFieldFeature
------------------------------------------------------------------------------------------------------------------------
Build resources:
 - 12.09GB of memory (75.6% of 16.00GB system memory, determined at start)
 - 16 thread(s) (100.0% of 16 available processor(s), determined at start)
SLF4J(W): No SLF4J providers were found.
SLF4J(W): Defaulting to no-operation (NOP) logger implementation
SLF4J(W): See https://www.slf4j.org/codes.html#noProviders for further details.
[2/8] Performing analysis...  [******]                                                                  (47.9s @ 3.05GB)
   22,523 reachable types   (89.5% of   25,178 total)
   33,912 reachable fields  (63.2% of   53,645 total)
  106,067 reachable methods (61.9% of  171,328 total)
    7,129 types, 1,403 fields, and 10,435 methods registered for reflection
       67 types,    67 fields, and    58 methods registered for JNI access
        5 native libraries: -framework CoreServices, -framework Foundation, dl, pthread, z
[3/8] Building universe...                                                                               (7.5s @ 2.25GB)
[4/8] Parsing methods...      [**]                                                                       (4.4s @ 3.13GB)
[5/8] Inlining methods...     [***]                                                                      (2.6s @ 2.70GB)
[6/8] Compiling methods...    [******]                                                                  (40.3s @ 3.66GB)
[7/8] Layouting methods...    [***]                                                                      (8.7s @ 2.36GB)
[8/8] Creating image...       [***]                                                                      (9.7s @ 3.66GB)
  51.55MB (51.34%) for code area:    69,320 compilation units
  48.47MB (48.28%) for image heap:  489,517 objects and 395 resources
 391.84kB ( 0.38%) for other data
 100.40MB in total
------------------------------------------------------------------------------------------------------------------------
                      13.5s (10.2% of total time) in 123 GCs | Peak RSS: 6.38GB | CPU load: 10.22

```

### 原生测试

由于原生镜像是基于“封闭世界”假设进行静态编译的，这与动态的 JVM 环境有本质区别。因此，仅在 JVM 上通过所有测试，并不足以完全保证应用在原生环境中的行为正确性。

原生测试 (Native Test) 是将测试代码与应用代码共同编译成一个临时的原生可执行文件并运行它。这是验证应用原生兼容性、确保迁移质量的关键步骤。

**使用追踪代理生成元数据**

在正式进行原生测试编译前，强烈建议先在 JVM 环境下，利用 GraalVM 提供的追踪代理 (Tracing Agent) 来运行您所有的单元测试和集成测试。

这个代理能自动捕获测试过程中发生的所有反射、JNI、资源加载等动态调用，并生成相应的可达性元数据（JSON 配置文件）。这是发现并补充缺失的 `RuntimeHints` 的最高效方法，特别是对于那些仅在测试代码中被覆盖的逻辑路径。

操作命令：

```bash
# -Dagent 会激活追踪代理，并自动捕获动态调用
mvn -Pnative test -Dagent
```

代理会将生成的元数据文件输出到 `target/native/agent-output/` 目录。您应仔细审查这些文件，并将其合并到 `src/main/resources/META-INF/native-image/` 目录下，以便在最终的原生编译中生效。

**执行原生测试**

当元数据准备就绪后，就可以执行真正的原生测试了。`native-maven-plugin` 插件已将原生测试的执行默认绑定到了 Maven 的 `test` 生命周期中。

```bash
# 激活 nativeTest profile 并执行 test 生命周期
mvn -PnativeTest test
```

此命令会触发一个完整的原生编译流程，但其目标不是生成应用可执行文件，而是生成并运行一个测试可执行文件。如果所有测试都通过，意味着应用的核心功能在原生环境中行为符合预期。如果失败，则需要根据错误信息（通常是 `MissingReflectionException` 等）回头补充 `RuntimeHints` 或检查代码兼容性。

### 原生运行

原生编译成功后，您会在 `target` 目录下找到一个与您项目名和操作系统相匹配的可执行文件（例如，在 Linux/macOS 上名为 `demo-service`）。

**直接运行**

与传统的 `java -jar` 命令不同，可以直接在命令行中执行这个文件：

```bash
./target/demo-service
```

你会立刻注意到最直观的变化：**启动速度**。应用几乎在瞬间就完成了启动并准备好接收请求，这与 JVM 应用需要数秒甚至更长时间的预热过程形成了鲜明对比。

**容器化部署**

原生可执行文件的另一大优势是其极小的体积和自包含的特性，这使其成为容器化部署的理想选择。由于不再需要外部的 JRE/JDK，我们可以使用极度精简的基础镜像。

*注意：由于原生编译，跨平台、跨 CPU 架构将不被接受，请确保编译平台与运行平台的一致性*

以下是一个简单的 `Dockerfile` 示例：

```dockerfile
# 使用一个极度精简的基础镜像
FROM alpine:latest

# 将原生可执行文件复制到镜像中
COPY target/demo-service /demo-service

# 定义容器启动命令
ENTRYPOINT ["/demo-service"]
```

使用此 `Dockerfile` 构建的容器镜像，其体积通常只有百兆字节左右，远小于包含完整 JRE 的传统镜像（通常为数百兆字节）。这不仅节省了存储资源，还显著加快了镜像的推送、拉取和部署速度，完美契合了云原生对敏捷和高效的要求。

## 常见陷阱与经验总结

### 第三方库兼容性

GraalVM 原生编译的核心是“封闭世界”假设，即在编译时必须知晓所有可执行的代码。然而，许多 Java 库广泛使用反射、动态代理、资源加载等动态特性，这与静态分析的原则直接冲突。因此，确保第三方库的兼容性是迁移过程中最关键也最具挑战性的一环。

解决此问题的策略可分为三个层次，应按以下优先级顺序尝试：

1. 优先选用原生就绪的框架与库

    现代框架如 Spring Boot 3+、Quarkus 和 Micronaut 已经将原生编译作为核心特性。它们不仅自身完全兼容，还为生态内的大量常用库（如 Jackson, Netty, Tomcat）提供了预置的 AOT (Ahead-of-Time) 配置和 `RuntimeHints`。选择这类框架是通往原生编译最平坦的道路，因为框架已经替开发者解决了绝大部分兼容性问题。

2. 利用 GraalVM 可达性元数据仓库

    对于许多尚未完全原生就绪但广受欢迎的库，GraalVM 社区维护了一个可达性元数据仓库。`native-maven-plugin` 构建插件可以被配置为自动从该仓库拉取所需依赖的元数据。

    这意味着，即使一个库（例如，某个版本的 `google-gson`）的作者没有提供官方原生支持，只要社区为其贡献了元数据，它也能在你的项目中“开箱即用”。在 `pom.xml` 中启用此功能是强烈推荐的最佳实践：

    ```xml
    <plugin>
        <groupId>org.graalvm.buildtools</groupId>
        <artifactId>native-maven-plugin</artifactId>
        <executions>
            <execution>
                <id>add-reachability-metadata</id>
                <goals>
                    <goal>add-reachability-metadata</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
    ```

3. 手动提供元数据

    当一个库既非原生就绪，也无社区提供的元数据时，就需要我们手动介入。

    - 使用追踪代理 (Tracing Agent)：这是首选的半自动方法。通过在 JVM 模式下运行应用的测试（`mvn -Pnative test -Dagent`），GraalVM 的追踪代理能自动捕获大部分动态调用，并生成所需的元数据 JSON 文件。这是发现和补充缺失配置最高效的方式。当然，这也要求应用的测试用例覆盖度高，以确保追踪到的调用都是必要的。

    - 编写 `RuntimeHints`：对于追踪代理无法覆盖的复杂场景，或为了更精细的控制，可以利用 Spring 的 `RuntimeHintsRegistrar` API 以编程方式注册元数据。这在本文的 HikariCP 和 MyBatis 适配部分已有体现。

    - 直接编写 JSON 配置文件：作为最终手段，可以直接在 `src/main/resources/META-INF/native-image/` 目录下手动编写 `reflect-config.json` 等文件。这种方式最为灵活，但也最繁琐且易错。

总结而言，处理第三方库兼容性的正确路径是：框架适配 > 社区元数据 > 手动配置。盲目地为每个库手动编写 `RuntimeHints` 是低效的。应首先充分利用框架和社区生态的力量，仅在必要时才进行手动干预。

### 原生镜像的调试

转向原生镜像意味着告别了整个基于 JVMTI 和 JMX 的传统诊断工具生态（如 JProfiler, VisualVM, jstack）。这并非技术缺陷，而是架构选择的必然结果。开发者必须从依赖 JVM 提供的“全能管家”模式，转向拥抱原生工具链和自包含的可观测性体系。

**可观测性**

对于生产环境中的问题，依赖可观测性的“三大支柱”是非常推荐的方法。

- 日志 (Logging)：这是最基础的诊断手段。确保 Logback、Log4j2 等日志框架及其配置文件（如 `logback-spring.xml`）被正确地包含在可达性元数据中，是保证日志在原生环境中正常工作的前提。
- 指标 (Metrics)：通过集成 Micrometer，原生应用可以像任何云原生服务一样，向 Prometheus 等监控系统暴露关键性能指标。Actuator 端点在原生环境中同样有效。
- 分布式追踪 (Distributed Tracing)：在微服务架构下，使用 OpenTelemetry 等标准，可以追踪一个请求在多个原生服务间的完整调用链路。

**交互式调试 (GDB/LLDB)**

对于开发阶段的逻辑错误，可以使用 GDB (Linux) 或 LLDB (macOS) 等原生调试器进行交互式调试。

首先，需要让 `native-image` 在编译时包含调试信息。在 `pom.xml` 的 `native-maven-plugin` 配置中加入 `-g` 参数：

```xml
<plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <configuration>
        <buildArgs>
            <buildArg>-g</buildArg>
        </buildArgs>
    </configuration>
</plugin>
```
然后重新编译原生可执行文件。

**启动调试会话**

使用 GDB 启动调试：
```bash
gdb ./target/demo-service
```

进入 GDB 后，可以使用标准命令：
- `b com.ffutop.demo.service.MyService.myMethod`: 在指定方法设置断点。
- `r`: 运行程序。
- `p myVariable`: 打印变量值。
- `bt`: 查看当前线程的堆栈回溯 (Backtrace)。
- `c`: 继续执行直到下一个断点。

虽然不如 IDE 中的 Java 调试器直观，但这为定位原生代码中的复杂逻辑问题提供了最直接的手段。

### 构建时间显著增加

AOT 编译是一个资源密集型过程，构建时间会比传统打包长得多。讨论在 CI/CD 流水线中的应对策略。

从传统的 `mvn package` 到 `mvn -Pnative package`，最直观的感受之一就是构建时间的急剧增加。传统打包通常在数十秒内完成，而原生编译则可能需要数分钟甚至更长时间。这并非缺陷，而是将大量运行时工作（如类加载、JIT 编译、初始化）前置到构建时所付出的必要代价。理解其原因并制定合理的 CI/CD 策略至关重要。

**AOT 编译为何耗时？**

1. 全程序静态分析：`native-image` 工具必须遍历整个应用及其所有依赖的每一条可能执行路径，以确定哪些代码是“可达”的。这是一个计算密集型的大规模图遍历过程。
2. 重量级优化与编译：所有可达的 Java 字节码都需要被编译成高度优化的本地机器码，这个过程远比简单的字节码生成复杂。
3. 镜像堆生成：执行所有标记为“构建时初始化”的类的静态代码块，并将其内存状态快照固化到可执行文件中，也是一个耗时步骤。

**CI/CD 应对策略**

在持续集成和部署流水线中，必须采取策略来管理这额外的构建开销，以避免拖慢开发节奏。

1. 分阶段构建与测试
    - 快速反馈环 (JVM 测试)：在每次代码提交时，应仅运行标准的 JVM 测试 (`mvn test`)。这能快速发现大部分业务逻辑错误，提供及时的反馈。
    - 集成阶段 (原生测试与构建)：将资源密集型的原生编译 (`mvn -Pnative package`) 和原生测试 (`mvn -PnativeTest test`) 推迟到更晚的阶段，例如在合并到主分支 (`main`) 或创建发布分支 (`release`) 时才触发。

2. 增强构建环境 (Beef Up Build Agents)

    原生编译对 CPU 和内存有很高的要求。在 CI/CD 平台上，应为原生构建任务分配更强大的执行器（Runner/Agent）。推荐使用至少 **8 核 CPU 和 16GB 以上内存**的构建环境，以显著缩短编译时间。

3. 善用缓存机制
    - 依赖缓存：确保 CI/CD 流水线正确配置了对 Maven (`.m2` 目录) 或 Gradle 依赖的缓存，避免每次都重新下载。
    - Docker 层缓存：在构建容器镜像时，精心设计 `Dockerfile` 的层次结构。应先将已编译好的原生可执行文件复制到镜像中，再处理其他变动较小的配置，以最大限度地利用 Docker 的层缓存。

    ```dockerfile
    # Dockerfile for Native App
    FROM alpine:latest
    
    # 先复制可执行文件，这一层在代码不变时可以被缓存
    COPY target/demo-service /app/demo-service
    
    # 再复制其他可能变动的配置文件
    COPY src/main/resources/config /app/config
    
    WORKDIR /app
    ENTRYPOINT ["./demo-service"]
    ```

4. 分离本地开发与 CI 职责

    开发者在本地进行日常编码和调试时，应始终工作在 JVM 模式下，以享受快速的重启和反馈。原生编译的验证和打包工作应主要交由 CI/CD 流水线来完成。这确保了本地开发的高效率和最终交付物的一致性。


## 成果量化对比

为了直观地展示迁移带来的收益与损耗，我们对同一个基础的 CRUD Web 应用在迁移前后的关键指标进行了对比。

<center>

| 指标 | Spring Boot 2.7.2 on JVM (Java 8) | Spring Boot 3.x Native (GraalVM 21) | 提升幅度 |
| :--- | :--- | :--- | :--- |
| 应用启动时间 (ms) | 2,474 | 268 | ~89% |
| 应用空闲内存占用 (MB) | ~204 | ~105 | ~48% |
| 容器镜像大小 (MB) | ~300 (alpine with JRE) | ~100 (from alpine) | ~67% |
| 构建时间 (min) | 0.09 | 2.5 | ~28x |

</center>

*（注：以上数据为示例应用迁移前后的对比结果，仅供参考）*

## 结论与展望

从 Spring Boot 2.7 迁移到 3.x 并拥抱原生编译，无疑是一项成本高昂、收益显著的技术投资，尤其对于追求极致启动性能和资源效率的云原生应用而言。本次迁移实践清晰地展示了原生镜像在启动时间、内存占用和部署体积上的巨大优势。然而，这一过程也要求开发者进行一次深刻的思维范式转变：从依赖 JVM 灵活的“运行时动态”，转向拥抱“编译期静态”的确定性世界。

**权衡与适用场景**

原生镜像并非解决所有问题的银弹。它最适合微服务、Serverless 函数、CLI 工具等对快速启动和低资源占用有严苛要求的场景。在这些领域，原生化带来的优势是决定性的。

然而，对于那些深度依赖运行时动态类加载、代码生成或复杂反射的大型单体应用，迁移成本和复杂性可能会急剧增加，需要谨慎评估投入产出比。此外，性能权衡也值得注意：虽然原生镜像提供了无与伦比的启动性能，但对于需要长时间运行且对峰值吞吐量要求极高的计算密集型应用，一个经过充分预热和动态剖析优化的 JIT 编译器可能在长时间运行后达到更高的峰值性能。

**展望与审慎评估**

尽管前景广阔，但我们必须清醒地认识到，Java 原生编译技术生态仍在快速演进中，尚未完全成熟。迁移过程中遇到的第三方库兼容性挑战、显著增加的构建时间、以及与传统 JVM 截然不同的调试与诊断范式，都表明这片领域仍在开拓阶段。

对于许多团队和项目而言，虽然原生化是未来的方向，但在当前阶段，是否立即在核心生产环境中大规模采用，可能需要更为审慎的评估。这不仅是对技术收益的考量，也是对团队驾驭新兴技术、解决未知问题能力的考验。持续关注 Spring、GraalVM 及社区的进展，逐步在非核心或新项目中进行试点，或许是更为稳妥的策略。

## 附录

### MyBatisNativeConfiguration.java

```java
import org.apache.commons.logging.LogFactory;
import org.apache.ibatis.annotations.DeleteProvider;
import org.apache.ibatis.annotations.InsertProvider;
import org.apache.ibatis.annotations.SelectProvider;
import org.apache.ibatis.annotations.UpdateProvider;
import org.apache.ibatis.cache.decorators.FifoCache;
import org.apache.ibatis.cache.decorators.LruCache;
import org.apache.ibatis.cache.decorators.SoftCache;
import org.apache.ibatis.cache.decorators.WeakCache;
import org.apache.ibatis.cache.impl.PerpetualCache;
import org.apache.ibatis.javassist.util.proxy.ProxyFactory;
import org.apache.ibatis.javassist.util.proxy.RuntimeSupport;
import org.apache.ibatis.logging.Log;
import org.apache.ibatis.logging.commons.JakartaCommonsLoggingImpl;
import org.apache.ibatis.logging.jdk14.Jdk14LoggingImpl;
import org.apache.ibatis.logging.log4j2.Log4j2Impl;
import org.apache.ibatis.logging.nologging.NoLoggingImpl;
import org.apache.ibatis.logging.slf4j.Slf4jImpl;
import org.apache.ibatis.logging.stdout.StdOutImpl;
import org.apache.ibatis.reflection.TypeParameterResolver;
import org.apache.ibatis.scripting.defaults.RawLanguageDriver;
import org.apache.ibatis.scripting.xmltags.XMLLanguageDriver;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.mapper.MapperFactoryBean;
import org.mybatis.spring.mapper.MapperScannerConfigurer;
import org.springframework.aot.hint.MemberCategory;
import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;
import org.springframework.beans.PropertyValue;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.BeanFactoryAware;
import org.springframework.beans.factory.aot.BeanFactoryInitializationAotContribution;
import org.springframework.beans.factory.aot.BeanFactoryInitializationAotProcessor;
import org.springframework.beans.factory.aot.BeanRegistrationExcludeFilter;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.ConfigurableBeanFactory;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.config.ConstructorArgumentValues;
import org.springframework.beans.factory.support.MergedBeanDefinitionPostProcessor;
import org.springframework.beans.factory.support.RegisteredBean;
import org.springframework.beans.factory.support.RootBeanDefinition;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportRuntimeHints;
import org.springframework.core.ResolvableType;
import org.springframework.util.ClassUtils;
import org.springframework.util.ReflectionUtils;

import java.lang.annotation.Annotation;
import java.lang.reflect.Method;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;
import java.util.stream.Stream;

@Configuration(proxyBeanMethods = false)
@ImportRuntimeHints(MyBatisNativeConfiguration.MyBaitsRuntimeHintsRegistrar.class)
public class MyBatisNativeConfiguration {
    @Bean
    MyBatisBeanFactoryInitializationAotProcessor myBatisBeanFactoryInitializationAotProcessor() {
        return new MyBatisBeanFactoryInitializationAotProcessor();
    }

    @Bean
    static MyBatisMapperFactoryBeanPostProcessor myBatisMapperFactoryBeanPostProcessor() {
        return new MyBatisMapperFactoryBeanPostProcessor();
    }

    static class MyBaitsRuntimeHintsRegistrar implements RuntimeHintsRegistrar {

        @Override
        public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
            Stream.of(RawLanguageDriver.class,
                    XMLLanguageDriver.class,
                    RuntimeSupport.class,
                    ProxyFactory.class,
                    Slf4jImpl.class,
                    Log.class,
                    JakartaCommonsLoggingImpl.class,
                    Log4j2Impl.class,
                    Jdk14LoggingImpl.class,
                    StdOutImpl.class,
                    NoLoggingImpl.class,
                    SqlSessionFactory.class,
                    PerpetualCache.class,
                    FifoCache.class,
                    LruCache.class,
                    SoftCache.class,
                    WeakCache.class,
                    SqlSessionFactoryBean.class,
                    ArrayList.class,
                    HashMap.class,
                    TreeSet.class,
                    HashSet.class
            ).forEach(x -> hints.reflection().registerType(x, MemberCategory.values()));
            Stream.of(
                    "org/apache/ibatis/builder/xml/*.dtd",
                    "org/apache/ibatis/builder/xml/*.xsd"
            ).forEach(hints.resources()::registerPattern);
        }
    }

    static class MyBatisBeanFactoryInitializationAotProcessor
            implements BeanFactoryInitializationAotProcessor, BeanRegistrationExcludeFilter {

        private final Set<Class<?>> excludeClasses = new HashSet<>();

        MyBatisBeanFactoryInitializationAotProcessor() {
            excludeClasses.add(MapperScannerConfigurer.class);
        }

        @Override public boolean isExcludedFromAotProcessing(RegisteredBean registeredBean) {
            return excludeClasses.contains(registeredBean.getBeanClass());
        }

        @Override
        public BeanFactoryInitializationAotContribution processAheadOfTime(ConfigurableListableBeanFactory beanFactory) {
            String[] beanNames = beanFactory.getBeanNamesForType(MapperFactoryBean.class);
            if (beanNames.length == 0) {
                return null;
            }
            return (context, code) -> {
                RuntimeHints hints = context.getRuntimeHints();
                for (String beanName : beanNames) {
                    BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName.substring(1));
                    PropertyValue mapperInterface = beanDefinition.getPropertyValues().getPropertyValue("mapperInterface");
                    if (mapperInterface != null && mapperInterface.getValue() != null) {
                        Class<?> mapperInterfaceType = (Class<?>) mapperInterface.getValue();
                        if (mapperInterfaceType != null) {
                            registerReflectionTypeIfNecessary(mapperInterfaceType, hints);
                            hints.proxies().registerJdkProxy(mapperInterfaceType);
                            hints.resources()
                                    .registerPattern(mapperInterfaceType.getName().replace('.', '/').concat(".xml"));
                            registerMapperRelationships(mapperInterfaceType, hints);
                        }
                    }
                }
            };
        }

        private void registerMapperRelationships(Class<?> mapperInterfaceType, RuntimeHints hints) {
            Method[] methods = ReflectionUtils.getAllDeclaredMethods(mapperInterfaceType);
            for (Method method : methods) {
                if (method.getDeclaringClass() != Object.class) {
                    ReflectionUtils.makeAccessible(method);
                    registerSqlProviderTypes(method, hints, SelectProvider.class, SelectProvider::value, SelectProvider::type);
                    registerSqlProviderTypes(method, hints, InsertProvider.class, InsertProvider::value, InsertProvider::type);
                    registerSqlProviderTypes(method, hints, UpdateProvider.class, UpdateProvider::value, UpdateProvider::type);
                    registerSqlProviderTypes(method, hints, DeleteProvider.class, DeleteProvider::value, DeleteProvider::type);
                    Class<?> returnType = MyBatisMapperTypeUtils.resolveReturnClass(mapperInterfaceType, method);
                    registerReflectionTypeIfNecessary(returnType, hints);
                    MyBatisMapperTypeUtils.resolveParameterClasses(mapperInterfaceType, method)
                            .forEach(x -> registerReflectionTypeIfNecessary(x, hints));
                }
            }
        }

        @SafeVarargs
        private <T extends Annotation> void registerSqlProviderTypes(
                Method method, RuntimeHints hints, Class<T> annotationType, Function<T, Class<?>>... providerTypeResolvers) {
            for (T annotation : method.getAnnotationsByType(annotationType)) {
                for (Function<T, Class<?>> providerTypeResolver : providerTypeResolvers) {
                    registerReflectionTypeIfNecessary(providerTypeResolver.apply(annotation), hints);
                }
            }
        }

        private void registerReflectionTypeIfNecessary(Class<?> type, RuntimeHints hints) {
            if (!type.isPrimitive() && !type.getName().startsWith("java")) {
                hints.reflection().registerType(type, MemberCategory.values());
            }
        }

    }

    static class MyBatisMapperTypeUtils {
        private MyBatisMapperTypeUtils() {
            // NOP
        }

        static Class<?> resolveReturnClass(Class<?> mapperInterface, Method method) {
            Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
            return typeToClass(resolvedReturnType, method.getReturnType());
        }

        static Set<Class<?>> resolveParameterClasses(Class<?> mapperInterface, Method method) {
            return Stream.of(TypeParameterResolver.resolveParamTypes(method, mapperInterface))
                    .map(x -> typeToClass(x, x instanceof Class ? (Class<?>) x : Object.class)).collect(Collectors.toSet());
        }

        private static Class<?> typeToClass(Type src, Class<?> fallback) {
            Class<?> result = null;
            if (src instanceof Class<?>) {
                if (((Class<?>) src).isArray()) {
                    result = ((Class<?>) src).getComponentType();
                } else {
                    result = (Class<?>) src;
                }
            } else if (src instanceof ParameterizedType) {
                ParameterizedType parameterizedType = (ParameterizedType) src;
                int index = (parameterizedType.getRawType() instanceof Class
                        && Map.class.isAssignableFrom((Class<?>) parameterizedType.getRawType())
                        && parameterizedType.getActualTypeArguments().length > 1) ? 1 : 0;
                Type actualType = parameterizedType.getActualTypeArguments()[index];
                result = typeToClass(actualType, fallback);
            }
            if (result == null) {
                result = fallback;
            }
            return result;
        }

    }

    static class MyBatisMapperFactoryBeanPostProcessor implements MergedBeanDefinitionPostProcessor, BeanFactoryAware {

        private static final org.apache.commons.logging.Log LOG = LogFactory.getLog(
                MyBatisMapperFactoryBeanPostProcessor.class);

        private static final String MAPPER_FACTORY_BEAN = "org.mybatis.spring.mapper.MapperFactoryBean";

        private ConfigurableBeanFactory beanFactory;

        @Override
        public void setBeanFactory(BeanFactory beanFactory) {
            this.beanFactory = (ConfigurableBeanFactory) beanFactory;
        }

        @Override
        public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
            if (ClassUtils.isPresent(MAPPER_FACTORY_BEAN, this.beanFactory.getBeanClassLoader())) {
                resolveMapperFactoryBeanTypeIfNecessary(beanDefinition);
            }
        }

        private void resolveMapperFactoryBeanTypeIfNecessary(RootBeanDefinition beanDefinition) {
            if (!beanDefinition.hasBeanClass() || !MapperFactoryBean.class.isAssignableFrom(beanDefinition.getBeanClass())) {
                return;
            }
            if (beanDefinition.getResolvableType().hasUnresolvableGenerics()) {
                Class<?> mapperInterface = getMapperInterface(beanDefinition);
                if (mapperInterface != null) {
                    // unofficial, see https://github.com/baomidou/mybatis-plus/issues/5826
                    ConstructorArgumentValues constructorArgumentValues = new ConstructorArgumentValues();
                    constructorArgumentValues.addGenericArgumentValue(mapperInterface);
                    beanDefinition.setConstructorArgumentValues(constructorArgumentValues);
                    // Exposes a generic type information to context for prevent early initializing
                    beanDefinition
                            .setTargetType(ResolvableType.forClassWithGenerics(beanDefinition.getBeanClass(), mapperInterface));
                }
            }
        }

        private Class<?> getMapperInterface(RootBeanDefinition beanDefinition) {
            try {
                return (Class<?>) beanDefinition.getPropertyValues().get("mapperInterface");
            }
            catch (Exception e) {
                LOG.debug("Fail getting mapper interface type.", e);
                return null;
            }
        }

    }
}
```