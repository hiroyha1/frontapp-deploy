# Lab 3 - AKS へのデプロイ

このラボでは Azure Pipelines を使用して AKS にアプリケーションをデプロイします。このラボを実施する前に [Lab 2](https://github.com/hiroyha1/frontapp) を完了してください。

## リポジトリの取得

1. ブラウザを起動し、Azure DevOps の対象のプロジェクトを開きます。
2. Repos の Files を開きます。
3. 画面上部のプロジェクト名が表示されたドロップダウンをクリックし、"Import Repository" を選択します。
4. Clone URL 欄に "https://github.com/hiroyha1/frontapp-deploy" と入力し、"Import" をクリックします。

## Environment の作成

1. Pipelines の Environments を開きます。
2. "New environment" をクリックします。
3. Name 欄に production と入力し、"Kubernetes" にチェックを入れて "Next" をクリックします。
4. "New environment" と表示される画面でデプロイ先の AKS リソースを選択します。リソースが見つからない場合やエラーが発生する場合は、対象の AKS リソースに対して現在のユーザーがクラスター管理者ロールの役割を持っているか確認してください。
5. Namespace 欄では "New" にチェックを入れ、任意の名前空間名を入力します。ここで入力した名前空間が AKS 内に作成されます。

## deployment.yaml の編集

1. Repos の Files を開きます。
2. 