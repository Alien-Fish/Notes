创建maven项目：mvn archetype:create
指定 group： -DgroupId=packageName
指定 artifact：-DartifactId=projectName

创建web项目：-DarchetypeArtifactId=maven-archetype-webapp

创建maven项目：mvn archetype:generate

验证项目是否正确：mvn validate

maven 打包：mvn package

只打jar包：mvn jar:jar

生成源码jar包：mvn source:jar

产生应用需要的任何额外的源代码：mvn generate-sources

编译源代码： mvn compile

编译测试代码：mvn test-compile

运行测试：mvn test

运行检查：mvn verify

清理maven项目：mvn clean

生成idea项目：mvn idea:idea

安装项目到本地仓库：mvn install

发布项目到远程仓库：mvn:deploy

在集成测试可以运行的环境中处理和发布包：mvn integration-test

显示maven依赖树：mvn dependency:tree

显示maven依赖列表：mvn dependency:list

下载依赖包的源码：mvn dependency:sources

安装本地jar到本地仓库：mvn install:install-file -DgroupId=packageName -DartifactId=projectName -Dversion=version -Dpackaging=jar -Dfile=path

跳过测试:
-Dmaven.test.skip=true

