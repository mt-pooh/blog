---
title: "Code with Claude 2026 参加レポート：長時間エージェント時代に「本番で信頼できるAI」をどう作るか"
emoji: "🦾"
type: "tech"
topics: ["claude", "claudecode", "ai", "llm", "anthropic"]
publication_name: "finatext"
published: false
---

## はじめに

Anthropic の開発者カンファレンス **Code with Claude 2026** に参加してきました。Claude を作っているチーム自身から、Claude Code やエージェント開発の最前線を聞ける2日間で、キーノート・ワークショップ・顧客／スタートアップ事例と、持ち帰るものが非常に多いイベントでした。

本記事では、特に学びの大きかった4つのセッションを、**1本のレポート**としてまとめます。

1. **キーノート** ― Claude Code のこれまでと、新モデル Claude Fable 5 / Mythos 5
2. **ワークショップ** ― エージェント駆動の検証（最初から検証を組み込む）
3. **Myrealtrip × AICX** ― プロトタイプを本番へ（責務分離）
4. **Federation** ― 2人チームが世界1位を取った AI ワークフォース

## 通底するテーマ：能力の天井が上がった先で、どう信頼するか

4つのセッションは、見事に一本の線でつながっていました。

キーノートで示されたのは、**エージェントが「数時間〜数日」走り続けられる時代に入った**という能力の天井の話です。そして残りの3セッションは、その裏返し――**「長く走れるようになったエージェントを、どうやって本番で信頼するか」**への、それぞれ異なる角度からの答えでした。検証（②）、決定論的な責務分離（③）、オーケストレーションと監査証跡（④）。賢さの追求から、**信頼できる形で出荷する**ための設計へ、関心が完全に移っているのを感じました。

以下、セッションごとに見ていきます。

---

## ① キーノート ― Claude Code のこれまでと Claude Fable 5 / Mythos 5

### Claude Code の歩み

キーノートは、Claude Code の生みの親 **Boris Cherny** 氏の原体験から始まりました。学生時代に TI-83（関数電卓）で TI-Basic を書き、ポケモンカードを売るために HTML を独学した――そうした「手を動かして遊ぶ」体験が原点だ、という導入です。

この感覚はプロダクト史にもそのまま重なります。Claude Code は **2025年2月にリサーチプレビュー**として小さなチームから世に出ましたが、立ち上がりは遅く、**中止寸前**まで行きました。スライドには、社内 Slack での Boris 氏の最初の投稿が映し出されました。

![Boris Cherny 氏が2024年9月に社内 Slack で Claude CLI を紹介した投稿](/images/code-with-claude-2026-report/IMG_0720.JPG)

そこから諦めずに改善を続け、モデル側の進化（Opus 4 / Sonnet 4 世代）と噛み合ったことで、ようやく PMF に到達した――派手なローンチではなく地道なイテレーションの先にフィットがあった、という語り口が印象的でした。

そして中盤で繰り返されたのが、**エージェントは「アイデアがある」状態と「それが動く」状態の距離を一気に縮める**というテーマ。仕事の重心は「コードを書く」から「エージェント・ループ・ルーティンを管理する」へ移る、という主張です。

### Claude Fable 5 / Mythos 5 の発表

ハイライトは、開催直前（2026年6月9日）に発表された新モデルでした。

- **Mythos クラス**は、これまでの Opus の上に位置づけられる新ティア
- **Fable 5** は、その Mythos クラスを**一般公開した初のモデル**。現時点で公開中では最も高性能
- Fable 5 と Mythos 5 は**同じベースモデル**を共有し、Fable 5 には安全のためのガードレールが付く
  - サイバーセキュリティ・生物・化学・蒸留などの高リスク領域では応答が制限され、**Claude Opus 4.8 にフォールバック**する
- **Mythos 5** はその制約を一部解除したモデルで、Project Glasswing のパートナー等に限定提供

