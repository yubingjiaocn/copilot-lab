# 使用控制台和CLI在ECS上部署应用

## 实验内容

在本次试验中，我们将使用ECS控制台和AWS CLI，体验：
* 在ECS控制台上创建一个ECS任务定义
* 使用ECS控制台创建ECS集群
* 使用AWS CLI创建任务
* 使用控制台和CLI查看任务运行状况

## 环境准备

本次我们会提供一个测试账号，该账号下包含一个Cloud9开发环境，预装：

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


## 创建任务定义

需要任务定义才能在 Amazon ECS 中运行 Docker 容器。任务定义是用来创建任务的模板，内含Docker镜像，资源配置，启动类型，日志等信息。任务定义以JSON形式保存，但可通过控制台或CLI创建。

本次使用的镜像为``ghcr.io/simonkowallik/httpbin:unit`，该镜像是httpbin的修改版，支持通过环境变量自定义返回的Header。我们会使用环境变量传入自定义Header内容。

* 在控制台上方搜索框中，输入**ECS**，进入ECS控制台。
* 选择左边的**Task definitions**，选择**Create new task definition**
* 在**Task definition family**，输入任务定义名称，这里输入`httpbin`
* 在下方的**Container - 1**，输入容器详细信息：在`Name`中输入`httpbin`，**Image URI**输入`ghcr.io/simonkowallik/httpbin:unit`
* 展开下方的**Environment variables - optional**，在**Add individually**中点击**Add environment variable**，增加一个环境变量。在**Key**中填入`XHTTPBIN_X_instance_id`，**Value**留空。
* 选择**Next**，切换到下一页
* 在下一页中，修改Task Size为`.5 vCPU`和`1GB`内存。无需修改其他内容，选择**Next**
* 点击`Create`，创建任务定义。

## 创建ECS集群
在创建完成任务定义后，需要在ECS集群上创建和运行任务。由于使用Fargate运行环境，无需预置EC2实例。

* 选择左边栏的**Clusters**，点击**Create cluster**
* 在**Cluster name**，输入集群名称`dev`，无需修改其他配置，点击**Create**即可完成创建。

## 使用AWS CLI创建任务

完成创建后，即可使用AWS CLI创建任务。在创建任务时，可以修改容器的环境变量和其他参数。，我们会通过修改环境变量的方式，使得httpbin在所有请求均返回含`instance1`的Header。

* 在上方搜索框中，输入**Cloud9**
* 选择**aws-copilot-workshop**一行的**Open**，打开Cloud9 IDE
* 运行 `sudo yum install jq -y`，安装`jq`以解析JSON
* 在终端中运行如下命令以获取子网ID：

  ```bash
  export SUBNET=$(aws ec2 describe-subnets --output json --query 'Subnets[*].SubnetId' | jq -r .[0]); echo $SUBNET
  ```

* 在终端中运行如下命令以获取安全组ID：

  ```bash
  export SG=$(aws ec2 describe-security-groups --group-names default --query 'SecurityGroups[*].[GroupId]' | jq -r .[0][0]); echo $SG
  ```

* 默认情况下，安全组不对外开放访问，但我们需要在终端中运行如下命令以开通80端口访问：

  ```bash
  aws ec2 authorize-security-group-ingress --group-id $SG --protocol tcp --port 80 --cidr 0.0.0.0/0
  ```

* 在终端中运行如下命令以创建任务：
  ```bash
  aws ecs run-task --cluster dev \
    --task-definition httpbin:1 \
    --network-configuration "awsvpcConfiguration={subnets=[$SUBNET],securityGroups=[$SG],assignPublicIp=ENABLED}" \
    --launch-type FARGATE \
    --overrides '{"containerOverrides":[{"name": "httpbin","environment":[{"name":"XHTTPBIN_X_instance_id","value":"instance1"}]}]}'
  ```

* 运行完成后，会得到Task的详细信息。等待20-30秒后，运行以下命令以列出任务列表：

  ```bash
  aws ecs list-tasks --cluster dev
  ```

* 如需获取任务详细信息，运行以下命令：

  ```bash
  export ARN=$(aws ecs list-tasks --cluster dev --query 'taskArns[0]' --output text)
  echo $ARN
  aws ecs describe-tasks --tasks $ARN --cluster dev
  ```

* 如需获取公网IP地址，运行如下命令：
  ```bash
  export ENI=$(aws ecs describe-tasks --task "$ARN" --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' --cluster dev --output text)

  export PUBLIC_IP=$(aws ec2 describe-network-interfaces --network-interface-ids "$ENI" --query 'NetworkInterfaces[0].Association.PublicIp' --output text)

  echo $PUBLIC_IP
  ```

* 通过`curl`访问对应的IP地址，检查环境变量是否正常传入。

  ```bash
  curl -v $PUBLIC_IP/get
  ```

  可以看到，在返回中有`< X-instance-id: instance1` Header，证明环境变量成功传入。
