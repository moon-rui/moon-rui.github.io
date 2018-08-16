---
layout: article
title: 使用Maven将src/test/java下的文件打包为jar并运行
tags: Maven
---

> 通过Maven将test目录下的测试用例打进jar包

<!--more-->

## 需求
src/test/java下包含MainTest.java等测试用例代码，需要将test和main下的代码以及相关依赖全部打包，并可通过java -jar命令执行MainTest.java
测试用例代码如下：
**MainTest.java**
```java
public class MainTest {

    public static void main(String[] args) {
        System.out.println("Running tests!");

        JUnitCore engine = new JUnitCore();
        engine.addListener(new TextListener(System.out)); // required to print reports
        engine.run(ServiceTest.class);
    }
}
```
**ServiceTest.java**
```java
public class ServiceTest {

    @Before
    public void init() {
        ...
    }

    @After
    public void dispose() {
        ...
    }

    @Test
    public void test1() {
        ...
    }

    @Test
    public void test2() {
        ...
    }
}
```
## 配置
在pom.xml中添加maven-assembly-plugin插件，如下：
```xml
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <!--<version>2.3</version>-->
    <configuration>
        <descriptor>src/main/assembly/assembly.xml</descriptor>
        <archive>
            <manifest>
                <mainClass>com.moon.test.MainTest</mainClass>
            </manifest>
        </archive>
    </configuration>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
根据maven-assembly-plugin插件配置中的路径src/main/assembly/assembly.xml，添加并配置assembly.xml文件：
```xml
<assembly
        xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 http://maven.apache.org/xsd/assembly-1.1.3.xsd">
    <id>assembly</id>
    <formats>
        <format>jar</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <dependencySets>
        <dependencySet>
            <outputDirectory>/</outputDirectory>
            <useProjectArtifact>true</useProjectArtifact>
            <unpack>true</unpack>
            <scope>test</scope>
        </dependencySet>
    </dependencySets>
    <fileSets>
        <fileSet>
            <!-- test目录的编译路径 -->
            <directory>${project.build.directory}/test-classes</directory>
            <outputDirectory>/</outputDirectory>
            <includes>
                <include>**/*.*</include>
            </includes>
            <useDefaultExcludes>true</useDefaultExcludes>
        </fileSet>
        <fileSet>
            <!-- main目录的编译路径 -->
            <directory>${project.build.directory}/classes</directory>
            <outputDirectory>/</outputDirectory>
            <includes>
                <include>**/*.class</include>
            </includes>
            <useDefaultExcludes>true</useDefaultExcludes>
        </fileSet>
    </fileSets>
</assembly>
```
## 打包并测试
执行maven命令：
> mvn clean compile test-compile assembly:single
运行jar包：
> java -jar test-1.0-assembly.jar