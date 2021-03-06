# Lab 3 - AKS へのデプロイ

このラボでは Azure Pipelines を使用して AKS にアプリケーションをデプロイします。このラボを実施する前に [Lab 2](https://github.com/hiroyha1/frontapp) を完了してください。

## リポジトリの取得

1. ブラウザを起動し、Azure DevOps の対象のプロジェクトを開きます。
2. Repos の Files を開きます。
3. 画面上部のプロジェクト名が表示されたドロップダウンをクリックし、"Import Repository" を選択します。
4. Clone URL 欄に "https://github.com/hiroyha1/frontapp-deploy" と入力し、"Import" をクリックします。

## Environment の作成

展開先の AKS をリソースとしてもつ Environment を作成します。

1. Pipelines の Environments を開きます。
2. "Create environment" (既に Environment が存在する場合は "New environment") をクリックします。
3. Name 欄に production と入力し、"Kubernetes" にチェックを入れて "Next" をクリックします。
4. "New environment" と表示される画面でデプロイ先の AKS リソースを選択します。リソースが見つからない場合やエラーが発生する場合は、対象の AKS リソースに対して現在のユーザーがクラスター管理者ロールの役割を持っているか確認してください。
5. Namespace 欄では "New" にチェックを入れ、任意の名前空間名を入力します。ここで入力した名前空間が AKS 内に作成されます。

## 解説 - Environment について

Environment は後述する Deployment ジョブで使用するもので、Kubernetes クラスタまたは仮想マシンのリソースの集まりです (リソースを一つも持たない Environment の作成できます)。

Environment を使用することで、次のような機能が実現できます。

- 展開履歴の確認
- 実行されたジョブや展開に紐づいたコミット、ワークアイテムの確認
- リソースの状態の確認
- その Environment にアクセス可能なユーザーやパイプラインの制限
- ジョブの実行前に承認処理を追加

## Variable group の作成

次の変数を持つ "aks-variable-group" という名前の Variable group を作成します。

| 名前 | 値 |
|--|--|
| resource_name | Environment の作成時に指定した名前空間名 |

## deployment.yaml の編集

レポジトリ内の deployment.yaml はサンプルのイメージ名が記載されているため、実際のイメージ名に変更します。

1. Repos の Files を開きます。
2. 現在のリポジトリが frontapp-deploy になっていない場合は、画面上部のプルダウン メニューから frontapp-deploy を選択します。
![1.png](resources/1.png)
3. `manifests/base/deployment.yaml` を開きます。
4. 16 行目のコンテナ イメージを Lab 2 で確認した、Azure Container Registry にプッシュされたイメージ名に変更します。
![2.png](resources/2.png)

変更前:
```yaml
    spec:
      containers:
      - image: acr2210220.azurecr.io/frontapp:102
```

変更後(例):
```yaml
    spec:
      containers:
      - image: myacr.azurecr.io/frontapp0225:153
```

## パイプラインの作成 - その 1

ここでは [Kustomize](https://kustomize.io/) を使用してプロダクション環境用のマニフェスト ファイルを生成します。

1. これまでのラボを同様に frontapp-deploy リポジトリに対して "Starter pipeline" テンプレートの YAML ファイルを作成します。
2. azure-pipelines.yml の内容を次のように書き換えます。
```yaml
trigger:
- master

variables: 
- group: acr-variable-group
- group: aks-variable-group

stages:
- stage: deploy
  displayName: Deploy to production
  jobs:
  - job: bake
    pool:
      vmImage: ubuntu-latest
    steps:
    - task: KubernetesManifest@0
      name: bake
      inputs:
        action: 'bake'
        renderType: 'kustomize'
        kustomizationPath: 'manifests/overlays/production'
    - publish: '$(bake.manifestsBundle)'
      artifact: manifests
```
3. "Save and run" をクリックすると YAML ファイルが保存され、パイプラインが実行されます。
4. ビルドに成功すると、生成されたマニフェストがパイプラインのアーティファクトに登録されます。ファイルは "1 published" と書かれたところをクリックすると確認することができます。
![3.png](resources/3.png)

## 解説 - Kustomize について

Kustomize を使うと、ベースとなる YAML ファイルとその差分の YAML ファイルを結合した YAML ファイルを生成することができます。

本ラボでは `manifests/base` の下に共通の Deployment と Service の YAML を配置し、開発環境向けの差分を `manifests/overlays/development` の下に、プロダクション環境向けの差分を `manifests/overlays/developoment` の下に配置しています。

Kustomize によって、開発、プロダクション向けのマージされた Kubernetes マニフェストがそれぞれ生成されます。

## 解説 - ラボで作成したパイプラインについて

```yaml
variables: 
- group: acr-variable-group
- group: aks-variable-group
```

2 つの Variable group を参照していますが、これらが定義する変数はラボの後半で使用します。

```yaml
stages:
- stage: deploy
  displayName: Deploy to production
  jobs:
  - job: bake
    pool:
      vmImage: ubuntu-latest
```

これまではステージ、ジョブを省略していましたが、ここでは明示的にステージとジョブを記述しています。

```yaml
    steps:
    - task: KubernetesManifest@0
      name: bake
      inputs:
        action: 'bake'
        renderType: 'kustomize'
        kustomizationPath: 'manifests/overlays/production'
```

[Kubernetes manifest](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/kubernetes-manifest?view=azure-devops) というタスクを実行しています。

このタスクでは様々な Kubernetes のマニフェストに関する処理を行うことができますが、ここでは Kustomize によってマニフェストを生成する `bake` アクションを実行しています。

このタスクが実行されると `<タスク名>.manifestsBundle` (ここでは `bake.manifestsBundle`) という名前の変数に生成されたマニフェスト ファイルのパスが格納されます。


```yaml
    - publish: '$(bake.manifestsBundle)'
      artifact: manifests
```

[Publish Pipeline Artifacts](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/publish-pipeline-artifact?view=azure-devops) というタスクを実行しています。`- publish:` という表記は `- task: PublishPipelineArtifact@1` の省略形です。

このタスクが実行されると、前のタスクで生成されたマニフェスト ファイル (パス `$(bake.manifestsBundle)` で指定されたファイル) を `manifests` という名前の Pipeline artifacts として公開します。

Pipeline artifacts は Azure Artifacts とは別もので、パイプライン内のステージ間や異なるパイプライン間などでファイルを共有する目的で使用することができます。

## パイプライン作成 - その 2

1. Pipelines を開きます。
2. "パイプライン作成 - その 1" で作成したパイプライン "frontapp-deploy" をクリックします。
3. "Edit" をクリックします。
4. 元の YAML の内容に続けて、以下の内容を記述します。
```yaml
  - deployment: deploy
    dependsOn: bake
    pool:
      vmImage: ubuntu-latest
    environment: production.$(resource_name)
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            inputs:
              action: 'createSecret'
              namespace: $(resource_name)
              secretType: 'dockerRegistry'
              secretName: 'regcred'
              dockerRegistryEndpoint: $(acr_connection_name)
          - task: KubernetesManifest@0
            inputs:
              action: 'deploy'
              namespace: $(resource_name)
              manifests: '$(Pipeline.Workspace)/manifests/*.yaml'
```
5. "Save" をクリックするとビルドが開始されます。("Run" をクリックしなくても、master ブランチへ azure-pipelines.yml がコミットされたことをトリガーにビルドが開始されます)
6. パイプラインの実行に成功すると、AKS 内の Environment で指定した名前空間に Deployment と Service が作成されます。
7. Pipelines の Environments を開きます。
8. "production" をクリックします。
9. 名前空間名をクリックします。
10. "Workloads" タブに展開された Deployment、"Services" タブに展開された Service が表示され、正常に動作していることを確認します。

![4.png](resources/4.png)

## 解説 - ラボで作成したパイプラインについて

```yaml
  - deployment: deploy
    dependsOn: bake
    pool:
      vmImage: ubuntu-latest
```

[Deployment](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs?view=azure-devops) は特殊なジョブで、主にリソースを展開する目的で使用します。

```yaml
    environment: production.$(resource_name)
```

Deployment は必ず Environment の指定が必要です。ここでは production という名前の Environment の中の AKS リソース (Variable group aks-variable-group で定義された変数 `resource_name` の値) AKS に対するサービス接続を明示的に指定することなく、タスク内で AKS への展開を行うことができるようになります。

```yaml
    strategy:
      runOnce:
        deploy:
          steps:
```

`runOnce` は最も単純な展開戦略で、一括で展開処理を行います。他には 10%, 20%, 100% のように徐々に更新する Pod の数を増やしていく AKS 向けの展開戦略 `canary` や、一度に N% ずつ更新する VM 向けの展開戦略 `rolling` があります。

```yaml
          - task: KubernetesManifest@0
            inputs:
              action: 'createSecret'
              namespace: $(resource_name)
              secretType: 'dockerRegistry'
              secretName: 'regcred'
              dockerRegistryEndpoint: $(acr_connection_name)
```

この Kubernetes manifest タスクでは `secretName` で指定した名前のシークレットを AKS の `namespace` で指定した名前空間内に作成します。ここでは `secretType` に `dockerRegistry` を指定しており、Azure Container Registry に接続するためのシークレットが作成されます。

```yaml
          - task: KubernetesManifest@0
            inputs:
              action: 'deploy'
              namespace: $(resource_name)
              manifests: '$(Pipeline.Workspace)/manifests/*.yaml'
```

最後に前のジョブで作成したマニフェストを AKS 上に展開します。
