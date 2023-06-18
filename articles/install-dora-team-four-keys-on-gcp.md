---
title: "dora-team/fourkeysを使ってFour Keysの計測環境を構築する"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [fourkeys, devops]
published: true
published_at: 2023-06-19 07:00
---

[dora-team/fourkeys](https://github.com/dora-team/fourkeys)を利用して、GCP 上に Four Keys を計測する環境を構築します。
GCP や Terraform を使うのは初めてです。リポジトリ内の Installation guide に沿って、環境を構築しています。

## 始める前に

- GitHub 上でリポジトリを Fork
- Google Cloud プロジェクトを作成
- プロジェクトで自分にオーナーロールを割り当てる
- ローカルマシンに Google Cloud CLI と Terraform をインストール

## Terraform でデプロイする

Cloud Build を使用して、ダッシュボード・イベントハンドラーのコンテナをビルドしてプッシュする。

```bash
export PROJECT_ID="YOUR_PROJECT_ID"
gcloud builds submit dashboard --config=dashboard/cloudbuild.yaml --project $PROJECT_ID && \
gcloud builds submit event-handler --config=event-handler/cloudbuild.yaml --project $PROJECT_ID
```

Cloud Build を使用して、GitHub のパーサーのコンテナをビルドしてプッシュする。

```bash
gcloud builds submit bq-workers --config=bq-workers/parsers.cloudbuild.yaml --project $PROJECT_ID --substitutions=_SERVICE=github
```

`terraform.tfvars` を編集し、必要な変数に値をセットする。

- **project_id**: GCP 上で作成したプロジェクトの ID
- **region**: asia-northeast1
- **bigquery_region**: US
- **parsers**: ["github"]

変数の説明やデフォルト値は `variables.tf` を参照すれば良い。

`example` ディレクトリで次のコマンドを実行

```bash
terraform init

terraform plan
Plan: 1 to add, 0 to change, 0 to destroy.
╷
│ Error: Invalid provider configuration
│
│ Provider "registry.terraform.io/hashicorp/google" requires explicit configuration. Add a provider block to the root module and configure the provider's required arguments as described in the provider documentation.
│
╵
╷
│ Error: Attempted to load application default credentials since neither `credentials` nor `access_token` was set in the provider block.  No credentials loaded. To use your gcloud credentials, run 'gcloud auth application-default login'
│
│   with provider["registry.terraform.io/hashicorp/google"],
│   on <empty> line 0:
│   (source code not available)
│
│ google: could not find default credentials. See https://cloud.google.com/docs/authentication/external/set-up-adc for more information
╵
```

エラーメッセージに書かれている URL にアクセスして gcloud CLI の認証する。

```bash
terraform plan

terraform apply
```

Terraform でデプロイが完了

## モック用のデータを作成

```bash
export WEBHOOK=`gcloud run services list --project $PROJECT_ID | grep event-handler | awk '{print $4}'`

export SECRET=`gcloud secrets versions access 1 --secret=event-handler --project $PROJECT_ID`

python3 data-generator/generate_data.py --vc_system=github
40 changes successfully sent to event-handler

bq query --project_id $PROJECT_ID 'SELECT * FROM four_keys.events_raw WHERE source = "githubmock";'
+-------------------+------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------------------------------+------------------+------------+
|    event_type     |                    id                    |                                                                                                   metadata                                                                                                   |    time_created     |                   signature                   |      msg_id      |   source   |
+-------------------+------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------------------------------+------------------+------------+
| deployment_status | c9bbf5c828c0c0a8596c193fd4a715361f4627a7 | {"deployment_status": {"updated_at": "2023-06-16 10:37:54.579162", "id": "c9bbf5c828c0c0a8596c193fd4a715361f4627a7", "state": "success"}, "deployment": {"sha": "06a8c11bc631ae17168fa3d227bf549f8b735f61"}} | 2023-06-16 10:37:54 | sha1=41acf5f9900e51c5f6b4115cd9c40da168d66085 | 8404948585377987 | githubmock |
+-------------------+------------------------------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------+-----------------------------------------------+------------------+------------+
```

## ダッシュボードを確認

`fourkeys-grafana-dashboard` というダッシュボード用の Cloud Run に URL が記載されているので、アクセスした管理画面のメニュー `Home > Dashboards > Four Keys` でダッシュボードを表示

![](/images/install-dora-team-four-keys-on-gcp/four-keys-grafana-dashboard.png)

## おわりに

すんなりとインストールすることができました。
GitHub から生のデータを取得し、コードと合わせて、Four Keys の計測ルールを解析したいです。
