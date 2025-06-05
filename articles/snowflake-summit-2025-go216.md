---
title: "Snowflakeにおける機密情報の取り扱い セッション参加レポート"
emoji: "❄️"
type: "idea"
topics: ["Snowflake", "security", "dataengineering", "SnowflakeSummit"]
publication_name: "finatext"
published: true
---

:::message
ナウキャストのSnowflake Summit 2025参加記の一覧は以下でご覧ください。

https://zenn.dev/finatext/articles/snowflake-summit-2025-summary-nowcast
:::

## はじめに

Snowflake Summit 2025で開催された「[Best Practices for Protecting Sensitive Data and Simplifying Compliance](https://reg.summit.snowflake.com/flow/snowflake/summit25/sessions/page/catalog/session/1738788106845001TlhP)」セッションでは、機密データを取り扱うケースにおけるデータガバナンスやセキュリティ、コンプライアンスの最新動向と実践例が紹介されました。本記事では、SnowflakeのPdMであるAnkit Gupta氏とNatWestのPrinciple EngineerであるGaurav Kumar氏による発表内容をもとに、Snowflakeで機密情報を取り扱うためのプラクティスや今後の展望、実際の金融機関での取り組みについてまとめます。

:::message
本記事は一部AIによって生成しています。また、セッション内容の紹介部分では、パネリストの発言をもとにしたものを日本語で要約しています。原文の意図と異なる場合がある点、ご了承ください。
:::

## パネリスト紹介
- Ankit Gupta（Senior Product Manager, Snowflake）
- Gaurav Kumar（Lakehouse Principal Engineer, NatWest Group）

## セッション内容

### 機密データ保護とコンプライアンスの重要性

Ankit Gupta氏は、IBMの2024年データ侵害レポートを引用し、データ侵害コストの増大や、特に医療・金融業界での影響の大きさを強調しました。データ侵害の約半数が個人データに関与しており、顧客の信頼喪失が金銭的損失以上に深刻な課題であると指摘しています。

![](https://storage.googleapis.com/zenn-user-upload/aecc901e81dc-20250606.png)

また、データガバナンスの3本柱として
- データの所在と機密性の識別（Know）
- 適切な保護制御の適用（Protect）
- アクセスと保護制御の継続的な監視・監査（Monitor）

が挙げられました。
![](https://storage.googleapis.com/zenn-user-upload/36ad94557166-20250606.png)

### Snowflake Horizonの概要と機能

Horizonカタログは、データガバナンス・セキュリティ・検出を初期段階から実現し、Snowflake内外のデータ全体を最小限のセットアップで保護・管理できると説明されました。
![](https://storage.googleapis.com/zenn-user-upload/206bc5526b79-20250606.png)
https://www.snowflake.com/en/product/features/horizon/ より引用


Horizon Catalogの詳細は以下の記事を参照ください！

https://zenn.dev/finatext/articles/session-report-snowflake-summit-wn209b?redirected=1

今回は機密データの保護に関わる機能を以下で紹介します。

#### 自動機密データ分類

名前や社会保障番号、クレジットカード番号などのPII（個人情報）を自動でスキャン・識別する自動機密データ分類が紹介されました。60以上の既成分類子に加え、カスタム分類子の作成も可能です。識別された機密データは自動でタグ付けされ、保護制御が適用されます。

![](https://storage.googleapis.com/zenn-user-upload/2a3b26059564-20250606.png)
![](https://storage.googleapis.com/zenn-user-upload/320cb128dc16-20250606.png)

この機能はすでにGA（一般提供）となっています。
[公式ドキュメント](https://docs.snowflake.com/en/user-guide/classify-auto)

#### オブジェクトタグ付けとマスキングポリシー

列・テーブル・スキーマ・データベースなどのオブジェクトに重要なメタデータをタグ付けでき、マスキングポリシーと連携することで、ユーザーの役割やデータ属性に応じて動的にデータをマスクできます。タグは自動分類と連動し、保護が自動化されます。

![](https://storage.googleapis.com/zenn-user-upload/4eb582f2ec4f-20250606.png)

こちらもGA機能です。
[公式ドキュメント](https://docs.snowflake.com/ja/user-guide/object-tagging)

#### 行アクセスポリシーと自動タグ伝播

[行アクセスポリシー](https://docs.snowflake.com/ja/user-guide/security-row-intro)により、ユーザー属性やデータ内容に応じて返す行を制御できます。

[自動タグ伝播](https://docs.snowflake.com/en/user-guide/object-tagging/propagation)により、タグがビューやコピー先のテーブルにも伝播し、データの流れ全体で保護が維持されます。


こちらもGA機能です。

:::message
この自動タグ伝播は素晴らしい機能です。上流で機密情報としてtaggingされたものが下流のテーブルにも伝播されるので、データの流れ全体で保護が維持されます。ガバナンスを実現するために非常に重要な機能だと感じました。
:::

#### アクセス履歴と監査

[アクセス履歴機能](https://docs.snowflake.com/en/user-guide/access-history)により、誰がいつ機密データにアクセスしたかを記録・レポートでき、未使用テーブルや列の特定、不要データの削減、リスク低減に役立ちます。

![](https://storage.googleapis.com/zenn-user-upload/4d9a127c051c-20250606.png)

こちらもGA機能です。

### NatWest銀行のデータガバナンス実践

Gaurav Kumar氏は、NatWest銀行でのデータガバナンスの取り組みを紹介しました。
- 1900万人以上の顧客、毎日3000万件以上のトランザクションを処理
- GDPRやPCI DSSなどの厳格な規制対応
- 「データを知る・保護する・監視する」の3本柱でデータメッシュ原則を実装
- 自動データ分類やドメイン専門知識を活用し、情報リポジトリを通じてドメインチームに通知
- テーブルレベルでの粒度管理、プライバシーカテゴリとセマンティックカテゴリの2種類のタグを活用
- アラート機能や継続的な改善サイクルを導入し、外部テーブル（S3やIceberg）も含めて分類・保護を実施


### Snowflake Horizonの今後の機能とベストプラクティス

Ankit Gupta氏は、今後のHorizonの機能強化として
- Trust Center内のデータセキュリティ機能強化
- アカウントレベルでのデータ分類設定
- 機密データインサイトダッシュボード
- 監査・コンプライアンスの標準化
- EUやインド向けの新しい分類子追加
- 新しい列やスキーマ変更時の自動分類トリガー

などが紹介されていました。

![](https://storage.googleapis.com/zenn-user-upload/e09e1c017278-20250606.png)

また、Horizon活用のベストプラクティスとして
- 自動分類の全社展開
- 一貫したタグ付け
- 最小特権の原則に基づくアクセス制御
- 行アクセス・マスキングポリシーの適用
- 機密データへのアクセス監視と変更監視の定期実施

を推奨し、コンプライアンスは継続的なプロセスであると強調されていました。

## まとめ・感想

本セッションを通じて、機密データの保護とデータガバナンスの重要性、Snowflake Horizonの実践的な機能と今後の進化、そして大規模金融機関での現場実装例が具体的に示されました。

最初に述べられていたデータガバナンスの3本柱（Know, Protect, Monitor）を実現しうる、SnowflakeのGA機能で高いレベルのデータガバナンスが実現できることを改めて認識しました（ウォッチできていなかった機能も振り返る良い機会でした）。

また、Xでもpostした通り、NatWestはデータメッシュを導入し、かつデータガバナンスを維持するためにドメインチームと中央のチーム（Hub）で明確に役割分担しているのが印象的でした。これによりアジリティも維持しつつ、守るべきところも守っているということで、機密情報を扱う組織には非常に参考になる事例でした。

https://x.com/mt_musyu/status/1930374963376320582

AIによるより幅広いデータ活用が求められる中、だからこそガードレール的なデータガバナンスが重要になっていくと感じます。自分自身も使えていない機能がたくさんあるので、日本でも事例を増やしていきたいと思います。
