# GitLab CI + ArgoCDでk8sのGitOpsを試してみる

**これは、[Kubernetes3 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/kubernetes3)の2日目の記事です。**

フォルシアでは複数のアプリにおいてKubernetesが用いられています。参考:[https://www.forcia.com/blog/001519.html](https://www.forcia.com/blog/001519.html)

しかしながら，デプロイ周りについてはまだまだ仕組み化がされておらず，いい感じにデプロイできる仕組みはないかと調べていると「GitOps」というワードが出てきました。

勉強がてら（結構こすられたネタだとは思うのですが）GitOpsを実際に構築してみた学習記録を記したいと思います。（筆者は1ヶ月前ではKubernetesなにそれ状態でした。）

## GitOpsとは

Weave社が提唱した概念です。

[https://www.weave.works/technologies/gitops/](https://www.weave.works/technologies/gitops/)

> GitOps can be summarized as these two things:An operating model for Kubernetes and other cloud native technologies, providing a set of best practices that unify deployment, management and monitoring for containerized clusters and applications.A path towards a developer experience for managing applications; where end-to-end CICD pipelines and Git workflows are applied to both operations, and development.

要するに

全てのリソースの変更や運用に対してコマンドラインを用いずにgit経由で行うことでコードとして履歴管理しようぜという思想

といった感じです。

よりイメージを深めるために，GitOpsを実現した結果期待される状態を述べると，以下のようになります。

- 開発者はデプロイを全く意識しなくていい。（Git/GitHub/GitLabの操作だけでなんかデプロイされる。）
- k8sで言うと手作業でkubectlとかしなくていい。
- アプリケーション部分（テスト/ビルド）とインフラ部分（デプロイ）を疎に繋げられる。
- Gitが信頼できる唯一の情報源（SSOT：Single Source of Truth）（差分検知／自動反映でGitのコードがインフラにある。）

すごい！！GitOps最高！！

これが実現されれば，デプロイ作業から人々が開放されます。

特にGitがSSOTになるというのは素晴らしいと個人的に感じます。本番環境やステージング環境の状態がGitレポジトリを見れば一発で分かるのです。

さて，またk8sのGitOpsには主に2つの派閥があります。

- Push型
    - CIのPipelineで`kubectl`してデプロイする
- Pull型
    - CDツールがSSOT(manifestレポジトリ)の更新を検知してデプロイ

Push型は以下のような問題があり推奨されていません。参考: [https://www.weave.works/blog/why-is-a-pull-vs-a-push-pipeline-important](https://www.weave.works/blog/why-is-a-pull-vs-a-push-pipeline-important)

- サービスの世代管理が困難
- 意図したデプロイ結果になっているか確認が困難
- パワフルな権限を持つCI
- etc...

なので今回はPull型のGitOpsを構築することにしてみました。

## GitLab CIとArgoCDでk8sのGitOpsを実現する

GitOpsを試すために今回はCIツールとして慣れ親しんだGitLab CI/CD（フォルシアではGitLabを用いてコード管理を行っています。[参考](https://www.forcia.com/blog/001413.html)）を，CDツールはGUIが用意されているArgoCDを選択しました（ただGUI画面眺めてニヤニヤしたかっただけです）

他に有名なCDツールとしては[Flux](https://docs.fluxcd.io/) (最近[Flux v2](https://toolkit.fluxcd.io/)がリリースされました)や[Jenkins X](https://jenkins-x.io/)などがあります。

以下構築した全体像です。

![image](/uploads/92506b6277eabd6754ede7f389cc52d2/image.png)

詳しくは今から述べていきます。

ポイントとして，アプリのレポジトリとマニフェストのレポジトリを分けているところがあります。これは[ArgoCDのベストプラクティス](https://blog.argoproj.io/5-gitops-best-practices-d95cb0cbe9ff) に則っています。運用の手間は増えますが，アプリの差分とmanifestの差分がはっきり分かれるのでわかりやすく僕も好みです。

### ArgoCDのインストール

事前にアプリを動かすk8s clusterにArgoCDをdeployしておきます。全体像の絵で示したようにArgoCDはk8sのcluster上で動くからです。

[https://argoproj.github.io/argo-cd/](https://argoproj.github.io/argo-cd/) の手順をそのままやります。

```bash
> kubectl create namespace argocd
> kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# しばらく待ってPodが作成されていることを確認(まあまあの時間がかかります)*
❯ kubectl get pod -n argocd
NAME READY STATUS RESTARTS AGE
argocd-application-controller-5785f6b79-s2cvr 1/1 Running 0 2m39s
argocd-dex-server-7f5d7d6645-z46hr 1/1 Running 0 2m39s
argocd-redis-cccbb8f7-dfbjk 1/1 Running 0 2m39s
argocd-repo-server-67ddb49495-nxkw4 1/1 Running 0 2m39s
argocd-server-6bcbf7997d-cj5bg 1/1 Running 0 2m39s
```

podが全て立ち上がったことを確認したのち，

```bash
> kubectl port-forward svc/argocd-server -n argocd 8080:443
```

でport-forwardさせてあげると，[http://localhost:8080](http://localhost:8080/) でGUI画面にアクセスできるはず。簡単。初期のログインアカウントはadmin, パスワードは以下のコマンドの実行結果（argocd-serverのPod名）です。

`kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2`

![image](/uploads/95bfa807aa7ef328408b1b9101ad9466/image.png)

CLIツールもインストールしておく

```bash
# ArgoCD CLIのインストール
> VERSION**=**$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')
> curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
> chmod +x /usr/local/bin/argocd
# login; 上記のport-forwardを行っている場合***>** argocd login localhost:8080
```

### CIパイプライン

以下の`.gitlab-ci.yml`はアプリのレポジトリに配置しています。

```yaml
stages:
  - build
  - update_manifest
  - open_MR

##############################################################################
##                              Variables                                   ##
##############################################################################
variables:
  APP_NAME: gitops-demo-app  # アプリレポジトリ名
  CI_REGISTRY_IMAGE: <DockerHub user name>/$APP_NAME  # Docker push先のレジストリ名
  CD_PROJECT_ID:   # manifestレポジトリID(GitLabのプロジェクトID)
  CD_CHART_REPO: gitops-demo-chart  # manifestレポジトリ名
  CD_GIT_REPOSITORY:   # manifestレポジトリのsshパス
  CD_MANIFEST_FILE: Chart.yaml  # image tag書き換え対象のmanifestファイル名
  TAG: $CI_COMMIT_REF_NAME-$CI_COMMIT_SHORT_SHA  # 書き換えのtag名

##############################################################################
##                              Build Image                                 ##
##############################################################################
build_image:
  image:
    name: mgit/base:kaniko-executor-debug-stable
    entrypoint: [""]
  stage: build
  before_script:
    - echo $CI_REGISTRY_IMAGE:$TAG $PWD
    # login
    - echo "{\"auths\":{\"https://index.docker.io/v2/\":{\"auth\":\"${DOCKERHUB_TOKEN}\"}}}" > /kaniko/.docker/config.json
  script:
    # Docker Build && Push image
    - cat Dockerfile
    - >
      /kaniko/executor
      --context $CI_PROJECT_DIR
      --dockerfile $CI_PROJECT_DIR/Dockerfile
      --destination $CI_REGISTRY_IMAGE:$TAG
      --build-arg COMMIT_HASH=$CI_COMMIT_SHORT_SHA

##############################################################################
##                              Deployments                                 ##
##############################################################################
update_manifest:
  image: mikefarah/yq:3.3.4
  stage: update_manifest
  variables:
    GIT_STRATEGY: none
  retry: 2
  script:
    # Add SSH key to root
    - mkdir -p /root/.ssh
    - echo "$SSH_PRIVATE_KEY" > /root/.ssh/id_rsa
    - apk add --no-ceche openssh
    - ssh-keyscan -H gitlab.fdev > /root/.ssh/known_hosts
    - chmod 600 /root/.ssh/id_rsa
    # Git
    - apk add --no-cache git
    - git config --global user.name $APP_NAME
    - git config --global user.email $APP_NAME"@gitlab.com"
    - git clone --single-branch --branch master $CD_GIT_REPOSITORY
    - cd $CD_CHART_REPO
    - git checkout -b update-image-tag-$TAG
    # Update Helm image tag
    - >
      yq write
      --inplace --verbose $CD_MANIFEST_FILE appVersion $TAG
    - cat $CD_MANIFEST_FILE
    - git commit -am "update image tag" && git push origin update-image-tag-$TAG
  only:
    - master

open_merge_request:
  image: registry.gitlab.com/gitlab-automation-toolkit/gitlab-auto-mr
  stage: open_MR
  variables:
    GIT_STRATEGY: none
  script:
    # Create merge request
    - >
      gitlab_auto_mr
      --source-branch update-image-tag-$TAG
      --project-id $CD_PROJECT_ID
      -t master
      -c WIP -r
  only:
    - master
```

これでアプリのレポジトリのmasterブランチにpushされると`<branch>-<commit hash>`とtag付けしたimageがbuildされ，DockerHubにpushされ，manifest repoのimage tagの値を更新したMRを自動生成してくれるところまでやってくれます。このパイプラインで直接（manifest repoのmasterブランチの）manifestのimage tagを更新してしまうところまでできるのですが，k8sにdeployする前に一旦人間のチェックが必要かと思い，MRを作成することにしました。

以下でステージごとにやっていることを説明していきます。

### 環境変数の設定

buildしたdocker image のpush先はDocker Hub, また異なるレポジトリ間で操作をしたいため，以下の環境変数を設定した．（`.gitlab-ci.yml`にベタガキは危ないためプレビルドインしておく）

- `DOCKERHUB_TOKEN`: DockerHubにloginするために必要なtoken (`echo -n USER:PASSWORD | base64`で作成)
- `GITLAB_PRIVATE_TOKEN`: CLIでMRを作るために必要
- `SSH_PRIVATE_KEY`: CI上でmanifest repoにアクセスするための秘密鍵

### build stage

```yaml
build_image:
  image:
    name: mgit/base:kaniko-executor-debug-stable
    entrypoint: [""]
  stage: build
  before_script:
    - echo $CI_REGISTRY_IMAGE:$TAG $PWD
    # login
    - echo "{\"auths\":{\"https://index.docker.io/v2/\":{\"auth\":\"${DOCKERHUB_TOKEN}\"}}}" > /kaniko/.docker/config.json
  script:
    # Docker Build && Push image
    - cat Dockerfile
    - >
      /kaniko/executor
      --context $CI_PROJECT_DIR
      --dockerfile $CI_PROJECT_DIR/Dockerfile
      --destination $CI_REGISTRY_IMAGE:$TAG
      --build-arg COMMIT_HASH=$CI_COMMIT_SHORT_SHA
```

CI パイプラインは Docker コンテナ Runner で実行することが一般的なので，パイプラインの中で docker build するには privileged モードで Runner のコンテナを実行する必要があります。いわゆる DinD (Docker in Docker) です。DinDはセキュリティ的に危ないことが知られています。なのでDinDせずにコンテナ内でdokcer buildできる[kaniko](https://github.com/GoogleContainerTools/kaniko)を使うこととします。

`-destination $CI_REGISTRY_IMAGE:$TAG`で`<branch>-<commit hash>`でtag付けしてDockerHubにpushしています。

### update_manifest stage

```yaml
update_manifest:
  image: mikefarah/yq:3.3.4
  stage: update_manifest
  variables:
    GIT_STRATEGY: none
  retry: 2
  script:
    # Add SSH key to root
    - mkdir -p /root/.ssh
    - echo "$SSH_PRIVATE_KEY" > /root/.ssh/id_rsa
    - apk add --no-ceche openssh
    - ssh-keyscan -H gitlab.fdev > /root/.ssh/known_hosts
    - chmod 600 /root/.ssh/id_rsa
    # Git
    - apk add --no-cache git
    - git config --global user.name $APP_NAME
    - git config --global user.email $APP_NAME"@gitlab.com"
    - git clone --single-branch --branch master $CD_GIT_REPOSITORY
    - cd $CD_CHART_REPO
    - git checkout -b update-image-tag-$TAG
    # Update Helm image tag
    - >
      yq write
      --inplace --verbose $CD_MANIFEST_FILE appVersion $TAG
    - cat $CD_MANIFEST_FILE
    - git commit -am "update image tag" && git push origin update-image-tag-$TAG
  only:
    - master
```

CIで一番ややこしいところ． 違うレポジトリ(manifest repo)をcloneしてきてtagの部分のみを上書きしてcommit, pushする作業を行っています。

manifest repoにアクセスするための秘密鍵を登録してレポジトリをclone, tagを更新したのちcommitして`update-image-tag-$TAG`ブランチにpushしています。

tagの更新はyamlのラッパーである[yq](https://github.com/mikefarah/yq)を用いて行っています。

`yq w <yaml_file> <path_expression> <new value>`

で値の更新ができます。

[https://mikefarah.gitbook.io/yq/commands/write-update](https://mikefarah.gitbook.io/yq/commands/write-update)

### open_MR stage

```yaml
open_merge_request:
  image: registry.gitlab.com/gitlab-automation-toolkit/gitlab-auto-mr
  stage: open_MR
  variables:
    GIT_STRATEGY: none
  script:
    # Create merge request
    - >
      gitlab_auto_mr
      --source-branch update-image-tag-$TAG
      --project-id $CD_PROJECT_ID
      -t master
      -c WIP -r
  only:
    - master
```

最後にmanifest repoでMRを自動でopenします．

いい感じのものが作られていたので使わせてもらってます．

[https://gitlab.com/gitlab-automation-toolkit/gitlab-auto-mr](https://gitlab.com/gitlab-automation-toolkit/gitlab-auto-mr)

中身は

[GitLabのMR API](https://docs.gitlab.com/ee/api/merge_requests.html)

を叩いているのですが，この時にprivate_tokenが必要なため，環境変数として`GITLAB_PRIVATE_TOKEN`

を設定しておかなければいけないのがミソかも。

### ArgoCD to Kubernetes

以上まででアプリの更新が行われば，自動でmanifestのimage tagの更新(のMR)が行われるまでできました。

あとはmanifestの更新を検知して自動でk8sにdeployするところをArgoCDでやってもらいます。

```bash
kubectl create namespace gitops-demo # アプリ用のNamespaceを作成

# 今回はCLIで設定したがGUIでも同様の設定が可能
argocd app create webapp \
--repo <manifestrepoのurl> \
--path . \
--dest-server https://kubernetes.default.svc \
--dest-namespace gitops-demo \
--sync-policy automated \ # GitRepoを監視して変更があったら自動更新する設定
--auto-prune \
--self-heal
```

こんな感じでGUIで確認できました。

![image](/uploads/961025c0b8e681292b1b08b47d1be4b0/image.png)

あとはアプリ用に適当にport-forwardさせてあげるとアプリの画面を見ることができました〜！

また，アプリのレポジトリの更新を行うと，パイプラインがまわり，マニフェストのレポジトリにMRが作成されます。そしてmergeを行うと，それをArgoCDが検知してdeployが勝手に走ります。

![image](/uploads/d670b50fe936ed1d629a938a17e04943/image.png)

そしてしばらく待ち（Argo CDは[３分おき（調整可能）](https://argoproj.github.io/argo-cd/operator-manual/high_availability/#argocd-repo-server)にリポジトリの変更をみてデプロイする），deployが完了するとアプリの更新が行えていることが確認できました．簡単！

### ArgoCDのその他機能
ArgoCD(≒k8sがデフォルトで提供する)のdeploy strategyはRollingUpdateなのですが，[Argo Rollouts](https://argoproj.github.io/argo-rollouts/)を使用するとBlue-Green updateやCanary updateなども選択できます。

また，deploy状況の通知関係も[Argo CD Notifications](https://argoproj-labs.github.io/argocd-notifications/)を使えば実現できます。例えばdeployが完了すればSlackに通知するみたいなことも簡単にできます。

以上のようなArgoCDのカスタマイズをしたものをArgoCDでdeployすることもできるのでArgoCDの設定もGit管理できるのも便利だったりします。

最初はGUIがあるのでArgoCDを選択したというのが大きかったのですが，シンプルながらかゆいところに手が届く機能が充実しており，完成度の高いCDツールであると使いながら感じました。


## まとめ
CDの部分よりはCIのところで時間を割いたのでCD部分の検証は不十分ですが，初期設定を除いてアプリレポジトリを更新すれば自動でk8sのデプロイが実現するところまで確認できました。これはとても便利。Gitの管理を行っているので再現などもかなりやりやすくなると思われます。
k8s化することだけでdeploy作業はしやすくなったと社内のエンジニアから聞いていましたが，GitOpsを導入することでより簡潔にできそうです。温かみのある作業を自動化するしてより生産性のある作業に没頭できる時間を増やしていきたいですね。


## 参考

[数時間で完全理解！わりとゴツいKubernetesハンズオン！！ - Qiita](https://qiita.com/Kta-M/items/ce475c0063d3d3f36d5d)

[https://argoproj.github.io/argo-cd/](https://argoproj.github.io/argo-cd/)

[https://medium.com/@andrew.kaczynski/gitops-in-kubernetes-argo-cd-and-gitlab-ci-cd-5828c8eb34d6](https://medium.com/@andrew.kaczynski/gitops-in-kubernetes-argo-cd-and-gitlab-ci-cd-5828c8eb34d6)

[https://argoproj.github.io/argo-cd/user-guide/private-repositories/](https://argoproj.github.io/argo-cd/user-guide/private-repositories/)

[GitLabCI+ArgoCDを使って、「マージしたら5分でKubernetesへデプロイ」を実現する - エニグモ開発者ブログ](https://tech.enigmo.co.jp/entry/2019/12/22/100000)

[GitOps in Kubernetes with GitLab CI and ArgoCD](https://levelup.gitconnected.com/gitops-in-kubernetes-with-gitlab-ci-and-argocd-9e20b5d3b55b)

[CodeBuild で Docker イメージに Git のコミットIDをタグ付けてバージョン管理する | Developers.IO](https://dev.classmethod.jp/articles/docker-image-tag-git-commit-id-by-codebuild/)

[gitops-using-flux-and-gitlab](https://speakerdeck.com/endok/gitops-using-flux-and-gitlab)

## この記事を書いた人

2020年新卒入社。主に大手旅行サイトの開発を行っている。社内を横断してDocker, Kubernetes, DevOps周りの調査/実装などにも励んでいる。

趣味はガジェット収集と山登り。貯金が全く貯まらないのが最近の悩み。
