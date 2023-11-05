---
title: Maven实战笔记
auther: gzj
tags:
  - Java基础
categories: Java基础
description: >-
  Maven 翻译为"专家"、"内行"，是 Apache 下的一个纯 Java
  开发的开源项目。基于项目对象模型（缩写：POM）概念，Maven利用一个中央信息片断能管理一个项目的构建、报告和文档等步骤。
abbrlink: a0d99dad
date: 2021-03-03 18:58:39
---

# 第二章 Maven使用入门

## 编写POM

POM(Project Object Model， 项目对象模型)定义了项目的基本信息，用于描述项目如何构建，声明项目依赖等等。

实例：

```xml
<?xml version="1.0" encoding="utf-8" ?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
    http://maven.apache.org/maven-v4_0_0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.gzj.testMaven</groupId>
    <artifactId>hello-world</artifactId>
    <version>1.0-SNAPSHOT</version>
    <name>my maven project</name>
    
</project>
```

第一行是XML头，指定了文档版本和编码方式。后面的project是所有pom.xml的根元素，它还声明了一些POM相关的命名空间和xsd元素，可以帮助我们快速编辑POM。

modelVersion：指定当前POM模型的版本，对于Maven2和Maven3来说，它只能是4.0.0。

groupId，artifactId，version这三个元素定义了一个项目的基本坐标。

- groupId：定义了项目属于那个组，这个组和项目所在的组织或公司存在关联。

- artifactId：定义了当前项目在组的唯一ID，因为可能有不同的子模块，所以要分配不同的artifactId。

- version：指定了项目的当前版本。

name：声明一个对用户更友好的项目名称，虽然不必须，但推荐写，方便信息交流。

## 编写主代码

项目主代码和测试代码不同，项目的主代码会被打包，而测试代码只会在运行测试时用到，不会被打包。默认情况下，maven约定项目的主代码位于src/main/java目录。

代码编写完毕后，使用Maven编译，在项目根目录下运行mvn clean compile。

clean是告诉maven清除输出目录target/，compile告诉maven编译项目主代码。

## 编写测试代码

测试代码位于src/test/java目录。测试一般使用JUnit，需要在pom文件新增一个JUnit依赖。

```xml
新增：
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.7</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

dependencies元素下可以包含多个dependency元素以声明项目的依赖。前面我们提到通过groupId，artifactId，version是任何一个maven项目最基本的坐标，JUnit也是，有了这段声明，maven就能自动下载JUnit。

上述POM代码还有一个值为test的元素scope，代表依赖范围，若依赖范围是test表示该依赖只对测试有效，就是在测试代码用是没有问题，但在主代码用就会编译错误。如果不声明依赖范围，那么默认值就是compile，表示对主代码和测试代码都有效。

编写完毕后调用Maven执行测试，运行mvn clean test。

有时maven输出提示我们需要使用 -source 5或更高版本，这时可能是maven核心插件的compiler插件默认支持编译的Java版本低，要改成支持Java5的版本。

在pom文件添加：

```xml 
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.5</source>
                <target>1.5</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 打包和运行

若没有指定打包类型的话，默认打包类型是jar，可以执行命令mvn clean package进行打包。

打包完毕后，可以在target/输出目录中看见，名称是根据artifactId-version.jar进行命名的。

如果想让其他的Maven项目直接引用这个jar，可以把它安装到maven仓库，执行mvn clean install就会把它安装到Maven本地仓库中。

虽然项目中有的类含有main方法，但默认打包生成的jar不能直接运行，因为带有main方法的类信息不会添加到manifest中(jar文件的*META-INF/MANIFEST.MF*文件)。

若要生成可执行的jar文件，需要借助maven-shade-plugin插件：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>1.2.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.gzj.testMaven.Main</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

配置mainClass为com.gzj.testMaven.Main，项目打包时就会把信息放到MANIFEST中。执行mvn clean install，构建完成后打开target/目录，可以看到hello-world-1.0-SNAPSHOT.jar和original-hello-world-1.0-SNAPSHOT.jar两个文件，前者是带有Main-Class信息的可运行jar，后者是原始的jar，打开hello-world-1.0-SNAPSHOT.jar的MANIFEST.MF可以看到多了`Main-Class: com.gzj.testMaven.Main`这一行信息。