強調されていた Fable 5 の強みは、まさに長時間・自律実行に効くものでした。長時間ランニングのゴールを信頼性高く走り切る、数百万トークン規模のメモリ／長コンテキスト、**並列サブエージェント**のディスパッチと進捗管理、ファーストショットの正答率向上、コードの読解・トリアージ（バグ発見、障害トリアージ、リポジトリのフォレンジック）。公式発表では、**Claude Code のようなエージェントハーネス上で「数日単位」で動作**し、計画・進捗確認・自己修正を繰り返せるとされています。

ベンチマークでも、長く難しいタスクほど差が開く傾向が示されています（一例）。

| ベンチマーク | Claude Fable 5 | Claude Opus 4.8 |
| --- | --- | --- |
| SWE-Bench Pro | 80.3% | 69.2% |
| FrontierCode（Cognition） | 29.3% | 13.4% |

価格は **入力 $10 / 出力 $50（100万トークンあたり）** で Opus 4.8 の概ね2倍。Claude API や AWS（Amazon Bedrock / Claude Platform on AWS）から利用可能で、Pro / Max / Team / Enterprise では **6月22日まで**利用でき、**6月23日以降**は容量が確保されるまで利用クレジットが必要になる可能性がある、という案内でした。

---

## ② ワークショップ ― 最初から「検証」を組み込む

「数日走るエージェント」が現実になると、裏返しの課題が立ち上がります。**長く走った末に成果物が壊れていれば、やり直しは高くつく**。実行時間（time horizon）が10〜15時間以上に伸びるほど、**失敗のコスト**が跳ね上がる――このワークショップは、その対策としての「計画と検証」の話でした。ワークショップを貫く3つの柱がこのスライドに集約されていました。

![Three tools for working with long-running agents：Removing ambiguity / Understanding & planning / Integrated verification](/images/code-with-claude-2026-report/IMG_0721.JPG)

### 曖昧さを先に潰す

長時間走らせる前に、まず曖昧さを潰します。「割り勘アプリを作って」のような曖昧な指示はNG。良いパターンは、**分からないことを認め、エージェントにユーザーをインタビューさせる**こと（ask-user 質問やMCQ、ラウンド数は3〜4問に制限）。

![Bad prompt vs Good prompt — 曖昧な指示は最初の1時間でエージェントが勝手に方向を決めてしまう](/images/code-with-claude-2026-report/IMG_0722.JPG)

### HTML を「対話の媒体」として使う

技術的に面白かったのが、**HTML をモデル出力を読む／触るための新しい言語**と位置づける発想です。軽量・安価・使い捨てが効くので、サッと作ってローカルで反復・クリックして確かめられる。生成した HTML を保存することで**ユーザーの美的な好み（配色・フォント・タイポグラフィ）を学習**でき、Markdown の塊を読ませる代わりに**実物に対して反応**してもらえます。

![HTML plans are easier to read — Markdown の箇条書きより HTML の方がモデルの思考をより多く引き出せる](/images/code-with-claude-2026-report/IMG_0723.JPG)

### DOM contract と決定論的な検証

核は **DOM contract**――React の状態（state/memory）を **DOM に「刻印（stamp）」**し、エージェントがアプリ状態を**機械可読**に読めるようにする考え方です。そのうえで4つの成果物を定義します。

- **Schema**：状態・フィクスチャの機械可読な「形」
- **Fixtures**：レンダリングする想定状態（アクティブ、空テキスト、長文 など）
- **Invariants**：常に真であるべきルール（例：DOM のクラスが React の state と一致）
- **Probes**：要素や状態を実際に操作・検査する自動チェック

ポイントは、これらを**決定論的（deterministic）に回す**こと。ランダムなブラウザ操作の「運任せ」を避け、毎回同じ probe を回して差分を見ます。

![Three principles：Build for it from the start / Modularize by verifiability / Verify across the stack](/images/code-with-claude-2026-report/IMG_0724.JPG)

検証の実行モードは3つ。**CI 駆動**（PR上で自動実行）、**ブラウザ / Playwright**（視覚的に操作・記録できるが遅く・ランダムで・トークンコストが高い）、**ローカルの人間可読**（HTML 可視化＋短い録画/GIF で素早くレビュー）。軽量なチェックで固めてから、必要に応じて高コストな Playwright を判断して使う、というコストバランスが繰り返し強調されました。

