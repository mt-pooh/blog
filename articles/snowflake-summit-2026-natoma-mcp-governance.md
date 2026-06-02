---
title: "【Snowflake Summit 2026】Snowflakeが買収した「Natoma」— エンタープライズMCPガバナンス設計を読み解く"
emoji: "🛡️"
type: "tech"
topics: ["snowflake", "mcp", "ai", "natoma", "governance"]
publication_name: "finatext"
published: false
---

:::message
ナウキャストの Snowflake Summit 2026 参加記の一覧は以下でご覧ください。
https://zenn.dev/finatext/articles/snowflake-summit-2026-summary-nowcast
:::

> Snowflake Summit のセッション **「Governing the Agentic Enterprise: Meet Natoma, the Enterprise MCP Platform (GO105)」** の参加レポートです。スライドと登壇内容をもとに、Snowflakeが買収を発表したエンタープライズMCPプラットフォーム Natoma のアーキテクチャとガバナンス設計を、MCPゲートウェイを自作している立場から読み解きます。

## はじめに

- **セッション**: Governing the Agentic Enterprise: Meet Natoma, the Enterprise MCP Platform（GO105 / Theater Session / 20分 / Introductory）
- **登壇者**: Paresh Bhaya（Co-Founder, Natoma） / Ryan Bradley（Head of Product, Natoma）
- **日時/場所**: Tuesday, Jun 2, 1:30–1:50 PM PDT、Basecamp South Theater 1

![Natoma is joining Snowflake](/images/snowflake-summit-2026-natoma/IMG_0457.jpg)

AIエージェントは、もはやデータを「クエリ」するだけの存在ではありません。メールを送り、レコードを更新し、ワークフローを起動する — つまり**アクションを取る**段階に入りました。しかし多くのエンタープライズには、エージェントが「実際に何をできるか」を統制するガバナンス基盤がまだ存在しません。

このギャップを埋めるのが Natoma で、Snowflakeは2026年5月27日にその買収意向を発表しました。タイミングも象徴的で、同じ日にSnowflakeはFY2027 Q1の決算ビートと、AWSへの大型コミットメントもあわせて公表しています。「データウェアハウス企業」から「エージェンティック・エンタープライズのコントロールプレーン」への明確なピボットを示す動きと言えます。

なお Natoma は2024年設立、従業員約27名、2025年5月にシード資金を調達したスタートアップです。CEO兼Co-Founderの Pratyus Patnaik は、Oktaにオペレーション基盤 atSpoke を売却した経歴を持つ元Okta幹部で、チームはMCP・ゲートウェイ基盤・アイデンティティガバナンス・特権アクセス管理（PAM）に強みを持ちます。このバックグラウンドは、後述のアーキテクチャの随所に効いています。

## なぜ今「エージェントのガバナンス」なのか

