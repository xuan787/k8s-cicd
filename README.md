# Jenkins + K8s 云原生CI/CD项目
## 项目简介
基于Jenkins Kubernetes Agent构建一套完整CI/CD流水线，实现SpringBoot项目从代码提交、单元测试、代码质量扫描、镜像构建、自动部署到Kubernetes集群。
> 技术栈：Jenkins、Kubernetes、Maven、SonarQube、Kaniko、Harbor、sed、kubectl

## 流水线整体流程
1. Checkout：jnlp容器拉取GitHub源码
2. Unit Test：builder容器执行`mvn clean test`单元测试，生成测试报告
3. SonarQube Analysis：Maven sonar插件上传源码+测试报告到SonarQube，做代码质量扫描
4. QualityGate：Jenkins插件轮询SonarQube质量门禁，不通过直接终止流水线
5. Build Package：Maven打包生成SpringBoot可执行jar包，跳过单元测试
6. Build & Push Image：Kaniko构建容器镜像，同时打BUILD_NUMBER版本标签与latest浮动标签，推送至Harbor私有镜像仓库；开启Kaniko层缓存，支持自建Harbor自签名证书跳过TLS校验
7. Deploy：kubectl客户端，使用sed替换yaml镜像占位符，kubectl apply声明式部署到K8s dev命名空间，触发Deployment滚动更新
8. Post钩子：流水线结束，区分成功/失败输出状态

## 架构说明
Jenkins Agent以临时Pod形式运行在K8s集群，Pod内多Sidecar容器：jnlp、builder(maven)、kaniko、kubectl。所有容器共享WORKSPACE工作目录；流水线执行完成后Agent Pod自动销毁。
- Kaniko：无需docker daemon，规避docker.sock安全风险，镜像构建+推送一体化
- kubectl仅作为客户端，调用K8s APIServer，业务Pod运行在集群Node节点，不是CI临时Pod

## 前置环境依赖
1. K8s集群
2. Jenkins部署在K8s，安装Kubernetes插件、SonarQube插件
3. SonarQube服务
4. Harbor私有镜像仓库（提前创建项目与cache缓存仓库）
5. Maven环境，SpringBoot Demo代码

## 部署要点
1. Harbor凭证：通过Secret挂载到kaniko容器 `/kaniko/.docker/config.json`
2. kubectl认证：kubeconfig挂载进kubectl容器，授予dev命名空间Deployment创建权限
3. SonarQube：Jenkins配置SonarQube服务地址与token，配置质量门阻断规则
4. sed替换：原生K8s yaml不支持环境变量，使用sed实现镜像版本动态注入

## 项目亮点
1. 多容器Jenkins Agent，工具环境隔离，临时Pod用完销毁，环境干净
2. Kaniko无daemon构建镜像，支持镜像层缓存加速构建，适配内网自建Harbor
3. SonarQube质量门禁卡点，代码不达标阻断发布
4. 声明式K8s资源部署，滚动更新应用
5. 流水线后置钩子，捕获成功/失败状态，可扩展消息通知
