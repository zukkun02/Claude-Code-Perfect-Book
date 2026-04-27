# RAG・Reflection・AI エージェントは何の学問か — マッピングと学習パス

> 課題：「こういう RAG とか Reflection とか、何かの学問になる？」への回答メモ。
> 結論：**ひとつの学問**ではなく、**いくつかの古い学問の交差点で生まれている新しい工学領域**。
> 関連：[Slack起点RAG実装メモ.md](./Slack起点RAG実装メモ.md) / [データ取得チャネル設計メモ.md](./データ取得チャネル設計メモ.md) / [0425レポート.md](./0425レポート.md)

---

## 結論を 1 行で

> **古典分野（情報検索・認知科学・システム理論）の上に、2020年以降の "AI Engineering" が乗ってきている。**

人間との比喩（脳・記憶・道具）でセミナーが進んでいたのは偶然じゃなくて、**もともと認知科学から発想された設計**だから違和感なく繋がる、ということ。

---

## 1. セミナーで出てきた概念の学問ルーツ

| セミナーの言葉 | 古典分野 | 現代の名前 | 重要論文 / 教科書 |
|---|---|---|---|
| **RAG** | **情報検索（Information Retrieval, IR）** + NLP | Retrieval-Augmented Generation | Lewis et al., 2020（Meta） |
| **エンベディング・セマンティック検索** | 分布意味論 + 計算言語学 | Vector Search / Distributed Representations | Mikolov, Word2Vec 2013 → BERT 2018 |
| **Reflection Loop** | サイバネティクス（Wiener, 1948）+ 強化学習 | Reflexion / Self-Improving Agents | Shinn et al., 2023 |
| **Agentic Loop / ReAct** | 多エージェントシステム（MAS, 1990s〜） | ReAct, AutoGPT, LangGraph | Yao et al., 2022（ReAct） |
| **エージェントオーケストレーション** | 分散 AI / BDI アーキテクチャ | Multi-Agent Orchestration | "An Introduction to MultiAgent Systems"（Wooldridge） |
| **CoT（思考の連鎖）** | 認知科学（メタ認知）+ NLP | Chain-of-Thought Prompting | Wei et al., 2022（Google） |
| **メモリ（短期 / 長期）** | 認知心理学（Atkinson-Shiffrin 多重貯蔵モデル, 1968） | Memory-Augmented LMs | Atkinson & Shiffrin, 1968 |
| **チャンク・構造化** | **図書館情報学（Library & Information Science）** | Document Chunking | Ranganathan のファセット分類（1933）まで遡れる |
| **インストラクション設計** | プロンプトエンジニアリング + ヒューマン-コンピュータ・インタラクション（HCI） | Prompt Engineering | "Prompt Report"（2024 メタ調査） |
| **PDCA on agents** | システム工学 / 品質管理 | LLMOps / AgentOps | "MLOps" 系の延長線 |
| **転移（経験から学ぶ）** | 教育心理学（転移学習） | Transfer Learning | Cognitive science 古典 |

→ **新しく見える概念ほど、実は 20〜80 年前の古典の焼き直し**。セミナーの「人間と一緒の構造」という比喩は、ここに根拠がある。

---

## 2. 統合する呼び名（学問の入口）

下に行くほど新しい / 実践寄り。

| 名前 | 性格 | 大学で学べる？ |
|---|---|---|
| **Information Retrieval（情報検索）** | 古典・成熟分野（50年） | ◎（CS の必修クラス） |
| **Natural Language Processing（NLP）** | 中堅、深層学習で再点火 | ◎（Stanford CS224N が世界標準） |
| **Cognitive Architectures**（SOAR, ACT-R 等） | 古典 AI、復権中 | ◯（認知科学・心理学） |
| **AI Agents / Multi-Agent Systems** | 1990s から続く | ◯（一部大学） |
| **AI Engineering**（新興） | 2023〜の実務分野 | △（まだコース化途中） |
| **LLMOps / AgentOps** | 運用工学 | ✕（OJT が主） |
| **Prompt Engineering** | 実践技法 | ✕（書籍・コース） |

→ アカデミックに 1 個選ぶなら **Information Retrieval + NLP**。
→ 実務に振るなら **AI Engineering**（書籍・コースが急速に整備中）。

---

## 3. "土台" として効く古典分野（隠れた本丸）

実は新しい論文より、**古典の方が本質を掴むのに早い**ことがある。

| 古典 | なぜ効くか |
|---|---|
| **サイバネティクス（Wiener, 1948）** | "フィードバックで自律する系" の祖。Reflection Loop の原典。 |
| **図書館情報学（Ranganathan, 1933）** | チャンク・タグ・ファセット分類。RAG の前身がここに全部揃ってる。 |
| **認知心理学（記憶モデル）** | 短期/長期記憶、転移、メタ認知 — エージェント設計の原型。 |
| **情報理論（Shannon, 1948）** | エントロピー、圧縮、信号と雑音 — チャンク戦略の根拠になる。 |
| **GTD / Zettelkasten / PKM** | "個人の RAG" の前史。Notion で考えDBを作るあなたの直系ルーツ。 |