## 使用Archetype生成项目骨架

每次构建项目时都要自己创建pom文件和src/main/java等一些固定的包，会让我们感到很繁琐，通过maven archetype可以快速创建项目的骨架。

输入maven archetype后，会先让我们选择archetype，然后输入项目的groupId、artifactId、version和package，完了后确认就生成成功了。

默认生成的pom.xml包含一个junit依赖，主代码和测试代码已被创建。

# 第三章 坐标和依赖

## 何为Maven坐标

界上任何一个JAR包或者WAR包都可以使用Maven坐标唯一标识，Maven坐标的元素包括groupId、artifactId、version和classifier等。只要我们提供了正确的坐标元素，Maven就能找到对应的JAR包或者WAR包。

## 坐标详解

groupId：定义当前Maven项目隶属的实际项目；我们要明白的是Maven项目和实际项目不一定是一对一的关系。这是由于Maven中模块的概念，因此实际项目往往会被划分成很多模块。groupId和我们在Java中定义顶级包名的规则是一样的，通常与公司或者组织的域名反向一一对应。

artifactId：该元素定义实际项目中的一个Maven项目（模块），一般推荐的做法是使用实际项目名称作为artifactId的前缀，比如spring-core的前缀是spring一样。

version：该元素定义Maven项目当前所处的版本。

packaging：定义Maven项目的打包方式，打包方式会影响到构建的生命周期，jar打包和war打包使用不同的命令。不定义packaging的时候，Maven默认使用jar。

classifier：该元素帮助定义构建输出的一些附属构件。比如主构件是nexus-indexer-2.0.0.jar，可能使用插件生成nexus-indexer-2.0.0-javadoc.jar，nexus-indexer-2.0.0-sources.jar等附属构件。javadoc和sources就是这两个附属构件的classifier。这样附属构件也有了自己唯一的坐标。

在项目中，groupId、artifactId、version是必须定义的，packaging是可选的（默认jar），而classifier是不能直接定义的。

同时，项目构件的文件名是与坐标相对应的，一般是artifactId-version[-classifier].packaging，[-classifier]表示可选。

## 依赖范围

Maven在编译项目主代码的时候要使用一套classpath，在编译和执行测试时会使用另外一套classpath，实际运行Maven项目的时候，又会使用一套classpath。

依赖范围就是控制依赖与这三种classpath的关系，Maven有以下几种依赖范围：

- Compile：编译依赖范围。如果没有指定，就会默认使用该依赖范围。该依赖范围的依赖，对于编译、测试、运行这三种classpath都有效。
- Test：测试依赖范围。使用此依赖范围的Maven依赖，只对测试calsspath有效，在编译主代码或运行项目时无法使用。
- Provided：已提供依赖范围。使用此依赖范围的Maven依赖，对于编译和测试classpath有效，但运行时无效。比如servlet-api，编译测试用，但运行时容器已经提供，不需要Maven再导入。
- Runtime：运行时依赖范围。使用此依赖范围的Maven依赖，对于测试和运行classpath有效，但编译主代码无效。比如JDBC驱动实现，主代码只需要JDK提供的JDBC接口，只有在执行测试或运行实际项目的时候才需要实现上述接口的具体JDBC驱动。
- Ststem：系统依赖范围。该依赖与三种classpath的关系和Provided依赖范围完全一致。但使用System范围的依赖必须通过systemPath元素显式指定依赖文件的路径。此类依赖不是通过Maven仓库解析，往往与本机系统绑定，可能造成构建的不可移植，谨慎使用。systemPath元素可以引用环境变量。
- Import（Maven2.0.9及以上）：导入依赖范围。该依赖范围不会test、compile、runtime的 classpath 产生实际的影响。它的作用是将其他模块定义好的 dependencyManagement 导入当前 Maven 项目 pom 的 dependencyManagement 中。

## 传递性依赖

### 何为传递性依赖

例如：A.jar依赖于B.jar，而B.jar依赖于C.jar，那要使A.jar 依赖于C.jar 当且仅当C.jar的范围是compile。