![Enterprises haven't met the AI moment](/images/snowflake-summit-2026-natoma/IMG_0458.jpg)

**「Enterprises haven't met the AI moment（エンタープライズはまだAIの瞬間に応えられていない）」** と題して、課題を3つの軸に整理していました。

1. **Disconnected apps, env & data** — アプリ・環境・データのサイロ化
2. **Security & Access** — セキュアなアクセス制御の欠如
3. **Governance & Auditability** — ガバナンスと監査可能性の欠如

それぞれに実顧客のコメントが添えられていて、現場のリアルな温度感が伝わってきます（要約・意訳）。

- 全社にChatGPT Enterpriseを展開したが、得られたのは「少しマシなメールとドキュメント」だけで、ゲームチェンジには至っていない
- オンプレ（Vertica）に大量のデータが眠っている。LLMからアクセスできるようにしてくれるなら予算は出す
- エンジニアが思い思いにMCPサーバーを立てており、セキュリティ上の時限爆弾になっている
- 「AIで何かやれ」というプレッシャーはあるが、強いガバナンスがなければ単なるカオス。もっとパイロットではなく、コントロールが欲しい
- Asana の MCP サーバーを止めた。エージェントがマネージャー向けのタスクを作成し、全員にタスクが露出してしまった

最後の一件は、ガバナンスのないツール接続が**実害（情報露出）**に直結した典型例です。「エージェントはアクションを取る」という前提に立つと、ツール呼び出し（tool call）レベルでの許可/拒否・監査が必須になる、というのがNatomaの問題設定です。

## Natomaとは：エージェンティック・エンタープライズのガバナンス・ファブリック

登壇者のParesh氏は、Natomaを「最も高い5万フィートの視点で言えば**MCPガバナンス・プラットフォーム**」と表現しました。エンタープライズの全アプリケーションの**前段（front）に立つファブリック**として、あらゆるシステムにインテリジェンスを接続する役割です。

ポイントは「AIを置き換えない」設計思想です。既存のSaaS・データセット・アプリはそのままに、**顧客が選んだAI（Claude / Microsoft Copilot / 任意のモデル）**を、Salesforceやオンプレ、各種DBへガバナンス付きで接続する。Snowflake環境ではCortex CodeやCoco（Cortex Intelligence、現COWORK）と深く統合されますが、それに閉じないマルチモデル前提が強調されていました。

「100+ のMCPサーバーを最初から（out of the box）」「検証済み（verified）のコネクタライブラリ」を持つ点が、自前でサーバーを立てるカオスに対する差別化軸です。

## アーキテクチャと主要機能

### 1. Connect Intelligence to Every System

![Connect Intelligence to Every System](/images/snowflake-summit-2026-natoma/IMG_0459.jpg)

中核コンセプトのスライドでは、左に各種MCPクライアント/モデル、中央にNatoma、右にエンタープライズシステム群（DB、SharePoint、SAP、GitHub等）が描かれ、Natomaがハブとして両者を束ねていました。チェックポイントは3つ。

- **Based on MCP & A2A Protocol**（MCPとAgent-to-Agentプロトコル準拠）
- **Flexible deployment**（柔軟なデプロイ）
- **Security & access controls**（セキュリティとアクセス制御）

### 2. Meet your data where it lives（デプロイの柔軟性）

![Meet your data where it lives](/images/snowflake-summit-2026-natoma/IMG_0460.jpg)

「データのある場所でデータに会いに行く」というスライドが、個人的には最も実装的な見どころでした。Natomaは**Self-hosted / Cloud / Desktop** のいずれの形態でも動作し、クライアントからは複数のエンドポイント形態で接続できます。

- `mcp.natoma.app/mcp`（クラウド、OAuth経由）
- `acme-internal.com/mcp`（自社ドメインでのセルフホスト）
- `localhost:8080/mcp`（デスクトップ）

そして、エージェントへは**データを大量に移動させるのではなく、適切なコンテキストだけを渡す**思想が明確でした。「データの移動は高くつく」というコスト観点は、運用規模が大きいほど効いてきます。

### 3. Agent Identity / IAM

Natomaは**AIエージェントの中央リポジトリ**を持ち、各エージェントにアイデンティティを付与します。Paresh氏はこれを「いわばAIのIAM」と表現していました。単にIDを与えるだけでなく、「そのエージェントが何と会話できるか」を統制する点が肝です。

### 4. Policy Engine（ポリシーエンジン）

![Policy Engine — Access画面](/images/snowflake-summit-2026-natoma/IMG_0466.jpg)

デモのAdmin画面（Access）では、ポリシーの作成・編集が示されました。象徴的だったのが **`Block Jira ticket creation`** というポリシーです。

- **Policy Type**: Tool Access Control / Content Validation の2種
- **適用対象（What does the policy apply to?）**: All actors
- **制御対象（What does it control access to?）**: 2 apps, 23 tools
- そのほか `Allow all tools for Slack / Snowflake / Playwright / Dropbox / ServiceNow / Zoom / Figma` などのポリシーが並ぶ

デフォルトは「全ツール許可」で、そこに**ブロック系のポリシーを重ねて最小権限へ寄せていく**モデルです。rate limit の設定や、ツール単位での許可/拒否、コンテンツバリデーションも備えます。

### 5. Observability / Audit（監査ログ）

![Activity Log — 全体ビュー](/images/snowflake-summit-2026-natoma/IMG_0467.jpg)

Activityビューでは、すべてのエージェント操作が**検証可能なトレース**として残ります。

- どのツールがどんな引数で呼ばれたか（tool calls / arguments）
- 接続先（Snowflake / Jira / ServiceNow など）
- アクター（**人間ユーザー** か、独自IDを持つ**プログラム的エージェント** か）
- IPアドレス、開始/終了時刻、実行結果（Executed / Client error / Policy denial）

クライアント情報（例：`coco-mcp-client (0.1.0)`）やモデル情報まで紐づいており、「誰が・どのクライアントで・何を実行/拒否されたか」が一目で追えます。

## デモ：Coco × Natoma

ライブデモ（Ryan氏）では、Coco（Snowflakeのエージェント体験）からNatoma経由でMCPサーバーに接続する流れが示されました。

1. Snowflakeウェアハウス（サンプルデータ）とJiraにあらかじめ接続済み
2. その場で **Slack** 接続を追加（ワンクリックセットアップ）
3. Cocoを起動すると「MCPサーバーの認証が必要」と表示され、**OAuthフロー**へ遷移（認証は**Okta等のエンタープライズIDP**がバック）

![Activity Log — Policy denial の様子](/images/snowflake-summit-2026-natoma/IMG_0468.jpg)

会場Wi-Fiの不調でライブ部分は途中までになりましたが、Activityログ側で**ポリシーが効いている様子**がはっきり確認できたのが収穫でした。`coco-mcp-client` のあるセッションでは:

- `getAccessibleAtlassianResources` → **Executed**
- `getVisibleJiraProjects` → **Client error**（×2）
- `createJiraIssue` → **Policy denial**（複数回）

まさに先ほどの `Block Jira ticket creation` ポリシーが、Jiraのチケット作成だけをツール呼び出しレベルで弾いている、というデモになっていました。一方でSnowflakeに対する `sql_exec_tool` は連続して **Executed** されており、許可/拒否が意図どおりに分岐していることが分かります。

![Activity Log — IT Agent（AWS Agent Core）](/images/snowflake-summit-2026-natoma/IMG_0469.jpg)

また、別のActivityでは `genesis-gateway (1.0.0)` 上の **IT Agent - AWS Agent Core** が **ServiceNow Managed connection** / **AWS Documentation Connection** に対して動作しており、**人間ユーザーとプログラム的エージェントを区別して**監査できる点も示されていました。

## ケーススタディ

### Case Study 1: HR Productivity（AI Recruiting Agent）

![Case Study: HR Productivity](/images/snowflake-summit-2026-natoma/IMG_0461.jpg)


- **課題**: 採用チームが毎週、面接パネルの調整に多くの時間を費やしていた。役割ごとに「誰を面接委員に入れるか」「いつ空いているか」「すでに面接過多になっていないか」を毎回判断する必要があった
- **解決**: Natomaのプラットフォーム上で、**AWS Agent Core** を基盤にHR Recruiting Agentを構築し、Greenhouse（ATS）・Google Workspace（Calendar）・Slack に接続。空き枠の検出・スケジュール調整・Slack通知を自動化
- **効果**: スケジューリングに費やすリクルーター時間を **40%削減**

### Case Study 2: Real-World Applications（サプライチェーン）

![Driving Results: Real-World Applications](/images/snowflake-summit-2026-natoma/IMG_0462.jpg)


- **コンセプト**: 「汎用チャットボット」ではなく、**実運用データに接地したChatGPTをワークフロー内に**置く
- **課題**: オペレーションチームが、分断されたシステムをまたいで例外を手作業でトリアージしており、対応が後手に回っていた
- **解決**: AI例外コパイロットが ERP（SAP）・WMS・TMS・履歴データを統合し、問題が顧客やマージンに影響する前に表面化・優先度付け・解決
- **効果**: 例外処理時間を **40〜60%削減**

いずれも「データを動かさず、コンテキストとアクションをガバナンス付きで束ねる」というNatomaの主張を、定量効果で裏付ける構成でした。

## Snowflakeにとっての戦略的意味

Snowflakeの公式メッセージは一貫していて、**ガバナンスの境界を「データアクセス」から「AIによるアクション」へ拡張する**というものです。13,000社超のエンタープライズが「ポリシー適用・セキュリティ・アクセス制御」を信頼してデータを預けてきた基盤に、エージェントが**データを操作する**時代の統制を持ち込む、という位置づけです。

買収後は、Cortex Agents / Snowflake Intelligence / Cortex Code / Coco から、SaaS・クラウド・VPC・オンプレにまたがるシステムへ、検証済みMCPサーバー群を通じてセキュアに接続できるようになります。SnowflakeのSridhar Ramaswamy氏は決算コールで、Snowflake IntelligenceやCocoを離れることなくメール送信・Slack要約・カレンダー確認・Jira起票ができる点に触れつつ、「重要なのは利便性ではなくコントロールだ」と強調しています。

裏を返せば、これは**Snowflakeがエージェントのオーケストレーション/コントロールプレーンの座を狙う**という宣言でもあります。

## 考察 — MCPゲートウェイを自作している立場から

私は同じくMCPゲートウェイ（MCPass / AI Connect Hub）をAWS Bedrock AgentCore Gateway 上で開発しているので、Natomaの設計には強く共感する点と、改めて考えさせられる点がありました。

- **共通項が多い**: OAuth 2.1ベースの認証フロー、ユーザー単位のトークン委譲、ツール呼び出しレベルのポリシー（NatomaのTool Access ControlはABAC的なポリシーエンジンの発想で、自分がCedarで実現しようとしている領域と重なる）、interceptorログを一次ソースとした可観測性 — これらは「エンタープライズMCPゲートウェイ」を作るなら避けて通れない共通の論点だと再確認しました。
- **AWS Agent Core の存在感**: ケーススタディ（HR Recruiting Agent）にもActivityログ（IT Agent - AWS Agent Core / `genesis-gateway`）にも Bedrock AgentCore が登場していたのが印象的でした。ゲートウェイとAgent Coreの責務分担をどう切るかは、自分の設計でも引き続き検討したい点です。
- **差別化は「検証済みコネクタの品揃え」**: 「100+ MCPサーバーを out of the box」「verified library」は、技術的な目新しさより**運用負荷とセキュリティリスクの肩代わり**として効く差別化要素だと感じます。1社で同等のカタログを維持するのは現実的でないので、ここはプラットフォーム/エコシステムの戦いになりそうです。
- **アクター識別と監査の粒度**: 人間ユーザーとプログラム的エージェントを区別し、IP・クライアント・モデル・引数まで残す監査設計は、コンプライアンス文脈で強い説得力を持ちます。自作ダッシュボードでも「一次ソース＝interceptorログ、安全網＝CloudWatch」の方針で同等の粒度を狙っているので、検証/拒否の表示UI（Executed / Client error / Policy denial）は良い参考になりました。

一方で、CIOアナリストが指摘していたように「MCPをプラグ&プレイの魔法と捉えるのは危険」で、**最小権限・監査トレース・高リスク操作のhuman-in-the-loop承認・データ漏洩対策・責任の所在**といった論点は、ゲートウェイを入れても自動で解決するわけではありません。ここはツールを入れる側の設計責任として残り続けます。

## まとめ

- Snowflakeは2026年5月27日、エンタープライズMCPプラットフォーム **Natoma** の買収意向を発表
- Natomaは**集中型MCPゲートウェイ**として、エージェントのID付与・ポリシー適用・監査を**ツール呼び出しレベル**で実現する
- デモでは `Block Jira ticket creation` ポリシーが `createJiraIssue` を拒否しつつ、Snowflakeへの `sql_exec_tool` は許可する様子が確認できた
- 狙いは、Snowflakeのガバナンス境界を**データ → アクション**へ拡張し、エージェンティック・エンタープライズの**コントロールプレーン**になること
- MCPゲートウェイを作る側にとっては、OAuth・委譲・ポリシー・可観測性という共通論点と、「検証済みコネクタの品揃え」という差別化軸を改めて突きつけられるセッションだった

---

*※本記事はセッションのスライドと登壇内容、および公開されているプレスリリース・報道をもとに筆者がまとめたものです。製品仕様は今後の統合により変更される可能性があります。*
