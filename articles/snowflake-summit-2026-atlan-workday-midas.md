---
title: "【Snowflake Summit】AIエージェントの精度を決めるのは「コンテキスト」— Atlan と Workday MIDAS"
emoji: "🧠"
type: "tech"
topics: ["snowflake", "ai", "mcp", "datacatalog", "llm", "SnowflakeSummit"]
published: false
---

:::message
ナウキャストの Snowflake Summit 2026 参加記の一覧は以下でご覧ください。
https://zenn.dev/finatext/articles/snowflake-summit-2026-summary-nowcast
:::

## はじめに

Snowflake Summit で「AIエージェント × コンテキスト」をテーマにした 2 つのセッションを聴いてきました。データカタログの **Atlan** が提示する *AI Context Layer* の構想と、**Workday** が自社の社内分析エージェント **MIDAS** をどう「信頼できる」ものに仕上げたかという実装事例です。

両者に共通していたメッセージはとてもシンプルで、

> **エージェントの精度を決めるのは、モデルでもプロンプトでもなく「コンテキスト」である**

ということでした。しかもここで言うコンテキストは、いわゆる「セマンティックレイヤー」と同義ではなく、もっと広い概念として整理されていたのが面白かったので、スライドを追いながら整理します。

:::message
**セッション情報**
「[Context Layer in Practice: How Workday Builds Trustworthy AI Agents](https://reg.summit.snowflake.com/flow/snowflake/summit26/sessions/page/catalog/session/1768236981007001yQGd)」（GO209）
登壇者：Jon Chen（Director, Enterprise AI/ML Solutions, Workday）、Austin Kronz（Director of AI and Data Strategy, Atlan）
2026年 6 月 3 日 12:30–12:50 PDT

本記事はスライドと聴講メモをもとにした要約です。一次情報として引用される際はセッション資料をご確認ください。
:::

---

## そもそも「コンテキスト」とは何か

まずAtlanのAustin 氏がメンションしていたのが

> 「コンテキストという言葉が、単なるセマンティックレイヤーの言い換えとして使われがち」

という点でした。Atlan のセッションでは、コンテキストを **3 つの要素の重なり** として定義していました。

![AI Analyst Agent — KNOWLEDGE / EXPERTISE / NORMS のベン図](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0543.jpg)

| 要素 | 一言で言うと | 中身 |
| --- | --- | --- |
| **Knowledge（知識）** | What things mean／ビジネスの地図 | エンティティ・定義・メトリクス・リレーション。「会社が自分自身を語るときのオントロジー」 |
| **Expertise（専門性）** | How work gets done／手続き的ノウハウ | プレイブック、ワークフロー、タスク分解。「ビジネスを回す実務スキル」 |
| **Norms（規範）** | What is allowed／許される行動のルール | ポリシー、権限、マスキング、承認フロー。「エージェントを安全に保つガードレール」 |

この 3 つが重なった部分こそが **Context** だ、という整理です。重要なのは、**この多くが暗黙知（tacit）でドキュメント化されていない**こと。だからこそエージェントはドメインの問いに正しく答えられない、と問題提起していました。

### 「Why is drive-through time up this week?」という罠

具体例として挙げられていたのが、ファストフード店をイメージした次の一見シンプルな問いです。

> **Why is drive-through time up this week?**（今週ドライブスルーの所要時間が伸びているのはなぜ？）

これに答えるだけで、エージェントには複数種類のコンテキストが必要になります。

- **Knowledge**：「drive-through time」は `avg_dt_secs` を指し、Finance が使う POS の数値ではない
- **Knowledge**：「this week」は店舗ローカル時間の月〜日（日曜 23:59 締め）であって、ローリング 7 日間ではない
- **Norms**：店長は自店舗のデータしか見られないが、VP Ops はチェーン全体を見られる（ペルソナスコープ）
- **Norms**：同じ「ドライブスルー時間」でも、店長は `avg_dt_secs`、Finance は POS メトリクスを見る（メトリクスの権威性）

人間のアナリストなら無意識にこれらの前提を補完していますが、これらが明文化されていなければエージェントは平気で誤った推論をします。**「時間窓の定義」や「指標の出典」といった暗黙の前提を、どこまで機械可読な形で書き下せるか**が肝になる、というわけです。

---

## Atlan が描く「AI Context Layer Architecture」

次のスライドが、コンテキストをプラットフォームとして扱うための全体像です。

![The AI Context Layer Architecture](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0544.jpg)

左から右への流れで整理されていました。

### 1. Context Mining（コンテキストの採掘）

ソースとなる各種システムからコンテキストを抽出します。

- Systems of Record
- Systems of Data
- Systems of Knowledge
- Systems of Work
- Runtime Signals

### 2. Context Foundation（コンテキストの基盤）

採掘したものを 3 つの構成要素に整理します。

- **AI-Ready Data & Knowledge Graph**：企業のデータと知識資産を AI が扱える形で表現したもの
- **Semantics & Ontology**：共有された意味・関係・ビジネスロジック
- **Skills**：「どう仕事をするか」を記述した、再利用可能・バージョン管理可能・テスト可能なユニット

この基盤を支えるのが **Context Development Lifecycle** で、`Build → Test → Review → Approve → Deploy → Learn` というソフトウェア開発さながらのループになっています。前半（Build/Test/Review）は **AI がブートストラップ・シミュレート**し、後半（Approve/Deploy/Learn）は **人間が曖昧さを解消し、暗黙知を加え、承認する**という役割分担です。

### 3. Compounding Learning Loop（複利的に効く学習ループ）

> Memory, feedback, and traces create self-improving context

メモリ・フィードバック・トレースが、コンテキストを自己改善させていく、という考え方。エージェントからのフィードバックでコンテキスト層を磨き込んでいくのがポイントでした。

### 4. Context Activation（コンテキストの活性化）

最終的に、コンテキストを様々な経路でアプリに供給します。

- Search / APIs / SQL / MCP / Vector Retrieval / Hybrid Assembly
- 供給先：Copilots / Agents / Analytics / Workflows / Enterprise Apps

そして全体を貫く **Context Governance & Observability**（Context Quality / Drift / Lineage / Versioning）。**コンテキストを「コードのように」バージョン管理し、ドリフトを観測し、リネージを持たせる**という主張は、MCP ゲートウェイや AI 基盤を運用する立場からも非常に共感できる整理でした。

---

## Workday MIDAS：「信頼できるエージェント」をどう作ったか

ここからは Workday のセッション「How We Build Trustworthy Agents」。登壇者は Jon Chen（Director, Enterprise AI/ML Solutions, Workday）氏です。

![Workday — How We Build Trustworthy Agents](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0545.jpg)

### MIDAS とは

![What is MIDAS?](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0546.jpg)

**MIDAS = Modular Insight Delivering Agentic Solution**。Workday は HR・Finance・給与計算を 6,000 万人分管理するプラットフォームです。そのスケールで稼働する社内 AI 分析エージェントがMIDAS で、ポイントは以下です。

- **Snowflake Cortex 上に構築**
- Sales / Marketing / Financial のデータドメイン（opportunities、MQL、performance、ARR など）を分析対象にする
- **SQL 不要** — データの民主化を狙う

そして「どうやって精度を担保するか？」については、

- ナレッジドメインごとに **Semantic View** を用意し、検証済みクエリと eval でテストする
- **Context Layer** で、Semantic View を **Atlan 上の認証済み定義（certified definitions）** に揃える
- **Semantic Search** で、プロンプトとユーザーコンテキストから最適なドメインを判定する

### コンテキストの「レイヤーケーキ」

Workday の整理がうまかったのがこの図。「セマンティックレイヤー 1 枚では足りない（A single technical semantic layer is not enough）」という主張です。

![The Knowledge and Context "Layer Cake"](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0548.jpg)

| レイヤー | 内容 | Human in the loop |
| --- | --- | --- |
| **03 Tacit Context** | アナリストがデータをどう解釈するか。フォローアップの問いや判断 | **人間が必要** |
| **02 Structural Context** | データが実際にどう使われるか。セグメントルール、計算ロジック、関係性など、セマンティック層が拾いきれないドメイン固有のニュアンス。グラフとして表現し、人間なしでも推論・実行可能に | **人間なしで可** |
| **01 Technical Context** | メトリクス・定義・スキーマ。標準的なセマンティックレイヤーが提供するもの | — |

Atlan の Knowledge / Expertise / Norms とも対応関係が見える整理で、**「機械が自律で扱える層」と「最後まで人間の判断が要る層」を明示的に分けている**のが実践的でした。

### 理想アーキテクチャ

![The Ideal Architecture](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0547.jpg)

4 層構成で示されていました。

1. **User Interface Applications**：MIDAS チャット、Workday Sana、その他のエージェント UI
2. **Context Layer**：Workday + Atlan + Snowflake／Context Agent (Engineering Studio)、Metadata Lakehouse、Semantic Views
3. **Agentic Orchestration**：**LangGraph / Strands + A2A / MCP**
4. **Activation / Execution Platforms**：
   - **Analyze**（何が起きた？）
   - **Synthesize**（それは何を意味する？／何をすべき？）
   - **Act**（どのエージェントにタスクを振る？）

オーケストレーション層に LangGraph / Strands と並んで **A2A / MCP** が据えられているのが象徴的でした。

### 「クリスマスツリーについて聞いてみた」

このセッションで一番会場が沸いたのがこのスライドでした。

![We asked it about Christmas trees.](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0549.jpg)

Jon 氏が MIDAS にクリスマスツリーについて聞いてみた、というデモです。返答はこうでした。

> Workday のビジネスデータにクリスマスツリーの情報はありません。具体的なビジネス上の問いがあれば喜んでお手伝いします！

笑いが起きつつも、スライドに書かれたひと言が刺さりました。

> **Knowing what you don't know is the feature.**（自分が知らないことを知っているのが「機能」だ）

コンテキストが効いているから、MIDAS はドメイン外の問いに対して自信満々に答えない。「データがありません」と言えること自体が、ちゃんとコンテキストで境界が引けている証拠です。ハルシネーションを起こさないことをバグではなく仕様として設計する、という話は当たり前に聞こえますが、実際にそれを本番で実現するのがどれだけ大変かは、エージェントを触ったことがある人なら分かると思います。

### 現在地とこれから

![Where We Are — and Where We're Going](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0550.jpg)

- **Live**：Context Studio が DBT のセマンティクスを Snowflake Cortex 可読フォーマットに変換。自信満々に答えるのではなく不確実性を表に出すエージェント。Atlan の用語集・メトリクス定義がエージェントのコンテキストを供給
- **In Progress**：DBT を超えて raw な Snowflake View や Sigma ダッシュボードへ拡張。Metadata Lakehouse のメタデータ深化、Context Studio 強化が MIDAS に与える影響の評価
- **Up Next**：設計要件ドキュメントを取り込んで用語集作成を自動化。**Atlan の MCP を定義リポジトリ兼コンテキストエンハンサーとして活用**。Context Studio をフルなセマンティックマスターに

ダッシュボードの数値とエージェントの出力が一致しない問題（DBT / Sigma / Atlan 間でのセマンティクスの整合）への言及もあり、ここは現場の生々しい課題だなと感じました。

---

## どこから始めるか：User-centric Design

最後に、実務に持ち帰れる「最初の一手」のまとめ。

![Where to Start? User-centric Design](/images/snowflake-summit-2026-atlan-workday-midas/IMG_0551.jpg)

1. **Pick one domain** — いま一番大事なメトリクスを 1 つ選ぶ。全部から始めない
2. **Define it once** — ロジック・例外・ビジネスコンテキストをちゃんと書き下し、エージェントが読める場所に置く。「それが仕事の全部」
3. **See how it's used** — 下流のプロンプト・分析・意思決定がどれだけ良くなるかを観察する。コンテキストの価値は**複利で効く**

> **That's the first move.**

「スタック全体を完璧にしてから始める」のではなく、**1 ドメイン・1 メトリクスで素早く立ち上げて使われ方を観察し、反復する**。よくいうsmall start, quick winですね。

---

## まとめ／所感

セッションを通して見えてきた論点を整理します。

- **コンテキスト = セマンティックレイヤー、ではない。** Knowledge / Expertise / Norms（Atlan）あるいは Technical / Structural / Tacit（Workday）と、複数レイヤーで捉えるべき
- **暗黙知をどう機械可読にするか**が最大の難所。SME に手作業でドキュメント化させるアプローチは何度も失敗してきた、という指摘は重い
- **コンテキストはコードのように扱う。** Build/Test/Review/Approve/Deploy/Learn のライフサイクル、バージョニング、ドリフト観測、リネージ
- **「知らないことを知っている」ことが信頼性の機能。** ハルシネーションを避け、データがなければ明示的にそう言う設計
- **MCP はコンテキストの供給／オーケストレーション層に自然に組み込まれつつある。** Atlan の Activation 経路にも、Workday の Orchestration 層にも MCP が登場していた

セマンティックレイヤーや MCP ゲートウェイを扱っている身としては、「コンテキスト層をプロダクトとして設計し、ガバナンス・オブザーバビリティまで含めて運用する」という発想は、今後のエージェント基盤の標準形になっていきそうだと感じました。

