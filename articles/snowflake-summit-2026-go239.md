---
title: "【Snowflake Summit 2026】AI時代の機密データ保護と監査 ― Horizon Catalogによるデータ&AIガバナンス"
emoji: "🛡️"
type: "tech"
topics: ["snowflake", "datagovernance", "ai", "security", "SnowflakeSummit"]
publication_name: "finatext"
published: false
---

:::message
ナウキャストの Snowflake Summit 2026 参加記の一覧は以下でご覧ください。
https://zenn.dev/finatext/articles/snowflake-summit-2026-summary-nowcast
:::

:::message
本記事は Snowflake Summit 2026 のセッション **「Best Practices for Protecting and Auditing Sensitive Data in an AI-First World（GO239）」**（2026年6月4日）の聴講メモをもとにしたまとめです。内容はセッション時点の情報であり、各機能のリリースステータス（In Dev / Private / Public / GA）は今後変わる可能性があります。
:::

## はじめに

- **セッション**: Best Practices for Protecting and Auditing Sensitive Data in an AI-First World（GO239）
- **登壇者**: Staff Product Manager, Snowflake / Senior Product Manager, Snowflake
- **日時**: Thu, June 4, 10:30 AM – 11:15 AM PDT

AIエージェントが「質問に答える」だけでなく、**自律的にアクションを起こし、機密データにアクセスし、意思決定まで行う**ようになった今、データガバナンスは「あれば良いもの」から「ないと成立しないもの」へと位置付けが変わりました。

本セッションは、Snowflakeのプロダクトマネージャーが、**Horizon Catalog** を軸にした機密データの分類・保護・監査の考え方と、AIワークロードに対する新しいガバナンス機能を一気通貫で解説するものでした。HIPAA・PCI-DSS・CCPA・GDPRといった規制対応を、いかに **インテリジェントな自動化** で現実的に回していくか、というのをテーマにしたものでした。

この記事では、セッションの流れに沿って次の5つの観点で整理します。

1. なぜ今ガバナンスなのか（The World Is Changing）
2. Horizon Catalog の全体像
3. **Discover / Protect / Trust** ― データガバナンスの3本柱
4. **Govern with AI** ― CoCoによる自然言語ガバナンス
5. **Govern for AI** ― AIエージェントそのものをどう統治するか

---

## 1. なぜ今ガバナンスなのか

セッション冒頭では、エージェント時代におけるガバナンスの緊急性が、いくつかの調査データとともに提示されました。

![The World Is Changing](/images/snowflake-summit-2026-go239/IMG_0586.jpg)

