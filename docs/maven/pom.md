# POM 相关

POM(Project Object Model) 是用于声明项目的标识和结构、配置构建以及相互关联的项目的地方。MAVEN项目就是通过文件名为pom的XML文件来定义的。POM不仅只用于JAVA工程，一些C#以及技术文档也可以用POM定义构建。

POM的描述及配置共分四类：

![1571993876389](..\image\1571993876389.png)

- 项目信息

  包含了项目名称，项目URL，赞助机构以及开发者贡献者许可证证等。

- 构建配置

  包含了默认的Maven构建行为，我们可以改变源代码和测试的位置，也可以新增插件，可以附加插件执行周期的目标，也可以自定义site的生成参数

- 构建环境

  构建环境由用于不同环境的配置组成。比如开发环境和生产环境的配置是不同的。构建环境为不同环境下自定义了不同配置。

- POM 关系

  一个项目很少情况下是独立的，往往会依赖其他项目。比如从父项慕继承，定义自身位置，也可能包含子项目。

## Super Pom

所有的Maven工程的POM文件都继承了Super POM，它定义了一系列的默认配置用于所有工程。Super POM是Maven安装文件的一部分，位置在`${M2_HOME}/lib`中的`maven-x.y.z-uber.jar`或者`maven-model-builder-xy.z.jar`中包名为`org.apache.maven.model`下的 `pom-4.0.0.xml`。

> `Super POM`在`POMs`中的作用类似于 `java.lang.Object`在所有`class`中的作用

详细如下：

``` xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- START SNIPPET: superpom -->
<project>
  <modelVersion>4.0.0</modelVersion>

  <repositories>
    <repository>
      <!-- maven 默认仓库地址, 可以在setting.xml文件中修改。注意默认仓库配置禁用了snapshot版本，如果
 		  需要使用，在setting.xml或pom.xml-->
      <id>central</id> 
      <name>Central Repository</name>
      <url>https://repo.maven.apache.org/maven2</url>
      <layout>default</layout>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
    </repository>
  </repositories>

  <pluginRepositories>
    <pluginRepository>
       <!-- 中央仓库也包含插件。同样禁用了snapshot, 并且即使插件发布新的release版本，永远也不会更新 -->
      <id>central</id>
      <name>Central Repository</name>
      <url>https://repo.maven.apache.org/maven2</url>
      <layout>default</layout>
      <snapshots>
        <enabled>false</enabled>
      </snapshots>
      <releases>
        <updatePolicy>never</updatePolicy>
      </releases>
    </pluginRepository>
  </pluginRepositories>

  <!-- build中设置了一系列的Maven标准目录 -->
  <build>
    <directory>${project.basedir}/target</directory>
    <outputDirectory>${project.build.directory}/classes</outputDirectory>
    <finalName>${project.artifactId}-${project.version}</finalName>
    <testOutputDirectory>${project.build.directory}/test-classes</testOutputDirectory>
    <sourceDirectory>${project.basedir}/src/main/java</sourceDirectory>
    <scriptSourceDirectory>${project.basedir}/src/main/scripts</scriptSourceDirectory>
    <testSourceDirectory>${project.basedir}/src/test/java</testSourceDirectory>
    <resources>
      <resource>
        <directory>${project.basedir}/src/main/resources</directory>
      </resource>
    </resources>
    <testResources>
      <testResource>
        <directory>${project.basedir}/src/test/resources</directory>
      </testResource>
    </testResources>
    <pluginManagement>
      <!-- NOTE: These plugins will be removed from future versions of the super POM -->
      <!-- They are kept for the moment as they are very unlikely to conflict with lifecycle mappings (MNG-4453) -->
      <!-- 从2.0.9开始，一些核心插件及版本会在super pom中默认配置，用户不需要在工程pom中指定版本就能使用 -->
      <plugins>
        <plugin>
          <artifactId>maven-antrun-plugin</artifactId>
          <version>1.3</version>
        </plugin>
        <plugin>
          <artifactId>maven-assembly-plugin</artifactId>
          <version>2.2-beta-5</version>
        </plugin>
        <plugin>
          <artifactId>maven-dependency-plugin</artifactId>
          <version>2.8</version>
        </plugin>
        <plugin>
          <artifactId>maven-release-plugin</artifactId>
          <version>2.5.3</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>

  <reporting>
    <outputDirectory>${project.build.directory}/site</outputDirectory>
  </reporting>

  <profiles>
    <!-- NOTE: The release profile will be removed from future versions of the super POM -->
    <profile>
      <id>release-profile</id>

      <activation>
        <property>
          <name>performRelease</name>
          <value>true</value>
        </property>
      </activation>

      <build>
        <plugins>
          <plugin>
            <inherited>true</inherited>
            <artifactId>maven-source-plugin</artifactId>
            <executions>
              <execution>
                <id>attach-sources</id>
                <goals>
                  <goal>jar-no-fork</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <inherited>true</inherited>
            <artifactId>maven-javadoc-plugin</artifactId>
            <executions>
              <execution>
                <id>attach-javadocs</id>
                <goals>
                  <goal>jar</goal>
                </goals>
              </execution>
            </executions>
          </plugin>
          <plugin>
            <inherited>true</inherited>
            <artifactId>maven-deploy-plugin</artifactId>
            <configuration>
              <updateReleaseInfo>true</updateReleaseInfo>
            </configuration>
          </plugin>
        </plugins>
      </build>
    </profile>
  </profiles>

</project>
<!-- END SNIPPET: superpom -->
```

## 最小Pom

如果只是想写一个简单的项目用于提供JAR，或者想在`src/test/java`执行单元测试。只需要在pom中定义 groupId ，artifacetId 和 version，这是每个Maven项目必需的三个坐标。

``` xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>simplest-project</artifactId>
    <version>1</version>
</project>
```

一个最简单的Maven项目如上所示，它不依赖其他项目。执行 mvn package，会在项目 target 目录中生成 jar 包。

## 有效POM

``` bash
$ mvn help:effective-pom
```

执行`effective-pom`命令会生成XML文本展示项目实际生效的Pom信息。

