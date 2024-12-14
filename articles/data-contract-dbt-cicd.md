---
title: "Data Contract CLIでデータテストを行い、dbtのschemaを生成する CI/CD"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CICD", "DataContract", "dbt", "GitHubActions"]
publication_name: "finatext"
published: false
---

## 背景
データのスキーマは様々な理由で変更されることがあり、データオーナー側はその変更の影響範囲を正確に把握することが難しいです。

そのため、複数のデータオーナーと安全にソースデータを連携するために、新規ソースデータの連携方法を明確化する必要があります。近年、ソースデータのスキーマ情報の管理やソースデータ監視などの仕組みの必要性が重要視されてきています。

そこでData Contractという概念を取り入れ、ソースデータのスキーマ情報の管理を行うことが最近のトレンドとなっています。

![](https://storage.googleapis.com/zenn-user-upload/1eff97b6177d-20241215.png)
*Data Contract flow https://datacontract.com/*

Data Contractでは、データの送り手（データオーナー）と受け手（データ分析基盤チーム）の間で連携するデータのスキーマ情報を明示的に定義し、その定義を遵守することで連携するソースデータに関する契約を結ぶ概念です（身近な類似している例としては、APIの共有する際に使用するOpenAPI Specification(Swagger)が近いです）。

Data Contractは概念としては素晴らしいですが、実際にどうやって運用していくかが課題だと思います。運用をしていく上で**自動化**は重要だと思っており、今回はData Contractのyamlは手動で作る前提で、Data Contractさえ作れば、データテスト、dbtのschema.ymlの生成を自動化するCI/CDを構築します。

### 実現したいこと
- data contractのyamlをvalidateして、かつ実データに対してテストを行う
- data contractのhtmlを生成してホスティングし、閲覧できるようにする
- data contractのyamlからdbtのschema.ymlを生成する

### 前提
- Data ContractはGitHubで管理する
- データ基盤チームがデータ基盤内で行う加工処理はdbtを使用して、かつdbtのコードは別のGitHubリポジトリで管理する
- CI/CDはGitHub Actionsを使用する
- 自由に使えるAWSアカウントを持っている

## 全体像
今回提示するCI/CDの全体像は以下の通りです。

![](https://storage.googleapis.com/zenn-user-upload/85febb2f06db-20241214.png)

1. (データオーナー)Data Contractを管理するリポジトリに連携したいソースデータのスキーマ情報を入力したYAMLファイルを作成し、GitHubにpush ⇒ Pull Requestを作成する
2. （Data Contract Workflow）Pull Requestが作成されると、GitHub Actionsにより以下が自動で実行される
    1. 作成したYAMLファイルの構文チェックおよび実データとの整合性のチェック
3. (データオーナー)上記が正常に終了後、データ基盤チームをreviewerに指定する
4. (データオーナー)データ基盤チームのapprove後、Pull Requestをマージする
5. （Data Contract Workflow）Pull Requestがmasterブランチにマージされると、自動でdbtが読み込むYAMLファイルとHTMLファイルを生成しS3にファイルをコピー、dbtプロジェクトを管理するリポジトリのワークフローをkickする
6. （dbt Workflow）dbtを管理するリポジトリのワークフローで、5でコピーされたS3のファイルを取得し、Pull Requestを作成する
7. (データ基盤チーム)上記で作成されたPull Requestを編集しパイプラインにソースデータを組み込み反映する。

## 各Stepの詳細

### Data ContractのYAML validation、データテスト
Data Contractのyamlファイルの構文チェックと、実データとの整合性チェックを行う

TBD

### Data ContractのHTML生成
Data Contractのyamlファイルからhtmlファイルを生成し、S3にアップロードする。これにより、データオーナーやデータ基盤チームがData Contractの内容を確認できるようになる。

TBD

### dbtのschema.ymlを取得しPull Requestを作成
Data Contractのyamlファイルからdbtのschema.ymlファイルを生成し、dbtプロジェクトを管理するリポジトリにPull Requestを作成する。これにより、データ基盤チームはソースデータを素早くデータ基盤に反映できるようになる。

TBD

## 今回のCI/CDによる恩恵
- 実データに対する整合性チェックにより、連携されるソースデータの信頼性が担保される
- データ基盤チームは、どんなソースデータが連携されたのか確認しやすい
- データ基盤チームとしてデータの期待する定義が確認しやすいので、データ品質向上のプロセスが踏みやすい
- データ基盤チームは、dbtのスキーマYAMLが半自動生成されるため、ソースデータを素早くデータ基盤に反映しやすい

## 課題点
- Data Contract CLIはまだ開発途中であり、機能が不足している
  - dbtのschema.ymlを生成する機能には不十分感が強い
  - なので、data contract生成からのdbt schema.yml生成までをCI/CDで完結させることは難しい(humanによる手動作業が必要)
- データオーナー側はData ContractのYAMLを作成する必要があるため、データオーナー側の負担が大きい
  - こちらに関しては、Data ContractのYAMLをより簡単に作成できるツール・仕組みがあれば解決できるかもしれない


## まとめ
本記事では、Data ContractとCICD連携によるデータテストとdbt schema生成の自動化について紹介しました。Data Contractの概念は素晴らしいものの、実際の運用には課題もあります。しかし、本CI/CDの取り組みにより、ソースデータの信頼性向上やデータ基盤への反映スピードアップなどの効果が期待できます。今後、Data Contractの利用が広がり、より使いやすいツールが登場することで、データ品質管理の自動化がさらに進むことが期待されます。
