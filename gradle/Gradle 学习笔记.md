# Gradle 学习笔记

## 构建工具简介

主要构建工具有：

1. ant
2. maven
3. gradle

### 构建工具可以做什么

构建工具可以执行软件开发人员在日常活动中执行的各种任务：

1. **下载和添加依赖项。**当您的项目依赖大量库时，这特别方便
2. **将源代码编译为字节码**。生成工具将为项目中的所有文件调用编译器
3. **包装已编译的代码。**将应用程序归档，例如JAR，APK或其他文件
4. **运行测试。**例如，每次测试应用程序存档以检查其是否正常运行。它允许程序员在修改应用程序后避免错误
5. **部署**到生产环境

## Gradle 介绍

**Gradle**是一种现代自动化工具，可帮助构建和管理Java，Kotlin，Scala和其他基于JVM的语言编写的项目。它描述了项目依赖关系，并确定了如何构建项目

Gradle 支持 `domain-specific language (DSL)` 语言

### 主要特性

Gradle有以下主要特性：

1. 配置文件：Gradle使用几种类型的设置文件来描述如何构建项目
2. 默认构建：无需指定需要完成的每个构建步骤。Gradle使用默认设置和行为。但是，可以根据需要自定义默认构建过程的每个步骤
3. 依赖管理：Gradle自动下载指定的外部库，并解决具有依赖性的冲突情况。您可以声明项目所需的多个依赖项
4. 构建：Gradle允许程序员设计结构合理且易于维护的可理解的构建过程。它还支持复杂的情况，例如多项目或部分构建
5. 易于迁移：Gradle可以轻松适应您拥有的任何项目结构
6. DSL（Groovy / Kotlin）：用于在设置文件中编写脚本

### 下载并安装Gradle

#### macOS安装

```bash
brew install gradle
```

#### 手动安装

