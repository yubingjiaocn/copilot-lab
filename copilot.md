# 使用Copilot CLI 在ECS上部署应用

## 实验内容

在本次试验中，我们将使用copilot CLI，体验：
* 利用copilot在ECS上部署一个已有镜像的应用
* 可选：利用Task Metadata Endpoint获取任务的性能指标

## 环境准备

本次我们会提供一个测试账号，该账号下包含一个Cloud9开发环境，预装：

* copilot
* Docker
* AWS CLI
* git

同时该账号有足够的权限管理ECS集群和工作负载。

访问此环境的步骤如下：

* 在浏览器中打开 https://catalog.workshops.aws/ ，选择**Getting Started**
* 认证方式选择**Email one-time password (OTP)**
* 输入您的邮箱，系统会将一次性密码发送到您的邮箱中
* 输入邮件中的9位密码，完成认证
* 在**Event Access Code**中，输入现场提供的12位Access Code，进入Workshop
* 在左下角的**AWS account access**中，选择**Open AWS console (us-west-2)**，进入AWS控制台
* 在上方搜索框中，输入**Cloud9**
* 选择**aws-copilot-workshop**一行的**Open**，打开Cloud9 IDE

## 利用copilot在ECS上部署一个已有镜像的应用

Copilot 将应用分为Application-Service两级，每个Application可包含多个Service。

同时每个Application可包含多个Environment，每个Environment对应一个独立的ECS集群，在部署时可以选择将不同版本的Service部署到不同的Environment。

所有的状态都存储在System Manager的Parameter Store。

在本阶段，我们会利用copilot的交互式CLI，初始化一个应用以及对应的服务，并将其部署到新创建的ECS集群。

请注意，以下所有的交互式CLI操作均可通过编写配置文件或命令行参数实现。

### 初始化应用

* 运行 `cd workshop/work-folder` 切换到工作目录
* 运行 `copilot init`
* 在 `What would you like to name your application? `，输入应用名称 `app`
* 在 `Which workload type best represents your architecture?` ，使用上下箭头选择 `Load Balanced Web Service`，该选项会创建带公网LB的ECS Service，运行在Fargate上。
* 在 `What would you like to name your service? `，输入应用名称 `httpbin`
* 在 `Which Dockerfile would you like to use for httpbin?`，由于我们使用现有镜像，选择 `Use an existing image instead`
* 在 `What's the location ([registry/]repository[:tag|@digest]) of the image to use?` ，输入 `kennethreitz/httpbin`
* 在 `Which port do you want customer traffic sent to?`，输入端口号 `80`
* 完成后，copilot会创建对应的配置文件和角色，配置文件会存储在 `copilot/httpbin/manifest.yml` 目录中。
* 在 `Would you like to deploy a test environment?`，输入 `N`，暂时不部署到测试环境。
* 运行 `cat copilot/httpbin/manifest.yml` 查看刚刚生成的配置文件。
* 编辑该配置文件，将 `connect: true` 改为 `connect: false` ，在该场景中我们不需要使用ECS Service Connect。其他内容保持不变。

此时，我们完成了应用配置文件的生成，可以部署到ECS上。

### 初始化环境

* 运行`copilot env init`
* 在 `What is your environment's name?` ，输入 `dev`
* 由于我们使用Cloud9上的权限，在 `Which credentials would you like to use to create dev` 中，选择 `[profile default]`
* 在 `Would you like to use the default configuration for a new environment?`，选择`Yes, use default.`以接受默认设置。默认设置会创建一个新的VPC，使用2个可用区并创建对应子网。
* copilot CLI会创建IAM Role等辅助资源，同时会将环境配置文件写入`copilot/environments/dev/manifest.yml`。本次我们无需修改。
* 运行`copilot env deploy --name dev`，创建ECS集群。该过程预计需要5-8分钟。

### 部署并测试应用

* 运行`copilot svc deploy --name httpbin --env dev`，将应用部署到ECS集群上。该过程预计需要3-5分钟。
* 运行完成后，会提示一个LB地址，使用浏览器访问即可。
* 也可使用 `copilot svc show -n httpbin` 查看服务状态和对应的LB地址。

---

## 利用Task Metadata Endpoint获取任务的性能指标