リポジトリには `/verify` ダッシュボードを含む検証ハーネスがあり、コンポーネント状態を DOM に刻印し、fixture 駆動の probe を回し、計画と現実の**差分（delta）を可視化**、PR 用に**クリップ/GIF を録画**できます。デモでは、**React の state は変わっているのに取り消し線が視覚的に表示されない**という、単体テストでは見落としがちなバグを DOM contract のミスマッチとして検出・修正していました。

![Anatomy of the system：verify/specs → registry.ts → runner.ts → Dashboard / UnitPage / window.__verify / matrix.test.ts](/images/code-with-claude-2026-report/IMG_0725.JPG)

加えて、**builder モデル（書く）と detector モデル（読んで検証するだけ）の2モデルを敵対的に反復させる**GANライクなパターンも紹介され、Claude Code の `/goal` で再現できるとのことでした。

---

## ③ Myrealtrip × AICX ― 責務分離でプロトタイプを本番へ

Myrealtrip（マイリアルトリップ）と AICX の Kim Nami 氏による顧客セッション。題材は**航空券のキャンセル／変更手数料の計算**で、「デモは動くのに本番で詰まる」壁を、具体的なアーキテクチャとして語ってくれました。

![Production is where things break：Prototype（Claude → impressive answer）vs Production（Claude + code + data）](/images/code-with-claude-2026-report/IMG_0727.JPG)

![Where we work on this：Myrealtrip（韓国 No.1 旅行プラットフォーム）→ AICX（100% 子会社、AI ワークフロー担当）](/images/code-with-claude-2026-report/IMG_0728.JPG)

### 「Calculate everything」の失敗

![Airline refund policies are messy：自然言語・航空会社ごとの例外・頻繁な変更・人間でも難しい](/images/code-with-claude-2026-report/IMG_0729.JPG)

航空券の手数料は、出発までの日数でティアが変わる複雑なルールです（3日以内／4〜14日／15〜60日／出発後、さらに変更とキャンセルで別物）。たとえばリクエスト 7/28・出発 8/13 なら**出発の16日前 → 15〜60日バンド → 90,000ウォン**、そして `872,000 − 90,000` のような算術は**決定論的でなければならない**。

初期設計は、**LLM にほぼ全部渡して「全部計算して（Calculate everything）」と頼む**もの。デモでは自然に動きましたが、本番では破綻しました。

![Asking the model to calculate money：arithmetic + RAG + NLG を1回の Claude 呼び出しに詰め込んだ失敗設計](/images/code-with-claude-2026-report/IMG_0730.JPG)

**レイテンシ約8秒**（コードなら約1ms）、**実行のたびに金額がブレる非決定性**（1万〜10万ウォン規模の差＝金銭リスク）、そして自然言語推論への**自動アサーションの書きにくさ**。

![What broke in production：8s vs 1ms / ₩90k vs ₩100k / assert == ?](/images/code-with-claude-2026-report/IMG_0731.JPG)

### 解決は「責務分離」