可以从[官方网站](https://gradle.org/releases/)下载Gradle，然后将压缩文件解压缩到计算机上的任意位置，然后将文件路径（到bin目录）添加到配置文件

#### 验证安装

在终端中输入

```bash
gradle -v
```

## 使用 Gradle 构建基础项目

### 项目和任务

项目：一个项目可能包含项目构建和部署等，每个`Gradle` 构建都包含一个或者多个项目

任务：是构建执行的单个工作（编译、测试、部署、生成文档等）。每个项目本质上都是一个或多个任务的集合

### 初始化基础项目 - Gradle管理的

1. 创建一个新目录来存储项目文件，然后转到该目录

```java
mkdir gradle-demo
cd gradle-demo
```

2. 调用`gradle init`命令以生成一个简单项目。现代版本的Gradle会要求您在对话框中填写几个参数。要熟悉该过程，只需选择`basic`作为项目类型和`Groovy`构建脚本DSL

   创建成功后，当前目录将会有 `gradle` 自动创建的文件

   ```
   .
   ├── build.gradle
   ├── gradle
   │   └── wrapper
   │       ├── gradle-wrapper.jar
   │       └── gradle-wrapper.properties
   ├── gradlew
   ├── gradlew.bat
   └── settings.gradle
   ```

   文件说明：

   - `build.gradle`: Gradle 构建项目的主要文件，该文件包含`任务`和`外部依赖库`
   - `gradle-wrapper.jar, gradle-wrapper.properties, gradlew, gradlew.bat` 是 Gradle 的包装器，可以使你不用手动安装就可运行 Gradle
   - `settings.gradle`: 用于设置在构建中包含哪些项目，如果是单个项目则该文件可选，如果是多个项目则该文件必须

   **使用 `gradle build` 命令构建项目**

3. 修改构建文件

   打开 `build.gradle` 文件，添加下面内容(Groovy DSL)

   ```kotlin
   description = "A basic Gradle project"
   
   task helloGradle {
       doLast {
           println 'Hello, Gradle!'
       }
   }
   ```

   然后在命令行运行构建命令 `gradle helloGradle` (如果要简化输出信息可加上 `-q` 参数：`gradle -q helloGradle`)，将会有下面输出

   ```bash
   > Task :helloGradle
   Hello, Gradle!
   
   BUILD SUCCESSFUL in 426ms
   1 actionable task: 1 executed
   ```

4. 所有任务列表

   `gradle tasks --all` 命令可查看所有可能运行的任务列表

## 使用 Gradle 构建应用

### 初始化应用程序

使用 `gradle init` 生成新的 `Gradle` 项目，项目类型选择：`application`，实现语言选择：`java / kotlin`

### 运行应用程序

`gradle run` 命令用于运行应用程序

`target`目录中包含字节码的`JAR`归档文件。如果要从`target`目录中清除项目文件，只需运行`gradle clean`命令

### 插件

添加`plugins` 以扩展项目功能

```kotlin
plugins {
    // Apply the org.jetbrains.kotlin.jvm Plugin to add support for Kotlin.
    id("org.jetbrains.kotlin.jvm") version "1.3.72"

    // Apply the application plugin to add support for building a CLI application in Java.
    application
}
```

`id`是插件的全局唯一标识符或名称

### 仓库和依赖

`repositories` 部分声明从哪里下载依赖项并将其添加到项目中

```groovy
repositories {
    jcenter()
}
```

`dependencies` 部分用于添加依赖库到项目

```
dependencies {
    // Align versions of all Kotlin components
    implementation(platform("org.jetbrains.kotlin:kotlin-bom"))
		...
}
```

### 应用程序插件配置

自动生成的 `application` 插件定义了应用程序的入口，由于有该文件所以才可以使用 `gradle run` 命令运行应用程序

```kotlin
application {
    // Defines the main class for the application
    mainClassName = "org.hyperskill.gradleapp.App"
}

// 在 Gradle 6 新版本中配置如下(Kotlin DSL)
application.mainClassName = "AppKt"
```

`mainClassName` 属性定义了应用程序的入口

### 打包为可执行jar

在 `build.gradle` 文件中添加以下配置

```kotlin
// Groovy DSL 配置
jar {
    // 将所有依赖打入jar包 Groovy 语法
    from {
        configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) }
    }
    
    manifest {
        // 添加程序入口函数
        attributes("Main-Class": "com.zbs.xxxx.AppKt")   // for Groovy DSL
    }
}

// Kotlin DSL 配置
tasks.jar {
    // 将所有依赖打入jar包
    from({
        configurations.runtimeClasspath.get().filter { it.name.endsWith("jar") }.map { zipTree(it) }
    })

    // 添加程序入口函数
    manifest {
        attributes("Main-Class" to "com.zbs.xxxx.AppKt")
    }
}
```

## 依赖管理

添加依赖只需要2步：

1. 定义**仓库**：搜索依赖
2. 定义**依赖**：添加到项目中的依赖

### 定义仓库

仓库：存储 `lib` 的地方，一个项目可以使用0或者多个仓库

下面是兼容`Maven`仓库的别名：

- `mavenCentral()`: fetches dependencies from the [Maven Central Repository](https://mvnrepository.com/repos/central).
- `jcenter()`: fetches dependencies from the [Bintray’s JCenter Maven repository](https://bintray.com/bintray/jcenter).
- `mavenLocal()`: fetches dependencies from the local Maven repository.
- `google()`: fetches dependencies from [Google Maven repository](https://maven.google.com/web/index.html).

在 `Gradle` 中定义仓库，只需要修改 `build.gradle` 文件

```kotlin
repositories {
    mavenCentral()
    jcenter()
}
```

默认情况下，`Gradle` 会将下载的 `lib` 包放在项目的 `libs` 目录中，但是可以通过下面的配置将下载的`lib` 包放在任意的位置

```kotlin
repositories {
    flatDir {
        dirs 'lib'
    }
}
```

### 定义依赖

要添加一个新的依赖到项目中，你需要通过`group`，`name`和`version`属性确定它

这些属性可以在仓库所在的网站搜索到

在`java` 和 `kotlin` 插件中定义了一些用于添加依赖的配置：

1. implementation：默认配置，配置编译时可用的依赖，同时不会将他暴露给其他使用你的编译包的用户
2. compileOnly：配置只在编译时可用的依赖，运行时不需要
3. runtimeOnly：配置只在运行时需要的依赖，编译时不需要
4. api：和 `implementation` 类似，但是会暴露给使用你的编译包的用户

另外，以`test` 为前缀 (e.g. `testImplementation`)的配置是测试使用的

`build.gradle` 文件中添加项目需要的依赖

```kotlin
dependencies {
    // This dependency is used by the application.
    implementation group: 'com.google.guava', 'name': guava, version: '28.0-jre'

    // Use JUnit test framework only for testing
    testImplementation 'junit:junit:4.12'

    // It is only needed to compile the application
    compileOnly 'org.projectlombok:lombok:1.18.4'
}
```

















