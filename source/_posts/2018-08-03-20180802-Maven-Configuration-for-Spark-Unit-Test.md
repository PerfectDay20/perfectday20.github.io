---
title: '20180802 Maven Configuration for Spark Unit Test '
date: 2018-08-03 07:47:57
tags: Spark
---
1. 由于平常开发所用的魔改版 Spark 无法在本地运行，在本地进行单测需要用原版 Spark，通过 Maven 的 profile 进行切换
2. 在 IDEA 的 Maven 插件中可以自定义配置，通过修改 Lifecycle 中的 test 和 package 的配置，可以实现在 test 时使用 local profile，在 package 时使用 online profile，不用再手动选择 profile
3. 由于 Maven 在 package 时会经过 test，而我们需要打包的魔改版 Spark 是无法运行 test 的，这时需要在 scalatest plugin 中增加 skipTests 配置，实现在打包时自动跳过
4. 根据 [Maven Profile Best Practices](https://dzone.com/articles/maven-profile-best-practices)，也可以删掉 online profile，直接将其内容作为默认配置，只保留 local，这样在不添加 profile 选项的时候也可以打包成功（毕竟我们需要的打包就是 online 版本的，而且之前两种 profile 的配置情况下，test 与 package 也要分别使用，所以这样做相比之前没有损失）

---
```xml
    <properties>
        <spark.version>online_version</spark.version>
        <skipTests>true</skipTests>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>${spark.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_2.11</artifactId>
            <version>${spark.version}</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.scalatest</groupId>
                <artifactId>scalatest-maven-plugin</artifactId>
                <version>1.0</version>
                <configuration>
                    <reportsDirectory>${project.build.directory}/surefire-reports</reportsDirectory>
                    <junitxml>.</junitxml>
                    <filereports>WDF TestSuite.txt</filereports>
                    <skipTests>${skipTests}</skipTests>
                </configuration>
                <executions>
                    <execution>
                        <id>test</id>
                        <goals>
                            <goal>test</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    <profiles>
        <profile>
            <id>local</id>
            <properties>
                <spark.version>2.2.1</spark.version>
                <skipTests>false</skipTests>
            </properties>
        </profile>
    </profiles>
```