---

## 4. みずきが学ぶならこの順（おすすめ学習パス）

### Lv.1：実践理解（1〜2 週間）
- **DeepLearning.AI** の無料コース（Coursera）
  - "LangChain for LLM Application Development"
  - "Building Systems with the ChatGPT API"
  - "Multi AI Agent Systems with crewAI"
- **書籍**：Chip Huyen *AI Engineering* (2024) ← 一番おすすめ。実務とアカデミックの中間。

### Lv.2：教養としての土台（1〜2 ヶ月）
- **Sönke Ahrens *How to Take Smart Notes***（邦訳『TAKE NOTES!』）
  - **Zettelkasten** ＝ 個人 RAG の前史。あなたの Notion 思考 DB と完全に同じ思想。
- **Marvin Minsky *Society of Mind***（『心の社会』）
  - "心は小さなエージェントの集まり" という発想の原典。エージェントオーケストレーションの祖。
- **Daniel Kahneman *Thinking, Fast and Slow***（『ファスト＆スロー』）
  - System 1 / System 2 の二重プロセス論 = LLM の高速応答 vs CoT 推論の比喩。

### Lv.3：論文を読む（半年〜）
- **Lewis et al. 2020**（RAG 原典）
- **Yao et al. 2022（ReAct）**
- **Shinn et al. 2023（Reflexion）**
- **Wei et al. 2022（CoT）**
- 全部 arXiv で無料。読み始めたら 1 つ 1〜2 時間。

### Lv.4：教科書（本気で土台作る）
- **Jurafsky & Martin *Speech and Language Processing***（無料 PDF）
  - NLP の世界標準教科書。第 14 章あたりに RAG が入ってる。
- **Russell & Norvig *Artificial Intelligence: A Modern Approach***
  - AI 全体の世界標準。エージェントの章は古典のまま今も通用する。

---

## 5. みずきへの実用アドバイス

> セミナーで「**実装が大事、Opus 一辺倒では精度が下がる**」と言われていた背景には、**学問的に "適材適所のモデル分業" が情報処理の基本** という事実がある。これは Distributed Computing の古い教えと同じ。

つまり：

- **学問を知っていると、流行に流されない**（Opus に全部やらせる ≠ 正解、と即判断できる）
- **比喩で繋がっている**ので、AI Engineering は他分野と接続しやすい（Notion 的整理術 ↔ Zettelkasten ↔ RAG）
- **コンサルティングで強くなる**：クライアントに「これは図書館情報学の応用です」と言えると、AI 嫌いの上の世代にも通じる

---

## ひとこと要約

> **新しい工学（AI Engineering）の皮を被った、古い学問の組み合わせ**。
> RAG = 情報検索 + 分布意味論。
> Reflection = サイバネティクス + 強化学習。
> エージェント = 認知科学 + 多エージェント系。
> "新しい概念" として身構えるより、**「これは図書館の話」「これはフィードバック制御の話」と古典に紐付けて読む** ほうが速く・深く理解できる。

> **claudeからの補足**
> - 学位が出るかというと、**"AI Engineering" の単独学位はまだ少ない**（2024〜25 で MIT・Stanford 等が修士コースを新設し始めた段階）。情報科学修士の中の選択トラック扱いが多い。
> - 一方で **Coursera / DeepLearning.AI / Andrew Ng の系列**は、実務家が修士相当の知識を体系化するのに最も効率が良い。
> - **趣味と仕事を兼ねるなら**、「Zettelkasten + AI Engineering」 の組み合わせがみずきの方向性に最もフィットする（個人ナレッジ × 機械的検索 × エージェント運用）。
> - 古典に弱気にならなくて OK。**Wiener 1948 と Ranganathan 1933 は、PDF や邦訳が無料で読める**水準で公開されている。

---

## 参考：本メモが扱う「新旧マッピング」の全体図

```
[古典]                          [現代]                          [現場]
─────────                       ─────────                       ─────────
情報理論 (Shannon 1948)    ─→   NLP / 分布意味論          ─→    Embedding API
図書館情報学 (Ranganathan)  ─→   Document Chunking         ─→    Notion 構造化
情報検索 (1960s〜)          ─→   Vector Search             ─→    Supabase pgvector
サイバネティクス (Wiener)    ─→   Reflexion / Self-Improve  ─→    エージェント運用
認知心理学 (Atkinson 1968)  ─→   Memory-Augmented LMs       ─→    Memory.md / RAG
多エージェント系 (1990s)    ─→   Agent Orchestration       ─→    Claude Code Subagents
GTD / Zettelkasten (1980s) ─→   Personal RAG              ─→    考えDB × Notion AI
教育心理学（転移）          ─→   Transfer Learning         ─→    人間の学び方の比喩
```

---

*Generated: 2026-04-26 / 関連メモ: Slack起点RAG実装メモ.md, データ取得チャネル設計メモ.md, エンべディングログ.md, 0425レポート.md*
