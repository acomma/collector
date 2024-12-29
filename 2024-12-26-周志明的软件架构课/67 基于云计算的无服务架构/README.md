你好，我是周志明。

首先我们要知道，无服务架构（Serverless）跟微服务架构本身没有继承替代的关系，它们并不是同一种层次的架构，无服务的云函数可以作为微服务的一种实现方式，甚至可能是未来很主流的实现方式。在课程中，我们的话题主要还是聚焦在如何解决分布式架构下的种种问题，所以相对来说，无服务架构并不是重点，不过为了保证架构演进的完整性，我仍然建立了无服务架构的简单演示工程。

另外还要明确一点，由于无服务架构在原理上就决定了它对程序的启动性能十分敏感，这天生就不利于 Java 程序，尤其不利于 Spring 这类启动时组装的 CDI 框架。因此基于 Java 的程序，除非使用[GraalVM 做提前编译](https://icyfenix.cn/tricks/2020/graalvm/substratevm.html)、将 Spring 的大部分 Bean 提前初始化，或者迁移至[Quarkus](https://quarkus.io/)这种以原生程序为目标的框架上，否则是很难实际用于生产的。

## 运行程序

Serverless 架构的 Fenix's Bookstore 是基于[亚马逊 AWS Lambda平台](https://aws.amazon.com/cn/lambda/)运行的，这是最早商用，也是目前全球规模最大的 Serverless 运行平台。不过从 2018 年开始，中国的主流云服务厂商，比如阿里云、腾讯云也都推出了各自的 Serverless 云计算环境，如果你需要在这些平台上运行 Fenix's Bookstore，你要根据平台提供的 Java SDK 对 StreamLambdaHandler 的代码做少许调整。

现在，假设你已经完成了[AWS 注册](https://repost.aws/zh-Hans/knowledge-center/create-and-activate-aws-account)、配置[AWS CLI 环境](https://aws.amazon.com/cn/cli/)以及 IAM 账号的前提下，就可以通过以下几种途径来运行程序，浏览最终的效果：

通过 AWS SAM（Serverless Application Model） Local 在本地运行：

AWS CLI 中附有 SAM CLI，但是版本比较旧，你可以通过[如下地址](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)安装最新版本的 SAM CLI。另外，SAM 需要 Docker 运行环境支持，你可参考[此处](https://icyfenix.cn/appendix/deployment-env-setup/setup-docker.html)部署。

首先编译应用出二进制包，执行以下标准 Maven 打包命令即可：

```shell
$ mvn clean package
```

根据 pom.xml 中 assembly-zip 的设置，打包将不会生成 SpringBoot Fat JAR，而是产生适用于 AWS Lambda 的 ZIP 包。打包后，确认已经在 target 目录生成了 ZIP 文件，并且文件名称与代码中提供的 sam.yaml 配置的一致，然后在工程根目录下运行如下命令，启动本地 SAM 测试：

```shell
$ sam local start-api --template sam.yaml
```

通过 AWS Serverless CLI 将本地 ZIP 包上传至云端运行：

在确认已经配置了 AWS 凭证后，工程中已经提供了 serverless.yml 配置文件，确认文件中 ZIP 的路径与实际 Maven 生成的一致，然后在命令行执行：

```shell
$ sls deploy
```

此时，Serverless CLI 会自动将 ZIP 文件上传至 AWS S3，然后生成对应的 Layers 和 API Gateway，运行结果如下所示：

```shell
$ sls deploy
Serverless: Packaging service...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service bookstore-serverless-awslambda-1.0-SNAPSHOT-lambda-package.zip file to S3 (53.58 MB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
..............
Serverless: Stack update finished...
Service Information
service: spring-boot-serverless
stage: dev
region: us-east-1
stack: spring-boot-serverless-dev
resources: 10
api keys:
  None
endpoints:
  GET - https://cc1oj8hirl.execute-api.us-east-1.amazonaws.com/dev/
functions:
  springBootServerless: spring-boot-serverless-dev-springBootServerless
layers:
  None
Serverless: Removing old service artifacts from S3...
```

访问输出结果中的地址（比如上面显示的https://cc1oj8hirl.execute-api.us-east-1.amazonaws.com/dev/）即可浏览结果。

这里要注意，由于 Serverless 对响应速度的要求本来就较高，所以我不建议再采用 HSQLDB 数据库来运行程序了，毕竟每次冷启动都重置一次数据库本身也并不合理。代码中有提供 MySQL 的 Schema，我建议采用 AWS RDB MySQL/MariaDB 作为数据库来运行。

## 协议

课程的工程代码部分采用[Apache 2.0 协议](https://www.apache.org/licenses/LICENSE-2.0)进行许可。在遵循许可的前提下，你可以自由地对代码进行修改、再发布，也可以将代码用作商业用途。但要求你：

* **署名**：在原有代码和衍生代码中，保留原作者署名及代码来源信息；

* **保留许可证**：在原有代码和衍生代码中，保留 Apache 2.0 协议文件。