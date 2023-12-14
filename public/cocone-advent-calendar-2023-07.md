---
title: GitHub Actions + Cloud Deploy で Cloud Run にデプロイする
tags:
  - GitHubActions
  - googlecloud
  - CloudDeploy
private: false
updated_at: '2023-12-07T07:02:04+09:00'
id: b0ff2e8178118bef256e
organization_url_name: cocone_inc
slide: false
ignorePublish: false
---
## はじめに

この記事は [Cocone Advent Calendar 2023](https://qiita.com/advent-calendar/2023/cocone) 7日目の記事です。

GitHub Actions と Google Cloud Deploy を使用して Cloud Run へデプロイします。おおまかな流れは以下の図のようになります。

<img width="600" alt="cloud_deploy_000.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/46427/932cb0a7-cf5c-6c94-0755-01a9b5298c7b.png">

この図を説明すると、

* リポジトリの `main` ブランチにプッシュ(かプルリクエストをマージ)されるとワークフローが実行され、 Cloud Deploy の環境とステージング用 Cloud Run サービスが作成(デプロイ)されます。(①②③)
* そのあと、 Cloud Deploy のプロモートを行うと本番用の Cloud Run サービスへデプロイされます。(④⑤)

本番環境をデプロイするまでに手動で行うことは①のプッシュと④のプロモートだけになります。

## GCP の設定

GitHub から GCP へアクセスさせるため準備を行います。 `gcloud` コマンドの初期設定と必要な API の有効化は済んでいるものとして割愛します。また、プロジェクト ID とプロジェクト番号の表記をそれぞれ `${GCP_PROJECT_ID}` と `${GCP_PROJECT_NUM}` としてあります。それらは実際の値に置き換えます。

### IAM サービスアカウント

GitHub から接続するためのサービスアカウントを作成します。

```bash
gcloud iam service-accounts create my-sa-gha
```

もう1つ、 Cloud Run 実行時に指定するサービスアカウントを作成します。ただ今回 Cloud Run のコンテナでは特別なことはしないのでこのアカウントには何も権限を付与しません。

```bash
gcloud iam service-accounts create my-sa-run
```

### Workload Identity 連携

Workload Identity 連携を設定します。これを行うことでサービスアカウントのシークレットキーを発行することなく GitHub から GCP へアクセスできるようになります。シークレットキーの漏洩による事故がなくなり、セキュリティが強化されます。詳しくは [こちら](https://docs.github.com/ja/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform) や [こちら](https://github.com/google-github-actions/auth) を参考にしてください。

Workload Identity プールを作成します。

```bash
gcloud iam workload-identity-pools create my-wi-pool --location="global"
```

GitHub Actions の OIDC プロバイダを作成します。

```bash
gcloud iam workload-identity-pools providers create-oidc my-oidc-gha \
  --location="global" \
  --workload-identity-pool="my-wi-pool" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository"
```

サービスアカウントと Workload Identity プールを接続します。この `${USERNAME}/${REPOSITORY}` は実際の GitHub ユーザー名/リポジトリに置き換えます。

```bash
gcloud iam service-accounts add-iam-policy-binding my-sa-gha@${GCP_PROJECT_ID}.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${GCP_PROJECT_NUM}/locations/global/workloadIdentityPools/my-wi-pool/attribute.repository/${USERNAME}/${REPOSITORY}"
```

### IAM カスタムロール

Cloud Deploy の操作に必要な権限を持つカスタムロールを作成します。権限を記述したファイルを作成し、それを指定してカスタムロールを作成します。 Cloud Run サービスの作成は別のサービスアカウントになり代わって行われるため、ここではそれらの権限は付与していません。

```yaml:permissions.yaml
title: myCloudDeployRole
stage: GA
includedPermissions:
- clouddeploy.deliveryPipelines.create
- clouddeploy.deliveryPipelines.get
- clouddeploy.deliveryPipelines.update
- clouddeploy.operations.get
- clouddeploy.releases.create
- clouddeploy.releases.get
- clouddeploy.rollouts.create
- clouddeploy.rollouts.get
- clouddeploy.rollouts.list
- clouddeploy.targets.update
- iam.serviceAccounts.actAs
- storage.buckets.create
- storage.buckets.get
- storage.buckets.list
- storage.objects.create
```

ロールを作成します。

```bash
gcloud iam roles create myCloudDeployRole --project="${GCP_PROJECT_ID}" --file="permissions.yaml"
```

サービスアカウントにロールを割り当てます。

```bash
gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
  --member="serviceAccount:my-sa-gha@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="projects/${GCP_PROJECT_ID}/roles/myCloudDeployRole"
```

### Artifact Registry

コンテナイメージのプッシュ先となる Artifact Registry のリポジトリを作成します。

```bash
gcloud artifacts repositories create my-app --repository-format="docker" --location="asia-northeast1"
```

この Artifact Registry リポジトリを操作する権限をサービスアカウントに付与します。

```bash
gcloud artifacts repositories add-iam-policy-binding my-app \
  --location="asia-northeast1" \
  --member="serviceAccount:my-sa-gha@${GCP_PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"
```

## GitHub の設定

リポジトリの変数とシークレットを設定します。ここの `${GCP_PROJECT_ID}` と `${GCP_PROJECT_NUM}` も実際の値に置き換えます。

|種類|名前|値|
|:-|:-|:-|
|変数|`APP`|`my-app`|
|変数|`GAR_LOCATION`|`asia-northeast1`|
|シークレット|`GCP_PROJECT_ID`|`${GCP_PROJECT_ID}`|
|シークレット|`RUN_SERVICE_ACCOUNT`|`my-sa-run@${GCP_PROJECT_ID}.iam.gserviceaccount.com`|
|シークレット|`WIF_PROVIDER`|`projects/${GCP_PROJECT_NUM}/locations/global/workloadIdentityPools/my-wi-pool/providers/my-oidc-gha`|
|シークレット|`WIF_SERVICE_ACCOUNT`|`my-sa-gha@${GCP_PROJECT_ID}.iam.gserviceaccount.com`|

Git で管理するファイルを用意します。
各種ファイルは [このリポジトリのサンプル](https://github.com/google-github-actions/example-workflows/tree/main/workflows/create-cloud-deploy-release) をベースに作成しています。

* main.go
* Dockerfile
* cloud-deploy.template.yaml
* skaffold.template.yaml
* app-stg.template.yaml
* app-prod.template.yaml
* .github/workflows/cloud-deploy-to-cloud-run.yml

それぞれのファイルを説明していきます。

### main.go

Hello world を返すだけのシンプルなHTTPサーバーです。確認のために環境変数も表示しています。 `TARGET` は Stg か Prod 、 `SHA` には Git のコミットハッシュが入ります。


```go:main.go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, world %s %s\n", os.Getenv("TARGET"), os.Getenv("SHA"))
}

func main() {
	http.HandleFunc("/", hello)
	http.ListenAndServe(":8080", nil)
}
```

### Dockerfile

こちらもビルドと実行だけのシンプルなものです。


```dockerfile:Dockerfile
FROM golang:1.21.4-alpine AS builder

COPY main.go ./
RUN CGO_ENABLED=0 go build -trimpath -ldflags='-w -s' -o /hello main.go

FROM scratch
COPY --from=builder /hello /

USER 65534:65534
EXPOSE 8080
CMD ["/hello"]
```

### cloud-deploy.template.yaml

Cloud Deploy デリバリーパイプラインとターゲットを作成するための構成ファイルのテンプレートです。他に設定できる項目については [こちらのページ](https://cloud.google.com/deploy/docs/config-files?hl=ja) を参照してください。

また、構成ファイルやテンプレート内にある `${APP}` のようなシェル変数の記述は置き換えずそのままにします。これらはワークフロー実行時に置き換わります。テンプレートはステップ途中で `envsubst` コマンドで展開され `cloud-deploy.yaml` のような名前で構成ファイルが生成されます。

```yaml:cloud-deploy.template.yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: '${APP}'
description: 'Deployment pipeline for ${APP}'
serialPipeline:
  stages:
    - targetId: 'stg'
      profiles: ['stg']
    - targetId: 'prod'
      profiles: ['prod']
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: 'stg'
description: 'Staging target'
run:
  location: 'projects/${PROJECT_ID}/locations/${REGION}'
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: 'prod'
description: 'Production target'
run:
  location: 'projects/${PROJECT_ID}/locations/${REGION}'
```

### skaffold.template.yaml

Cloud Deploy リリースを作成するための Skaffold 構成ファイルのテンプレートです。デプロイ先を Cloud Run にしてステージングと本番用のCloud Run サービス構成ファイルを指定しています。設定できる項目は [こちらのページ](https://skaffold.dev/docs/references/yaml/) にあります。

```yaml:skaffold.template.yaml
apiVersion: skaffold/v4beta7
kind: Config
metadata:
  name: '${APP}'
deploy:
  cloudrun: {}
profiles:
  - name: 'stg'
    manifests:
      rawYaml:
        - 'app-stg.yaml'
  - name: 'prod'
    manifests:
      rawYaml:
        - 'app-prod.yaml'
```

### app-stg.template.yaml と app-prod.template.yaml

ステージング環境と本番環境の Cloud Run サービス構成ファイルのテンプレートです。 `image: 'app'` は Cloud Deploy リリースを作成するステップ実行時に置き換えられます。 設定できる項目は [こちらのページ](https://cloud.google.com/run/docs/reference/yaml/v1) が参考にしてください。

```yaml:app-stg.template.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: '${APP}-stg'
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: '1'
        run.googleapis.com/execution-environment: gen2
    spec:
      serviceAccountName: ${RUN_SERVICE_ACCOUNT}
      containers:
      - name: '${APP}'
        image: 'app'
        env:
          - name: 'TARGET'
            value: 'Stg'
          - name: 'SHA'
            value: '${SHA}'
```

```yaml:app-prod.template.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: '${APP}-prod'
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: '1'
        run.googleapis.com/execution-environment: gen2
    spec:
      serviceAccountName: ${RUN_SERVICE_ACCOUNT}
      containers:
      - name: '${APP}'
        image: 'app'
        env:
          - name: 'TARGET'
            value: 'Prod'
          - name: 'SHA'
            value: '${SHA}'
```

### .github/workflows/cloud-deploy-to-cloud-run.yml

GitHub Actions ワークフローの構成ファイルです。


```yaml:.github/workflows/cloud-deploy-to-cloud-run.yml
name: Cloud Deploy to Cloud Run

on: 
  push:
    branches:
      - main

jobs:
  deploy:
    permissions:
      contents: 'read'
      id-token: 'write'

    runs-on: ubuntu-latest
    steps:
      - name: 'Checkout'
        uses: 'actions/checkout@v4'

      - name: 'Google auth'
        id: 'auth'
        uses: 'google-github-actions/auth@v1'
        with:
          workload_identity_provider: '${{ secrets.WIF_PROVIDER }}'
          service_account: '${{ secrets.WIF_SERVICE_ACCOUNT }}'

      - name: 'Set up Cloud SDK'
        uses: 'google-github-actions/setup-gcloud@v1'
        with:
          project_id: '${{ secrets.GCP_PROJECT_ID }}'

      - name: 'Docker auth'
        run: |-
          gcloud auth configure-docker ${{ vars.GAR_LOCATION }}-docker.pkg.dev

      - name: 'Build and push container'
        run: |-
          docker build -t "${{ vars.GAR_LOCATION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ vars.APP }}/${{ vars.APP }}:${{ github.sha }}" .
          docker push "${{ vars.GAR_LOCATION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ vars.APP }}/${{ vars.APP }}:${{ github.sha }}"

      - name: 'Render templatised config manifests'
        run: |-
          export PROJECT_ID="${{ secrets.GCP_PROJECT_ID }}"
          export REGION="${{ vars.GAR_LOCATION }}"
          export APP="${{ vars.APP }}"
          export SHA="${GITHUB_SHA::7}"
          export RUN_SERVICE_ACCOUNT="${{ secrets.RUN_SERVICE_ACCOUNT }}"
          for template in $(ls *.template.yaml); do envsubst < ${template} > ${template%%.*}.yaml ; done

      - name: 'Create Cloud Deploy delivery pipeline'
        run: |-
          gcloud deploy apply --file cloud-deploy.yaml --region ${{ vars.GAR_LOCATION }}

      - name: 'Create release name'
        run: |-
          echo "RELEASE_NAME=${{ vars.APP }}-${GITHUB_SHA::7}-${GITHUB_RUN_NUMBER}" >> ${GITHUB_ENV}

      - name: 'Create Cloud Deploy release'
        id: 'release'
        uses: 'google-github-actions/create-cloud-deploy-release@v0'
        with:
          delivery_pipeline: '${{ vars.APP }}'
          name: '${{ env.RELEASE_NAME }}'
          region: '${{ vars.GAR_LOCATION }}'
          description: '${{ env.GITHUB_COMMIT_MSG }}'
          skaffold_file: 'skaffold.yaml'
          images: 'app=${{ vars.GAR_LOCATION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ vars.APP }}/${{ vars.APP }}:${{ github.sha }}'

      - name: 'Report Cloud Deploy release'
        run: |-
          echo "Created release ${{ steps.release.outputs.name }} "
          echo "Release link ${{ steps.release.outputs.link }} "
```

このワークフローで行われていることはおおまかに、

* 認証
* コンテナをビルドして Artifact Registry へプッシュ
* テンプレートから構成ファイルを生成
* Cloud Deploy デリバリーパイプライン作成
* Cloud Deploy リリース作成

となっています。

これらの7つのファイルが `main` リポジトリへプッシュされるとワークフローの実行を通して、 Cloud Deploy デリバリーパイプラインとリリース、ステージング用の Cloud Run サービスがデプロイされます。

## GCP のリソースを確認

まだ本番用の Cloud Run サービスはデプロイされていません。 Cloud Deploy デリバリーパイプラインのプロモートを行うことによってデプロイされます。

Cloud Deploy デリバリーパイプラインの Web UI 画面では以下の画像のようになっています。プロモートをクリックして進行します。

<img width="480" alt="cloud_deploy_001.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/46427/afd3e17a-6ed7-cf95-f0f6-ad474f58d6ef.png">


プロモートが完了すると以下のようになり本番用の Cloud Run サービスがデプロイされます。

<img width="480" alt="cloud_deploy_002.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/46427/e05391a8-7048-1e2a-ed5b-a75a362060ae.png">


:::note
作成された Cloud Run サービスへのアクセスは制限されています。必要に応じて開放します。例えばインターネットに公開する場合は以下のようにコマンドを実行します。

```bash
gcloud run services add-iam-policy-binding my-app-prod \
  --region="asia-northeast1" \
  --member="allUsers" \
  --role="roles/run.invoker"
```
:::

実際にアクセスするとこのように表示されます。

<img width="480" alt="cloud_run_001.png" src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/46427/262ea643-0710-ea13-e306-61677904f7fd.png">


## おわりに

GitHub Actions + Cloud Deploy を使用して Cloud Run へデプロイを行いました。

これで `main` ブランチにプッシュされるたびにステージング環境へデプロイが行われ、本番環境へのデプロイは Web UI から数クリックで行えるようになりました。ロールバックしたい時も同じく Web UI から数クリックで行えます。

今回は試していませんが Cloud Deploy デリバリーパイプラインには他にも、複数のターゲットに並列にデプロイする、プロモート時に承認を設ける、カナリアデプロイ、自動化の構成などもあり、まだまだ色々なことができそうで興味深いです。


## 参考リンク

* [GitHub Actions と Google Cloud Deploy が連携 \| Google Cloud 公式ブログ](https://cloud.google.com/blog/ja/products/devops-sre/using-github-actions-with-google-cloud-deploy)
* [GitHub Actions を使用して Cloud Run にデプロイする \| Google Cloud 公式ブログ](https://cloud.google.com/blog/ja/products/devops-sre/deploy-to-cloud-run-with-github-actions)
* [GitHub \- google\-github\-actions/example\-workflows: Repository to demonstrate example workflows\.](https://github.com/google-github-actions/example-workflows)
* [Google Cloud Platform での OpenID Connect の構成 \- GitHub Docs](https://docs.github.com/ja/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform)
* [GitHub \- google\-github\-actions/auth: A GitHub Action for authenticating to Google Cloud\.](https://github.com/google-github-actions/auth)
* [Cloud Run サービスまたはジョブをデプロイする  \|  Cloud Deploy  \|  Google Cloud](https://cloud.google.com/deploy/docs/run-targets?hl=ja)
*  [デリバリー パイプライン、ターゲット、自動化構成  \|  Cloud Deploy  \|  Google Cloud](https://cloud.google.com/deploy/docs/config-files?hl=ja)
*  [skaffold\.yaml \| Skaffold](https://skaffold.dev/docs/references/yaml/)
* [Cloud Run YAML Reference  \|  Cloud Run Documentation  \|  Google Cloud](https://cloud.google.com/run/docs/reference/yaml/v1)