有了Maven的传递性依赖机制，不用担心引入多余的依赖。 Maven会解析各个直接依赖的POM, 将那些必要的间接依赖，以传递性依赖的形式引入到当前的项目中。

### 传递性依赖和依赖范围

假设A依赖于B，B依赖于C，那么A对于B是第一直接依赖，B对于C是第二直接依赖，A对于C是传递性依赖。第一直接依赖的范围和第二直接依赖的范围决定了传递性依赖的范围。

 如下图： 最左边一列表示第一直接依赖范围， 最上面一行表示第二直接依赖范围， 中间交叉单元格则表示传递性依赖的范围。

|          | compile  | test | provided | runtime  |
| -------- | -------- | ---- | -------- | -------- |
| compile  | compile  | ---  | ---      | runtime  |
| test     | test     | ---  | ---      | test     |
| provided | provided | ---  | provided | provided |
| runtime  | runtime  | ---  | ---      | runtime  |

通过上表可以发现规律：当第二直接依赖的范围是compile的时候，传递性依赖与第一直接依赖的范围一致； 当第二直接依赖的范围是test的时候，依赖不会得以传递；当第二直接依赖是provided的时候，值传递第一直接依赖范围也为provided的依赖，且传递性依赖范围同样为provided; 当第二依赖的范围是runtime的时候，传递性范围与第一直接依赖的范围一致，但compile例外，此时传递性依赖的范围为runtime。

### 依赖调解

大部分情况下，通过Maven的依赖传递机制，我们只需要关心项目的直接依赖是什么，但有时候，传递性依赖也会造成一些问题。

比如项目A有这样的依赖关系：A->B->C->X(1.0), A->D->X(2.0), X是A的传递性依赖，但是两条依赖路径上有两个版本的X，这时，Maven会进行依赖调解，根据==路径最近者优先==的调解原则，该例中X(1.0)的路径长度为3，而X(2.0)的路径长度为2，因此X(2.0)会被解析使用。

但这种情况不能解决所有问题，如果路径长度一样的情况下，在Maven2.0.8及之前的版本中，这时不确定的，但从Maven2.0.9版本开始，Maven定义了依赖调解的第二原则：==第一声明者优先==。在依赖路径长度相同的情况下，在POM中依赖声明的顺序决定了谁会被解析使用，顺序最靠前的那个依赖优胜。

### 可选依赖

假设有这样一个依赖关系，项目A依赖于项目B,项目B依赖于项目X和Y, B对于X和Y的依赖都是可选依赖： A->B, B->X(可选)，B->Y(可选)。 根据传递性依赖的定义，如果所有这三个依赖的范围都是compile,那么X, Y就是A的compile范围传递性依赖。然而，由于这里X,Y是可选依赖，依赖将不会得以传递。换句话说，X,Y将不会对A有任何影响。

可选依赖可以解决如下情况，如果项目B实现了两个特性，一个特性依赖于X，另一个依赖于Y，这两个特性互斥，比如B是一个持久层工具包，支持X和Y数据库，那构建B工具包时需要这两个依赖，但使用时只会用到一个。

可选依赖配置：

```xml
<dependencies>
   <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.10</version>
      <optional>true</optional>
   </dependency>
   <dependency>
      <groupId>postgresql</groupId>
      <artifactId>postgresql</artifactId>
      <version>8.4-701.jgbc3</version>
      <optional>true</optional>
   </dependency>
</dependencies>
```

optional：这个元素表示mysql-connector-java和postgresql这两个依赖为可选依赖，它们只会对当前项目B产生影响，当其他项目依赖于B的时候，这两个依赖不会被传递。

因此，当项目A依赖于项目B的时候，如果其实际使用基于MySQL数据，那么项目A中就需要显示的声明mysql-connetor-java这一依赖，如下：

```html
<dependencies>
   <dependency>
      <groupId>org.rogueq.mvnbook</groupId>
      <artifactId>project-b</artifactId>
      <version>1.0.0</version>
   </dependency>
   <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.10</version>
   </dependency>
</dependencies>
```

但理想情况不应该使用可选依赖，根据单一职责原则，在上述例子中，最好是根据Mysql和PostgreSQL分别创建Maven项目，分配不同artifactId，直接声明JDBC驱动依赖，不使用可选依赖。