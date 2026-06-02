---
title: '【Snowflake Summit 2026】地方銀行の "AI-Ready" データ基盤 — Peoples Bank（BI202）'
emoji: "🏦"
type: "tech"
topics: ["snowflake", "dataengineering", "dbt", "ai", "datagovernance"]
publication_name: "finatext"
published: false
---

:::message
ナウキャストの Snowflake Summit 2026 参加記の一覧は以下でご覧ください。
https://zenn.dev/finatext/articles/snowflake-summit-2026-summary-nowcast
:::

> ※ 本記事は Snowflake Summit 2026 のセッション
> **「The AI-Ready Bank: How Peoples Bank Unified Core Systems on Snowflake」(BI202)**
> を現地で聴講したメモを元にした参加レポートです。スライドの図と、登壇内容の要約を再構成しています。

### セッション概要（公式）

- **タイトル**: The AI-Ready Bank: How Peoples Bank Unified Core Systems on Snowflake（BI202）
- **登壇**:
  - Priscilla Ellis — Senior Director - Industry Advisor, Coastal
  - Ann Helmick — SVP, Director of Client Experience, Peoples Bank
- **概要**: Most regional banks run on fragmented systems — core banking, Salesforce, nCino — siloing customer data. As AI capabilities accelerate, banks without a unified data foundation risk missing the next wave of agentic automation and intelligence. Peoples Bank of Ohio, a $10 billion+ institution, partnered with Coastal to do something about it. It replaced a legacy SQL registry with Snowflake medallion architecture, used dbt to unify operational systems and turned its general ledger into a trusted customer 360. Get the blueprint: See how the bank delivered golden IDs and automated revenue monitoring and marketing segmentation via zero-copy integration while building an AI-ready foundation for Snowflake.
- **トラック**: BI & Analytics／Breakout Session／45 分／Intermediate
- **対象機能**: dbt Projects on Snowflake、Snowflake Intelligence
- **日時**: 2026年6月1日（月）14:30–15:15 PDT、Room 211（Moscone South, Level 2）
- [公式セッションページ](https://reg.summit.snowflake.com/flow/snowflake/summit26/sessions/page/catalog/session/1768236996669001yCZG)

## はじめに

![タイトルスライド](/images/snowflake-summit-2026-bi202/title-slide.jpg)

Snowflake Summitのセッションの中では新機能使って〇〇%コスト削減！といったようなキャッチーな事例もありますが、淡々とやるべきことをやって成果を出したというセッションもあります。その一つが、今回紹介する **Peoples Bank of Ohio** が、パートナーのCoastalと共同で語った「The AI-Ready Bank: How Peoples Bank Unified Core Systems on Snowflake」です。

テーマは一言でいうと「AI に取り組む前に、AI が乗るデータ基盤を作り直した」という話。Cortex や Snowflake Intelligence といった派手なレイヤーの"下"で、断片化したコアシステムをどう統合し、信頼できる Gold レイヤーまで持っていったか、という非常に地に足のついた内容でした。

登壇は、Peoples Bank の SVP・Director of Client Experience である Ann Helmick 氏と、Coastal の Senior Director - Industry Advisor である Priscilla Ellis 氏。エンジニア向けというより「事業側のオーナーが語るデータ基盤刷新」という構成で、技術選定だけでなく組織・ガバナンス・人の話に踏み込んでいたのが良かったです。

この記事では、

- 何が課題で（データの断片化）
- どう設計し（Medallion + dbt + 4 本柱）
- どんな成果と学びがあったか（Weeks → Hours、そしてガバナンスと人）

を、自分の理解と考察を交えて整理します。

---

## 1. 課題:「Data everywhere. Insight nowhere.」

冒頭で出てきたのがこのフレーズでした。

![Data everywhere. Insight nowhere.](/images/snowflake-summit-2026-bi202/data-everywhere-insight-nowhere.jpg)

> Data everywhere. Insight nowhere.
> — Ann Helmick, Peoples Bank

「どの地方銀行に会っても診断結果は同じ。世界クラスの業務システムが揃っているのに、それらが互いに会話できていない」という問題意識です。データはあるのにインサイトがない、という状態。耳が痛い組織は多いと思います。

### 表向きは「3 つのサイロ」

![Three Silos, One Customer](/images/snowflake-summit-2026-bi202/three-silos-one-customer.jpg)

スライドでは、顧客を分断している代表的なシステムを 3 つに整理していました。

| システム | 役割 | 抱える問題 |
| --- | --- | --- |
| **Banking Systems**（コア） | Transactional Heart（預金・融資・総勘定元帳） | 数十年分の履歴。誰も触りたがらないスキーマ |
| **Salesforce** | Relationship Layer（RM が住む場所） | 顧客が「実際に何を取引しているか」から切り離されている |
| **nCino** | Lending Workflows | 独自の ID 体系・独自のデータオーナー → もう一つの"顧客の真実" |

結果として、**「顧客の単一ビューがない＝Single Source of Truth がない」**。同じ顧客が、システムごとに別 ID・別定義で存在している状態です。

### 実態はもっと深刻だった（~20 サイロ）

実際には合計およそ **20 個のサイロ**にデータが散らばっていたと語られていました。「3 silos, one customer」は聴衆向けの単純化で、M&A を重ねた地方銀行ほど現実はもっと複雑です。

### 断片化の「隠れたコスト」

![The Hidden Cost of Fragmentation](/images/snowflake-summit-2026-bi202/hidden-cost-of-fragmentation.jpg)

スライド "The Hidden Cost of Fragmentation" では、コストを 3 軸で言語化していました。

- **Time（時間）**: アナリストが、たった 1 つのポートフォリオレベルの問いに答えるためだけに、複数システムをまたいで顧客レコードを手作業で突き合わせる
- **Confidence（信頼）**: 2 つのレポートの数字が食い違った瞬間、その数字は使い物にならなくなる。一度崩れた信頼は、回復より崩壊の方が速い
- **Risk（リスク）**: 本当のリスクは「今日のレポートが遅いこと」ではなく、「**Agentic な時代の窓が開いたときに、自社が繋ぎ込めないこと**」

より具体的には「レポートが事業チームに届くまで **2〜3 週間**かかり、待ちきれず手作業で抜き出す → 精度・リスクの懸念が増える」という悪循環が語られていました。

> We weren't worried about AI. We were worried about not being ready for it.
> — Ann Helmick

「AI が怖かったのではなく、AI に乗り遅れることが怖かった」。この一言が、このプロジェクト全体の動機を端的に表しています。

---

## 2. 中間ステップ:Data Registry と暫定 Customer 360

いきなり Snowflake に全部移したわけではなく、Data Registry（データレジストリ）という中間ステップを挟んでいたのが実務的でした。

- SQL ベースのマッチングロジックで暫定的な **Golden ID** を生成し、原始的な Customer 360 を組んだ
- Salesforce + レジストリで「見える化」は進んだが、**トランザクションのカバレッジが欠けていた**
- レジストリは高レベルな見取り図としては有効だが、**スケールせず・取引データ的に不完全**

ただ、このレジストリ作業には副次的に大きな価値がありました。**「どのデータがどこから生まれているのか」を棚卸しせざるを得なくなった**点です。隠れたレポート依存関係やデータの欠損が、ここで初めて表に出てきた。後続の Snowflake への取り込み（Bronze 設計）のためのソースマッピングは、このレジストリ作業が土台になっています。

いきなり理想形を作ろうとせず、「暫定の Golden ID で現状を可視化 → ソースを棚卸し → 本格基盤の設計に反映」という段階移行を取っている点が好印象でした。データ基盤刷新あるあるのビッグバン移行リスクをうまく回避しています。

---

## 3. アーキテクチャ:Medallion（Bronze → Silver → Gold）+ dbt

中核は王道の **Medallion Architecture**。各レイヤーの役割と、そこで何が難しかったかをまとめます。

![The Medallion Architecture](/images/snowflake-summit-2026-bi202/medallion-architecture.jpg)

### Bronze — Raw & Captured

- すべてのソース（Core / Salesforce / nCino / GL …）を**受領したそのままの形**で Snowflake にランディング
- **Immutable & Auditable**（不変・監査可能）

ここはシンプルですが、後述する「ベンダーから生ファイルをもらう」段でハマりどころがありました。

### Silver — Cleansed & Conformed

- dbt モデルで型を強制し、重複排除（dedupe）し、**かつてスプレッドシートに散らばっていたビジネスロジックをここに集約**
- **One definition per concept（概念ごとに定義は 1 つ）**

このプロジェクトで**最も難しかったのは技術ではなく、この Silver 層での「定義合意」だった**と明言していました。

- householding（世帯名寄せ）、revenue（収益）、各種メトリクスの定義を、事業の意思決定者を**早期に巻き込んで**合意する
- 複数の矛盾した計算式が乱立するのを防ぐ
- クロスファンクショナルなワークショップと"強制的な"定例で合意形成。事業部門が変更に抵抗する場面では柔軟に妥協も必要だった

ここは完全に同意で、Silver の本質は SQL ではなく「**誰がメトリクスの意味を決めるのか**」という政治・ガバナンスの問題です。dbt はそれを「versioned / tested / documented」な形で固定化する道具にすぎない。

### Gold — Trusted & Consumable

- **Customer 360**、GL に整合したセグメンテーション、RM 向けビュー
- 「**事業が実際にクエリし、Cortex が読みにいく**」レイヤー

公式の要旨で個人的に注目したのが「**総勘定元帳（GL）を、信頼できる Customer 360 に転換した**」という表現です。CRM や営業データではなく、銀行で最も"硬い" 真実である GL を顧客 360 度ビューのアンカーに据える、という発想。RM 向けの見え方（リレーション）と、会計上の真実（GL）を同じ Gold 上で整合させているからこそ、後述の自動収益モニタリングが成立します。

> Orchestrated with dbt. すべての変換が versioned / tested / documented されているので、Gold レイヤーは **人間にもエージェントにも**信頼される。

「人間にもエージェントにも」という言い回しが、このセッションの通奏低音です。Gold を作る目的が、最初から "BI ダッシュボード" ではなく "エージェントのクエリ対象" として設計されている。

---

## 4. AI-Ready 基盤の「4 本柱」

単なる DWH 移行と一線を画した、と主張する根拠が以下の 4 つの設計判断でした。

![Four Pillars of the AI-Ready Foundation](/images/snowflake-summit-2026-bi202/four-pillars.jpg)


1. **Golden IDs** — 1 顧客 1 ID を Core / Salesforce / nCino 横断で確立。Snowflake ネイティブの名寄せ（ID 解決）が、レガシーな SQL レジストリを置き換える
2. **Zero-Copy to Salesforce** — Snowflake と Salesforce がデータをライブで共有。ETL なし・レプリカなし・ドリフトなし。RM とアナリストが**同じ瞬間に同じ顧客を見る**
3. **Containerized by Source** — 各ソースシステムが Snowflake 上の独立コンテナに存在。**Cost-to-serve をソース単位で追跡** → 「謎のウェアハウス請求」が消える
4. **Cortex Code & Cortex AI from Day One** — AI を後付けしない。Cortex AI を sprint 1 から、エージェント前提の基盤として組み込む

3 番目の「ソース単位のコンテナ化＝ソース単位でのコスト可視化」は地味ですが、運用とコストガバナンスの観点でかなり実践的だと思いました。Snowflake のコスト最適化を語るとき、ウェアハウス単位ではなく「どのソースのデータがいくら食っているか」で見られるのは強い。

---

## 5. 設計思想:「AI-enabled. Not AI-curious.」

個人的にいちばん持ち帰りたいフレーズがこれでした。

![AI-enabled. Not AI-curious.](/images/snowflake-summit-2026-bi202/ai-enabled-not-ai-curious.jpg)

> AI-enabled. Not AI-curious.

「多くの銀行は AI を後付け（retrofit）している。Peoples はそうしない」。Gold レイヤー、Golden ID、Salesforce 同期 — **すべてのピースが『いずれエージェントがクエリする』前提で作られている**、と。

Ann Helmick 氏の言葉を要約すると、ゴールは「RM に "AI ツールを使わせる" ことではない」。RM は**すでに使っているシステムの中で質問を投げ、答えが自社データに根ざした形で、自社の Snowflake 環境の中から返ってくる** — それが理想だ、と。

これは UX の話に見えて、実はアーキテクチャの話です。「AI ツールを開く」体験ではなく「いつものシステムが賢くなる」体験を成立させるには、その裏に Golden ID と Gold レイヤーと Zero-Copy 同期が揃っていないといけない。**"curious"（興味本位で触ってみる）と "enabled"（基盤として組み込む）の差は、ほぼデータ基盤の差**だという主張です。

### 「AI Angle の下にある Foundation」

別スライドでは、Native AI の前提条件を 3 点に整理していました。

![The Foundation Behind the AI Angle](/images/snowflake-summit-2026-bi202/foundation-behind-ai-angle.jpg)


- **Native AI in a secure environment**:Cortex は Snowflake のガバナンス環境内で動く。**データが壁の外に出ない**
- **AI Performs Best on Trusted Data**:Cortex は不完全なデータでも動くが、セマンティックモデル・明確なビジネス定義・クリーンな入力に支えられたときに真価を発揮する
- **PEBO Chose a Platform, not just a Warehouse**:基盤を流し込んだ後に AI を後付けするのではなく、初日から組み込み

> The AI opportunity is real, but it is only as strong as the foundation underneath it.

「AI の機会は本物だが、その強さは下のファウンデーションの強さに等しい」。Summit のキーノートが上のレイヤーの話で盛り上がるほど、この "下" の地味な話の価値が際立ちます。

---

## 6. 成果:From Weeks to Hours

ビフォーアフターのスライドはシンプルでした。

![From Weeks to Hours](/images/snowflake-summit-2026-bi202/from-weeks-to-hours.jpg)

「すべての下流を解き放つ運用上のシフト」として、以下の成果が示されていました。

- **Golden IDs in production**:レガシーな SQL マッチングを廃止。Core / Salesforce / nCino 横断の単一 ID グラフ
- **自動収益モニタリング（Automated revenue monitoring）**:GL を整合させた Gold をベースに、収益のモニタリングを自動化。手作業のスコアカード／インセンティブ集計から解放される
- **Customer 360 + segmentation operational**:マーケが行動ベースのセグメントを Gold レイヤーに直接実行 — **チケットなし・エクスポートなし**（Zero-Copy 連携で実現）
- **Architecture ready for Snowflake Intelligence**:Intelligence が成熟するにつれ、必要なデータが既に clean・governed・所定の場所にある状態

ビジネスインパクトとしては、手作業の突合せ削減、手戻り削減、運用リスク低減、そして**顧客対応や戦略に使える時間が増える**こと。「2〜3 週間」が「数時間」になる、というのが象徴的なメッセージでした。

---

## 7. いちばん効いた話:ガバナンスと「人」

技術セッションのつもりで入ったら、後半は完全に**組織・ガバナンス・人**の話で、ここが本セッションの白眉でした。

### Start With Governance Clarity（2 つの助言）

![Start With Governance Clarity](/images/snowflake-summit-2026-bi202/start-with-governance-clarity.jpg)


1. **Don't Wait for a Perfect Strategy** — 完璧な戦略を待つな。まず**ガバナンスの明確化**から:実効性のある RACI、名前のついたデータオーナー、そして「**メトリクスの意味を誰が決めるのか**」の合意。これだけで 1 年分のツール議論が前に進む
2. **Enablement Is Harder Than the Tech** — **組織のアラインメントは技術より難しい**。規制当局・取締役会・失敗プロジェクトからのプレッシャーが来る"前"にやれ。来てからでは遅い。技術は、人の準備ができたときに後から着地する

### もっと生々しい学び

- **プロジェクト途中で CDO（Chief Data Officer）を採用**したことがスコープを変え、成熟を一気に加速させた。**社内に専門性を持つ（あるいは専門パートナーを持つ）ことが必須**で、外部ベンダーに丸投げではダメ
- **ベンダーから生ファイル（raw file）を取得するのに、想定外の実装費用・月額費用がかかる**ことがある。**ソースデータの取得コストを予算化せよ**。これは Bronze 層を作るうえで地味に致命的なハマりどころ
- 実践ガイド:
  - 各データ要素が**どこから来て、どう届くか**（フラットファイル / レポート / API）をマッピングし検証する
  - **事業の意思決定者を早期に巻き込み**、定義に合意してバイインを得る
  - **ソース単位でコンテナ化**し、変換を Bronze → Silver → Gold と段階的に重ねる

### AI ガバナンスのスタンス

AI の現状利用はまだ限定的で、戦略は「**AI を人の代替ではなく、手作業を消すツールとして使う**」（プロフィール要約、インサイト抽出など）。

さらに現実的だったのが、**従業員がパブリックな無料 AI ツールに流れるのを防ぐ**ために、安全な社内向け AI ユースケースと統制を提供する、という発想。シャドー AI 対策を「禁止」ではなく「社内に良い代替を用意する」で解こうとしている点は、規制業種らしい賢い割り切りでした。

---

## 8. 考察 — エンジニアとして持ち帰ったこと

セッションの主張を自分の言葉で整理すると、こうなります。

**「AI-Ready とは、エージェントが信頼してクエリできる Gold レイヤーを、ガバナンス込みで先に作っておくこと」**

派手な Agentic デモは、その下に Golden ID・セマンティックモデル・Zero-Copy 同期・dbt で固めた lineage がなければ、結局 "AI-curious" で終わる。Cortex / Snowflake Intelligence の話が魅力的に聞こえるほど、評価すべきは「そのクエリ対象の Gold が本当に信頼できるか」だ、と改めて思いました。

特に刺さった 3 点:

- **Silver の本質は SQL ではなく「定義の所有権」**。`one definition per concept` を技術で固定する前に、人で合意する。dbt は合意の固定化装置
- **コストをソース単位で見る**設計（Containerized by Source）。FinOps 文脈でそのまま転用できる
- **シャドー AI を "良い社内代替" で抑える**ガバナンス。禁止ベースの統制より持続可能

そして、自分が普段考えている「**非エンジニアが自然言語で、ガバナンスされたデータに安全にアクセスする**」というテーマと、このセッションの結論はほぼ同じ方向を向いていました。エージェントが安全に動く前提は、結局「信頼できる Gold + 明確なセマンティクス + 壁の外にデータが出ないガバナンス」に集約される。AI ゲートウェイ／MCP 的な"接続層"の議論も、この Foundation が無ければ砂上の楼閣になる、というのが今回の最大の学びでした。

もう一つ印象的だったのは、アメリカの地方銀行が抱える課題が、日本の金融機関の現場と驚くほど重なっていた点です。断片化したシステム、サイロ化した顧客データ、レポートまでの長いリードタイム——これは特定の国や規模の問題ではなく、金融業界に共通する構造的な難しさだと改めて感じました。派手な新技術より、定義の合意・ガバナンスの整備・段階的な移行といった当たり前のことを粘り強く続けることが結局は近道だ、という教訓は、自分たちの仕事にもそのまま当てはまります。日本の金融でも面白い事例を作れる手応えが少しずつ出てきているので、引き続き取り組んでいきたいと思います。

---

## まとめ

- 課題は **データの断片化**(~20 サイロ)。コストは Time / Confidence / Risk の 3 軸で顕在化していた
- 解は **Data Registry（暫定）→ Snowflake Medallion（Bronze → Silver → Gold）+ dbt**
- AI-Ready の正体は **4 本柱**（Golden IDs / Zero-Copy to Salesforce / Containerized by Source / Cortex from Day One）と「**AI-enabled, not AI-curious**」という設計思想
- 成果は **Weeks → Hours**、Customer 360、セルフサービス分析
- 最大の学びは技術ではなく、**ガバナンスの明確化・CDO 採用・ソースデータ取得コスト・定義の合意**といった「人と組織」の話

「AI の強さは、その下のファウンデーションの強さに等しい」。地味ながら、本質を突いたセッションでした。

---

*本記事はセッション聴講メモを元にした個人の参加レポートです。数値・固有名詞・引用は聴講時のメモに基づくため、正確性は公開資料をご確認ください。*
