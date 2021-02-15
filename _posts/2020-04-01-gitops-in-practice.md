---
title:  GitOps 实践
classes: wide
categories:
  - 2020-04
tags:
  - gitops
  - cicd
  - jenkins
  - argocd
---

在看了 CERN 的 GitOps [视频介绍](https://blog.justinzx.com/managing-helm-deployments-with-gitops-at-cern/)后，决定实践一下，使用 git 仓库作为 K8S 集群唯一事实来源（single source of truth）, 集群的所有改动需要通过 git 来同步。工作流如下图：

![gitops](/assets/images/2020/04/gitops.png)

该流程相比传统 CI/CD 流水线最大的区别在于：引入 Deploy Repo 将 CI 和 CD 解耦。

CI 只负责集成，推送镜像到仓库，更新 deploy 仓库，无权访问生产环境，减少生产环境的攻击层面（[attack surface](https://en.wikipedia.org/wiki/Attack_surface)）。

CD 只负责从 deploy 仓库读取最新状态，并把差异应用到生产环境。CI 可以用 Jenkins 或任何其他工具，CD 可以使用 CNCF 下的 ArgoCD 或 FluxCD。

此外，开发、测试、生产环境的所有改动在 git 仓库记录，方便审计和回退。


## CI

这里使用 Jenkins 作为 CI 系统，流水线如下：

![jenkins](/assets/images/2020/04/jenkins.png)

1. 获取源码，执行 Lint, Unit Test, E2E test等;

2. 构建 Docker image 并推送到镜像仓库;

3. 在构建、测试都完成后，最后一段流水线负责更新 deploy 仓库里的版本号信息：

```
   stage('Update deploy repository') {
        /* Update deploy repo */
        if (!isMR) {
            withCredentials([usernamePassword(credentialsId:'xxx', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                sh(script: """
                pwd
                git clone --depth=1 --single-branch --branch master http://$USERNAME:$PASSWORD@gitlab.xxx.com/user/oms-deploy
                cd oms-deploy && ./update-image-tag.py dev/app/deploy.yaml $appImgVersion &&  ./update-image-tag.py dev/app/migration.yaml $appImgVersion && ./update-image-tag.py dev/web/deploy.yaml $appImgVersion
                git add -u . && git commit -m "[jenkins] Update dev image version to $appImgVersion" && git push origin master
                """)
              }
        } else {
            echo "Skip in merge request."
        }
    }

```

## Deploy Repo

Deploy 仓库存放着 K8S 应用的 manifest 文件， 例如 deploy.yaml:

![deploy_yaml](/assets/images/2020/04/deploy_yaml.png)

这些 manifest 文件里的 image 版本号会被 Jenkins 更新，一旦版本号更新后，CD 系统就会把最新的版本应用到相应的环境里。


## CD

这里使用 ArgoCD 作为 CD 系统，创建完 app 并连接到 deploy repo，同步完成后，就会有如下结果：

![argocd](/assets/images/2020/04/argocd.png)


至此，整个 GitOps 流程执行完毕。

---

2021-02-01 update: 这套流程运行半年多下来，每天更新几十次开发测试环境，最大程度减少了人工干预，效果显著。偶尔在 Jenkins 出问题时，也能通过手工更新 deploy 仓库来更新环境。