- 生成AIに関して **56%** の組織が、ガバナンスやコンプライアンス上の課題を少なくとも1つは抱えている（[Snowflake, 2026 ROI of Gen AI and Agents](https://www.snowflake.com/en/blog/trusted-data-trusted-ai/)）
- **2027年までに、組織の60%** が、ガバナンスフレームワークの整合性の欠如によってAIユースケースの期待価値を実現できないと予測（[Gartner, Data Governance Trends](https://www.gartner.com/en/data-analytics/topics/data-governance)）
- **2027年までに、企業の40%** が、ガバナンスの失敗を理由に自律型AIエージェントを格下げ・廃止すると予測（[Gartner, 2026-05-26 プレスリリース](https://www.gartner.com/en/newsroom/press-releases/2026-05-26-gartner-says-applying-uniform-governance-across-ai-agents-will-lead-to-enterprise-ai-agent-failure)）

AIエージェントが、アクションを実行し、機密データに触れ、意思決定を下す存在になったからこそ、**人間と同じ（あるいはそれ以上の）統治の仕組み** が必要になる、ということです。

---

## 2. Horizon Catalog の全体像

Snowflakeがガバナンスの中核に据えるのが **Snowflake Horizon Catalog** です。Apache Polaris をベースに構築された、データとAIのための単一カタログ（メタデータレイク）であり、**ストレージとコンピュートの間の「契約」** として機能する、という位置付けでした。

![Snowflake Horizon Catalog](/images/snowflake-summit-2026-go239/IMG_0587.jpg)

レイヤー構成はおおむね次の通りです。

- **Compute and AI 層**
  - Snowflakeネイティブのコンピュート（SQL / Data Science & ML / Streamlit Apps / Snowflake CoCo / Snowflake CoWork）
  - 外部コンピュート、外部エージェント
  - 接続インターフェースとして SQL API / REST API / **Iceberg REST Catalog API** / **MCP** を提供
- **Catalog 層（Horizon Catalog 本体）**
  - 横断テーマは Interoperability（相互運用性） / Context（文脈） / Governance and Security（ガバナンスとセキュリティ）
  - 機能群：Search、Lineage、Semantic Views、Data Quality、Access Control、Sensitive Data Protection、AI Security、AI Guardrails
- **Data and Storage 層**
  - Snowflake管理ストレージ（Snowflake Tables / Iceberg）
  - 外部データ（SAP、Salesforce、PostgreSQL、MySQL など）、外部メタデータ（BIツール等）

ここで重要なのは、**MCP や Iceberg REST Catalog API が「外部エージェント・外部エンジンとの接続点」として明示的にカタログに組み込まれている**点です。Snowflake内に閉じない、エージェントや外部レイクハウスを含めた統治を前提にした設計思想がうかがえます。

### ガバナンスの3要素：Discover / Protect / Trust

![Snowflake Horizon Catalog: Governance Built In](/images/snowflake-summit-2026-go239/IMG_0588.jpg)

Horizon Catalogのガバナンスは、3要素で整理されていました。

- **Discover（発見）**：何を持っているかを、設定する前にまず把握する
- **Protect（保護）**：ポリシーがデータと一緒に「移動」し、どこでも効く
- **Trust（信頼／監視）**：データ品質・リネージ・アクセス監査を、人間とAIの双方に対して継続的に監視する

「Classification → Protection → Quality → Lineage → Audit」が Horizon Catalog を中心にぐるりと連なるホイール図で、これらが独立した機能ではなく**相互に連結したもの**として動くことが強調されていました。

以降は、この3本柱に沿って具体機能を見ていきます。

---

## 3. Discover ― 機密データの自動分類

### Automatic Sensitive Data Classification（GA）

![Automatic Sensitive Data Classification](/images/snowflake-summit-2026-go239/IMG_0589.jpg)

まずは「機密データがどこにあるか」。**Trust Center** から **Classification Profile** を作成すると、Snowflakeがプラットフォーム横断で機密データを自動検出します。

- **150種類以上のビルトイン分類器**（PII / PCI カテゴリをカバー）
- 検出した機密カラムに **自動でタグ付け**
- 新しいオブジェクトを見つけるための**日次スキャン**（`auto_tag=true` によるオートタグ）
- **カスタム分類器**で「自社にとっての機密」を定義可能
- GDPR / CCPA / HIPAA / PCI-DSS などのコンプライアンス要件に対応

Trust Center のダッシュボードでは、「レビューが必要なオブジェクト数」「未マスクの機密カラムを持つオブジェクト数」「機密データにアクセスできるユーザー数」「分類エラー」といったメトリクスや、分類カテゴリ別・規制標準別の集計、マスキングステータスが一覧で確認できます。**まず全体像をスコア化して可視化する**ところができるようになります。

---

## 4. Protect ― タグベースでスケールする保護

Discoverで付与した「タグ」を起点に、保護をスケールさせる方針です。

### Tag-Based Masking（GA）

![Protect Sensitive Data with Tag Based Masking](/images/snowflake-summit-2026-go239/IMG_0590.jpg)

- クエリ実行時に**動的にマスキング**するポリシー
- **マスキングポリシーをタグに紐付ける**と、Classificationがカラムにタグを付けた瞬間に自動でマスキングが適用される
- 「1つのポリシーで数千カラムを保護」できる（属性ベースアクセス制御＝ABACの考え方）
- ロールやユーザーコンテキストに応じて、フィールド単位で条件付きマスキングが可能

運用フローは「①ポリシー本体にロジックを定義 → ②マスキングポリシーをタグに関連付け → ③タグが付いたカラムは自動的に保護」というシンプルな3ステップ。**新しく追加されたカラムも、タグさえ付けば自動で守られる**点が効いてきます。

### AI-powered Sensitive Data Protection（Public Preview 近日公開）

![AI-powered Sensitive Data Protection](/images/snowflake-summit-2026-go239/IMG_0591.jpg)

さらに新しいのが、LLMを統合した機密データ保護です。Classification Profile作成時に **AI mode** をオンにすると、

- **数分で**機密データを自動検出（LLM統合）
- 文脈・セマンティックな理解に基づく検出精度（カラム名だけに頼らない）
- 検出 → タグ付け → ポリシー自動適用 を一気通貫で実行

「**Automatically Detect → Automatically Tag → Auto Apply Policy**」の流れで、**タグと関連ポリシーがデータと一緒に移動する**のが核心。データがどこへコピー・共有されても保護が追従します。

### Managing Access at Scale with ABAC（Public Preview 近日公開）

![Managing Access at Scale with ABAC](/images/snowflake-summit-2026-go239/IMG_0594.jpg)

アクセス制御も同じ「タグ＝属性」の発想で統一されています。

- ロールをポリシーにハードコードするのではなく、**タグの属性値**で制御
- `ユーザー属性 → データ属性 → タグ値` の照合
- カラムマスキング・行アクセス・projection・join・aggregation の **全ポリシータイプ** で機能

メリットとして強調されていたのは、**「アクセス変更 = CODEの変更ではなく DATAの変更」** という考え方。ロールやデータが変わってもポリシーを書き直す必要がなく、タグの値を更新するだけで動的に制御できます。

実際のセッションでは、リージョンベースの行アクセスポリシーのSQL例も示されました（要点を簡略化したもの）。

![ABAC Row Access Policy デモ](/images/snowflake-summit-2026-go239/IMG_0595.jpg)

```sql
CREATE OR REPLACE ROW ACCESS POLICY region_rap
  AS (region STRING) RETURNS BOOLEAN ->
  CASE
    -- data_admin ロールはすべて見える
    WHEN IS_ROLE_IN_SESSION('DATA_ADMIN') THEN TRUE
    -- ロールに割り当てられた region タグが 'ALL' なら全行
    WHEN SYSTEM$GET_TAG('abac_demo.governance.region_access', CURRENT_ROLE(), 'ROLE') = 'ALL' THEN TRUE
    -- ロールの region タグが行の region と一致すれば許可
    WHEN SYSTEM$GET_TAG('abac_demo.governance.region_access', CURRENT_ROLE(), 'ROLE') = region THEN TRUE
    ELSE FALSE
  END;
```

> **ポイント**：ポリシー引数名（`region`）がテーブルのカラム名（`region`）と一致していないと、タグ経由でバインドした際に正しく機能しない

![ABAC: アナリストの region_access タグを NA → EMEA に変更するデモ](/images/snowflake-summit-2026-go239/IMG_0596.jpg)

デモでは、アナリストの `region_access` タグを `NA → EMEA` に変更するだけで、ポリシーを一切書き換えずにクエリ結果が動的に絞り込まれる様子が示されました。

---

## 5. Trust ― Access Historyによる監査

保護したら、次は「誰が・いつ・何にアクセスしたか」を追えるようにする番です。

### Data Audit with Access History（GA）

![Data Audit with Access History](/images/snowflake-summit-2026-go239/IMG_0597.jpg)

- 各クエリがアクセスした**テーブル・ビュー・カラムの監査ログ**
- **間接アクセスも捕捉** ― ビューやAIエージェントがベーステーブルを参照した場合、そのベーステーブルへのアクセスも記録される
- `ACCESS_HISTORY` ビューでSQLからクエリ可能、**365日保持**

ユースケースとしては、レポーティング・監査のための履歴クエリ、使われていないテーブル/カラムの発見（削除・アーカイブ判断）、頻繁にクエリされるテーブルの特定（パフォーマンス最適化）など。

```sql
-- 例：ACCOUNT_USAGE.ACCESS_HISTORY を直接クエリ
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY;
```

そして象徴的なのが、**「CoCoに『過去90日間でSSNデータにアクセスしたのは誰？』と聞くだけでよい」** という運用イメージ。SQLを書かずに監査ログへ問い合わせられる世界が示されました。SIEMへのエクスポートによる集中監視にも対応します。

---

## 6. Govern with AI ― CoCoによる自然言語ガバナンス

**ガバナンスの「操作」自体をAIに委ねる**という話です。

### Snowflake CoCo for Data Governance（GA）

![Snowflake CoCo for Data Governance](/images/snowflake-summit-2026-go239/IMG_0598.jpg)

- ガバナンスを**誰でも扱えるようにする**CoCoスキル
- ガバナンス状況の照会、ポリシー作成、監査の実行を、**すべて自然言語チャット**で
- スキルはSnowflakeのドキュメントとアカウントのメタデータを読み、**ユーザーに代わってSQLを生成・実行**

「`ANALYTICS.PUBLIC`スキーマの機密データを分類して」「過去7日間で給与データにアクセスしたのは誰か見せて」といったプレーンな指示で動く、というデモが示されました。**技術的な専門知識なしにガバナンスを回せる**点が価値です。

### Intent-driven Governance with CoCo（Private Preview）

![Intent-driven Governance with CoCo](/images/snowflake-summit-2026-go239/IMG_0599.jpg)

さらに踏み込んだのが、**「意図（intent）を記述すれば、Snowflakeが実装する」** というスキルです。

- プレーンな英語の意図を、完全に統治された状態（governed state）へ変換
- アカウントの現状を理解し、**ギャップを特定して、アクションを実行**
- 分類・タグ・カラム/行ポリシー・ドリフト検知・アラート・監査を横断して動作
- HIPAA / PII などの**ビルトインコンプライアンステンプレート**と**継続的なドリフト監視**

デモでは「Govern my account with HIPAA controls.（HIPAA準拠でアカウントを統治して）」と伝えると、スキルがアカウントをスキャンし、人間が読める"ガバナンス仕様書（Governance Spec）"を生成。CISOが内容を承認すると、保護がワンクリックで一括デプロイされる、という流れでした。仕様はバージョン管理され、監査可能で、いつでも更新できます。

> SQLの専門知識がなくても、意思決定者が「何を守りたいか」を記述すれば、Snowflakeがそれを構築する。セキュリティ管理者は深いSnowflake知識なしに監査できる ― という分業のかたちが提示されていました。

---

## 7. Govern for AI ― AIエージェントそのものを統治する

「Govern **with** AI（AIでガバナンスする）」に対して、「Govern **for** AI（AIワークロードをガバナンスする）」が最後のテーマです。Horizon Catalogを、**すべてのAIエージェント・ツール・データに対する統一AIガバナンスプレーン**として使う、という構想でした。

### Governance for AI Workloads ― 5つの観点

![Governance for AI Workloads](/images/snowflake-summit-2026-go239/IMG_0600.jpg)

- **Identity（識別）** ― 最小権限の委譲アクセスを持つ、専用のエージェントID
- **Inventory（在庫）** ― バージョンベースの昇格ゲートを備えた中央エージェントレジストリ
- **Observability（可観測性）** ― 推論ログ、評価、アクションのリネージ
- **Safety（安全性）** ― プロンプトインジェクションと機密データのガードレール
- **Compliance（コンプライアンス）** ― スペンドキャップ、法的来歴、データ主権

### Agent Identity and Fine-grained Access Control（Private Preview）

![Agent Identity and Fine-grained Access Control](/images/snowflake-summit-2026-go239/IMG_0601.jpg)

エージェントを**ユーザーセッション内で識別可能な一意のAgent Principal**として扱えるようになります。これにより、

- エージェント識別に基づいて、セッションに**ターゲットを絞ったポリシー**を適用（例：機密データへのアクセス制限、読み取り専用化）
- ユーザーに代わってエージェントが実行したアクションの**トレーサビリティ**を確保

```sql
CREATE OR REPLACE MASKING POLICY agent_masked_sensitive AS (val STRING)
RETURNS STRING ->
  CASE
    WHEN SYS_CONTEXT('snowflake$current', 'is_agent_activated') IS TRUE
      THEN '*** MASKED ***'
    ELSE val
  END;

-- マスキングポリシーをタグに割り当てる
ALTER TAG sensitive_data_tag SET MASKING POLICY agent_masked_sensitive;
```

`SYS_CONTEXT('snowflake$current', 'is_agent_activated')` を使えば、**「エージェントが動いているセッションのときだけマスクする」** といった制御ができる点が実務的です。ACCOUNT_USAGEビューでもAgent固有IDでエージェントの行動を追跡できます。

### Cortex AI Guardrails ― プロンプトインジェクション防御（GA）

![Cortex AI Guardrails for Prompt Injection Defense](/images/snowflake-summit-2026-go239/IMG_0602.jpg)

- エージェント入力に対する、**ランタイムかつ文脈認識型**のプロンプトインジェクション検知
- RAGパイプライン・メール・Webコンテンツ・ツールから取得した内容を検査
- **正当な指示と注入された悪意あるペイロードを区別**
- CoCo / Cortex Agents / Snowflake Cowork を保護する統一ガバナンスフレームワーク

```sql
ALTER ACCOUNT SET AI_SETTINGS = $$
  guardrails:
    advanced_prompt_injection:
      - enabled: true
$$;
```

外部コンテンツを扱うエージェントはプロンプト上書きに脆弱であり、これは**セキュリティ監査を受けるエンタープライズAIデプロイにおいて必須**、と位置付けられていました。

### Cortex AI Guardrails ― 機密データ保護（Private Preview）

![Cortex AI Guardrails for Sensitive Data Protection](/images/snowflake-summit-2026-go239/IMG_0603.jpg)

- プロンプトやツールレスポンスに含まれる機密データ（PII / PCI / 各種ID）を検出・保護
- **エージェントログ内の機密データもマスク**
- CoCo / Snowflake Intelligence / Cortex Agents 全体で利用可能

「Customer data estate → Classifiers → Masked input prompt → Guardrail rules → Masked output response / Masked in logs」という流れで、**入力・出力・ログのすべてでマスキング**が効きます。上流のマスキングが不完全だった場合の **安全網（safety net）** として機能する、という説明が印象的でした。

### Governance Dashboard for AI（Private Preview）

![Governance Dashboard for AI](/images/snowflake-summit-2026-go239/IMG_0604.jpg)

そしてこれらを束ねるのが、**AI Governance ダッシュボード**です。

- エージェント・スキル・MCPサーバーにまたがるAIアセットを、**エンドツーエンドで可視化**
- エージェント品質の監視と機密データ露出の防止を同時に実現
- **スコープクリープの検知**（意図したドメイン外のテーブルにアクセスしているエージェントの発見）
- **EU AI Act / NIST AI RMF** といった新しいAI監査要件への対応

![Snowflake CoCo for AI Governance デモ出力](/images/snowflake-summit-2026-go239/IMG_0605.jpg)

デモでは、CoCoに「Evaluate the AI governance posture for my account（アカウントのAIガバナンス状況を評価して）」と指示すると、次のようなサマリが返ってくる様子が示されました（直近7日間）。

| 項目 | 状況 |
| --- | --- |
| Agents（active/total） | 79 / 214 |
| Invocations WoW | +72.2%（198 vs 115） |
| Active users | 13 |
| MCP servers | 5 |
| Idle agents（30日） | 56 |
| Prompt injection guardrails | Enabled |
| Agent-exposed policy gaps | 0（リスクなし） |
| Account-wide policy gaps | 369 / 396（93%）― 高リスク |

そのうえで、優先対応として「①機密データ（369の分類済みオブジェクトにマスキングポリシーがない）②エージェントの整理（56のidle＋135の未実行エージェント）③呼び出し急増（+72%が意図的か確認）④MCPの所有権（5サーバーすべてがACCOUNTADMIN所有 → スコープを絞る）」が提示されました。**「数」ではなく「次に何をすべきか」を返してくれる**のが、従来のダッシュボードと違うところです。

---

## 8. 今日から始められること

![Accelerate your governance journey](/images/snowflake-summit-2026-go239/IMG_0606.jpg)

セッションの締めくくりとして、即効性のあるアクションが3カテゴリで提示されました。

**Discover & Protect Data**
- Trust Center > Data Security > Sensitive Data Classification を有効化する
- アクセス変更が「ポリシー書き換え」ではなく「タグ更新」で済むよう **ABAC** をセットアップする
- Access History クエリをスケジュールし、「先週、機密データに触れたのは誰か」を把握する

**Govern with AI**
- CoCoに「マスキングポリシーが付いていないテーブルはどれ？」と聞いてみる
- **Intent-driven Governance** のPrivate Previewにサインアップする

**Govern for AI**
- **プロンプトインジェクションガードレールを今日有効化する**（GA済み・全アカウント推奨）
- 機密データガードレールのPrivate Previewにサインアップする
- AI Governance DashboardのPrivate Previewにサインアップする

![Accelerate your governance journey — CoCo プロンプト例](/images/snowflake-summit-2026-go239/IMG_0607.jpg)

最後のスライドでは、こんなCoCoプロンプトが示されました。

> "What is my current Data and AI governance posture? Give me a maturity score and top 3 recommendations. Generate maturity report."

ガバナンスの成熟度を自然言語で評価・レポート化できる体験として、セッションが締めくくられました。

---

## まとめ・所感

このセッションを通して一貫していたのは、**「タグを起点に、分類 → 保護 → 監査を自動でスケールさせる」** というSnowflakeの設計哲学でした。Classificationが付けたタグにマスキングやABACポリシーが紐づき、ポリシーがデータと一緒に移動する。この"タグ中心"のモデルがあるからこそ、AIエージェントという新しいアクターに対しても、同じ枠組み（タグ＋ポリシー＋`is_agent_activated`のようなコンテキスト関数）を素直に拡張できている、という構造がよく見えました。
従来Snowflake上では任意アクセス制御（DAC）とロールベースのアクセス制御（RBAC）を中心とした権限管理でしたが、やはりAIでよりガバナンスを柔軟に効かせていくためにはABACを積極的に取り入れる必要があり、そこに舵を切った、というのが個人的には感じたところです。

また、特に注目したいのは次の2点です。

- **エージェントを「一意のAgent Principal」として識別し、人間と同じガバナンス基盤（タグ／マスキング／Access History）で統治する**という方向性。MCPサーバーやエージェントを社内に展開していくうえで、ID・最小権限・監査ログ・スコープクリープ検知をプラットフォーム側で担保できるのは大きい。
- **Intent-driven Governance** と **AI Governance Dashboard** が示す「自然言語で意図を述べ、ギャップ分析と次アクションまでAIが提示する」体験。ガバナンスの担い手を専門家から意思決定者へ広げる狙いがはっきりしている。

一方で、Intent-driven Governance や各種AIガードレール、AI Governance DashboardはまだPrivate Preview段階のものが多く、GA済みのもの（プロンプトインジェクションガードレール、Access History、タグベースマスキング等）と区別して検証しないといけないと思っています。まずはGA機能（Sensitive Data Classification + Tag-based Masking + Access History）で"タグ中心の土台"を固め、その上にAI向けガバナンスを段階的に載せていくのが現実的な進め方だと感じました。