![From 'calculate it' to 'don't calculate it'：Claude はポリシーを抽出するだけ、Python が計算する](/images/code-with-claude-2026-report/IMG_0732.JPG)

再設計の肝は、**モデルの責務を絞る**こと。

- **モデルによるパース**：LLM はポリシー文と予約を読み、**構造化データ（JSON的なティア情報）を抽出**
- **コードによる計算**：Python が決定論的な日付・金額の算術と区間マージを行う

![PR #23: tools/ → use_cases/ ：before/after のコードで責務分離を実装](/images/code-with-claude-2026-report/IMG_0733.JPG)

![Current production: One Claude call, rest is Python](/images/code-with-claude-2026-report/IMG_0734.JPG)

![Parse with Claude, calculate with Python：Claude は JSON のみ・日付計算なし、Python はすべて決定論的・ユニットテスト可能](/images/code-with-claude-2026-report/IMG_0735.JPG)

これで、速いタスクはコードが担当して**ほぼ即時**になり、計算ロジックは Python のアサーションで**テスト可能**になり、パースと計算が**疎結合**になってモデル変更でコアが壊れなくなりました。**「自然言語の解釈はモデル、決定論的な計算はコード」**というきれいな線引きです。

そして後半は組織の話へ。**AIでコーディングの実現可能性が上がるほど、ボトルネックは"コードを書くこと"から"調整とオーナーシップ"へ移る**。

![Claude Code shifted the bottleneck：Before は「コードが書けるか」待ち、After は「誰が決めるか・コンテキストをどこに置くか」待ち](/images/code-with-claude-2026-report/IMG_0736.JPG)

小さな実行ユニットと、エンドツーエンドで責任を持つ単一オーナーに寄せて、高コストなクロスチーム調整を減らす。

![Same principle, two layers：Systems（モデルがパース、コードが計算）と Teams（オーナーが決定、AI が実行）](/images/code-with-claude-2026-report/IMG_0737.JPG)

![AI Native operating model：Smaller teams move faster with AI — Customer-driven R&D / White-glove partnership / Trusted peer network](/images/code-with-claude-2026-report/IMG_0738.JPG)

**「AIはシステムアーキテクチャとチームアーキテクチャの両方を変える」**という一言がセッションの核でした。

![Takeaways：AI changes architecture / Don't give the model everything / Don't give endless handoffs / Reduce expensive reasoning / Reduce expensive coordination](/images/code-with-claude-2026-report/IMG_0739.JPG)

---

## ④ Federation ― 2人チームが世界1位を取った AI ワークフォース

最後は Federation の創業者 **Gahee Seo** 氏によるセッション「The 1% problem」。**わずか2人のチームが、Alibaba のグローバル HS コード分類ベンチマークで世界1位**を取った話です。

![The 1% problem: How domain expertise + Claude let a 2-person team hit #1 on a global classification benchmark — Gahee Seo, CEO, Federation](/images/code-with-claude-2026-report/IMG_0740.JPG)

創業者の家業が突然の規制変更で制裁金を課された原体験から、「専門チームを持てない中小企業を底上げする」という課題意識で始まっていました。

掴みは3つの数字。**1桁の HS コード誤分類が2024年に生んだ $365M**、専用アーキテクチャ無しのベンチで**29%**、人手なら1ディールに**1週間**。HS コードは1桁の誤りが貨物の留置・制裁金・破談を招き、しかも正解は最終製品名ではなく**素材構成比・加工方法・用途**に依存します。

![1% accuracy, millions in fines, that's the territory：$365M / 29% / 1 week](/images/code-with-claude-2026-report/IMG_0741.JPG)

### 分類器ではなく「専門家のワークフロー」を再現する

最大のポイントは、精度の高い分類器を訓練するのではなく、**専門家の意思決定ワークフロー（プロセス＋根拠）を再現する**こと。専門家は22,000のコードを暗記せず、**5段階の階層プロセス**で絞り込み、**WCO（世界税関機構）の解説書・GRI（解釈通則）1〜6・類似ケース**を参照します。だからシステムも、**HSコード＋適用GRI通則＋参照解説＋類似ケース引用**を返す。狙いは、**当局に説明可能で監査に使える**出力にすることです。

![We didn't build a better classifier. We replicated how the expert decides. — 分類器は精度の天井に縛られるが、専門家ワークフローの再現は監査証跡で競争できる](/images/code-with-claude-2026-report/IMG_0742.JPG)

内部では**11個の専門ツール**が並列・直列に動き、全体は**7つのフレームワーク（4システム＋3横断レイヤー）**で構成。

![Four systems, eleven tools, seven frameworks：HS code classification / Regulatory analysis / Supply chain risk / Trade assistant](/images/code-with-claude-2026-report/IMG_0743.JPG)

モデル選定も**生のベンチマークでなく、タスクに必要な能力プロファイル（長コンテキスト・引用規律）で選ぶ**という原則で、HS 分類には200Kコンテキストが効きました。規制分析では**4管轄にまたがる100ページ超の規制文書（CBAM：炭素国境調整メカニズムとみられる）を並列パース**し、サプライチェーンは**約80ステップ**（供給元の集中度＝Herfindahl 指数とみられる、世界銀行のガバナンス指標、5要素スコアリング、代替調達判断）を連鎖。トレードアシスタントは **think-act-observe-decide のループ**で動き、**不確実なときは止まって人間ゲートに渡す**設計でした。

![Each system needed a different capability：HS code / Regulatory / Supply chain / Trade assistant それぞれが異なる Claude の能力を必要とする](/images/code-with-claude-2026-report/IMG_0744.JPG)

### 3つの失敗モードと対処

1. **レイテンシの壁**：単一エージェントで6分以上 → **段階分解＋並列化**で同じモデルのまま約30秒へ

![Single agent worked. 6+ minutes per answer. No customer waits that long. — 段階分解＋並列化で 30 秒へ](/images/code-with-claude-2026-report/IMG_0745.JPG)

2. **入力品質の低さ**：誤ったインボイス記述で誤分類（「machinery part」だが正解は医療用冷蔵）→ **入力品質ゲートのエージェント**で検証・明確化・エスカレーション

![Our system never failed, customer-written descriptions did — モデルでなく入力が原因。Self-aware system > confident system](/images/code-with-claude-2026-report/IMG_0746.JPG)

3. **オーバーエンジニアリング**：全ルールをプロンプトに詰めて遅く脆く → **検索ベースのKB＋引用をクロスチェックする検証エージェント**へ

### AI ワークフォースという結論

最終的に、**各貿易ルールがそれぞれ独立したエージェントになり、中央の「ブレイン」エージェントがオーケストレーションする**という「AIワークフォース」に到達。承認ゲート、出荷トラッキング、リード発見・ディール起案エージェント（例："Maggie"）、そして**完全な監査証跡**を備えます。出力が当局に対して説明可能なレベルに達していたため、**韓国の税関・規制当局が実際に使い、新人教育にも利用した**とのことでした。

![Five lessons that we learned：Decompose by stage / Match model to profile / Don't trust input / Outside knowledge / Teach the workflow](/images/code-with-claude-2026-report/IMG_0747.JPG)

5つの教訓も明快でした。**段階で分解する／ベンチマークでなくドメイン能力でモデルを選ぶ／入力を額面通り信じない／知識はプロンプトの外に置き、監査証跡こそをプロダクトとする／答えだけでなく専門家のワークフローを再現する**。

---

## 4セッションを貫く学び

通して振り返ると、いくつかの原則が繰り返し現れていました。

- **能力の天井は上がった**（Fable 5：数日走る・並列サブエージェント）。だからこそ問いは「賢さ」から「**本番でどう信頼するか**」へ移る
- **決定論的なロジックは LLM の外（コード）に出す**。金額・日付・権限判定のように**ブレてはいけないもの**はコードで固め、モデルには自然言語の解釈という、モデルにしかできない仕事を任せる（③④共通）
- **検証・監査証跡こそがプロダクト**。状態を機械可読にして決定論的に検証し（②）、引用と推論トレースを残す（④）
- **不確実なときに止まって人間に渡せること**（human gate）が、規制・金銭領域で信頼されるための核（③④）
- **ボトルネックは「コードを書くこと」から「何をリリースするか決めること」へ**。AIで実行速度が上がるほど、意思決定とオーナーシップの設計が勝負どころになる（①③）

フロントエンドの検証も、航空券の手数料計算も、貿易コンプライアンスも、根っこは同じでした。**状態を機械可読にし、決定論的に検証し、不確実なら止まり、根拠（監査証跡）を残す**。これは、エージェント基盤やゲートウェイを設計するうえで――つまり「何が起きたかを観測・検証し、権限とガバナンスを効かせる」基盤づくりにおいて――そのまま参照点になる原則だと感じました。「ドメインの専門性＋良いアーキテクチャ」が、基盤モデル単体では越えられない堀になる。Code with Claude 2026 を貫いていたのは、この一点だったように思います。

---

:::message
本記事はカンファレンスの書き起こしメモと公式発表をもとに再構成したものです。数値・固有名詞・提供条件は発表時点のものであり、最新情報は各社・各当局の公式情報をご確認ください。
:::
