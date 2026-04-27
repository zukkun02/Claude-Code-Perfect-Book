# Slack 起点で「考えDB → ポスト案」を生成する RAG 実装メモ

> 課題：「Slack からポスト作成案を生成したい」。エンベディング＋セマンティック検索を、ローカル実験から本番運用に格上げするための設計メモ。
> 前提：Notion DB「DB_録音文字起こし」を真実のソースとして、自分の考えに沿ったポスト案を返す。
> 関連：[エンべディングログ.md](./エンべディングログ.md) / [ポスト案.md](./ポスト案.md) / [0425レポート.md](./0425レポート.md)

---

## 1. ゴール

Slack で `/post-draft Notion AI` のようにコマンドを叩くと、

1. クエリをベクトル化
2. 自分の考えDB の中から意味的に近いチャンクを取り出す
3. それを context に LLM がポスト案を生成
4. Slack に投稿として返す

…が **数秒以内** に完了する状態を作る。

---

## 2. 全体像（4 コンポーネント）

```
①Slack（トリガー）
   │ /post-draft Notion AI
   ▼
②バックエンド（HTTPS endpoint）
   │ クエリを Gemini Embedding でベクトル化
   │ pgvector に近傍検索（Top-K）
   ▼
③ベクトル DB（Supabase pgvector など）
   ▲
   │ 別ジョブで定期同期
④Notion DB（自分の考え／録音文字起こし）
```

ローカル実験で持っていた `embeddings_full.json` を **③に置き換える**、`embed.py` を **②でホストする**、それだけ。
**やることの本質は変わらない。配置先と起動方法が変わるだけ**。

---

## 3. 3 段階の実装パス

| Lv | 構成 | 向いているケース | 工数 |
|---|---|---|---|
| **Lv.1** | Slack Workflow + Make / n8n（ノーコード配線） | まず動かしたい・PoC | 30 分 |
| **Lv.2** | Slack Slash Command + Vercel + Supabase pgvector | **本命運用**（推奨） | 1〜2 日 |
| **Lv.3** | Claude Code 自身に Slack Web API を叩かせる | 個人用・最速で試す | 1 時間 |

---

## 4. Lv.2 本命構成（推奨）

セミナーで推奨されていた「**セマンティック検索 × クエリ**（ハイブリッド検索）」を最も素直に実現できる構成。

### 4-1. Supabase 側の準備

```sql
-- 1) pgvector 拡張
create extension if not exists vector;

-- 2) チャンクテーブル
create table thought_chunks (
  id           bigserial primary key,
  doc_id       text not null,                       -- Notion ページID
  title        text,
  text         text not null,
  category     text,                                -- フィルタ用メタデータ
  tags         text[],
  created_at   timestamptz,
  embedding    vector(1536)                         -- ※下の補足参照
);

-- 3) 近傍検索用インデックス
create index on thought_chunks
  using hnsw (embedding vector_cosine_ops);
```

> **次元の注意**：`gemini-embedding-001` の既定次元は **3072** だが、HNSW インデックスの上限は 2000。
> → API リクエスト側で `output_dimensionality=1536` を指定して落とす（Gemini Embedding は matryoshka 学習なのでこれが正規の運用）。

### 4-2. ハイブリッド検索 RPC

```sql
create or replace function match_chunks(
  query_embedding vector(1536),
  match_count int,
  filter_category text default null
)
returns table (id bigint, title text, text text, score float)
language sql stable as $$
  select id, title, text,
         1 - (embedding <=> query_embedding) as score
  from thought_chunks
  where filter_category is null
     or category = filter_category        -- ★クエリ側（メタデータフィルタ）
  order by embedding <=> query_embedding  -- ★セマンティック側
  limit match_count;
$$;
```

→ この関数 1 個で **「フィルタで絞る → 残りでセマンティック」** がワンショットで終わる。

### 4-3. Slack エンドポイント（Vercel Serverless 例）

```ts
// /api/slack/post-draft.ts
export default async function handler(req, res) {
  // 0) Slack 署名検証（必須）
  if (!verifySlackSignature(req)) return res.status(401).end();

  const query        = req.body.text;          // "Notion AI"
  const responseUrl  = req.body.response_url;
  res.status(200).end();                        // ★3秒以内にACK必須

  // 1) クエリをベクトル化（taskType=RETRIEVAL_QUERY）
  const qvec = await embedQuery(query);

  // 2) pgvector でハイブリッド検索
  const { data: chunks } = await supabase.rpc("match_chunks", {
    query_embedding: qvec,
    match_count: 15,
    filter_category: null   // 必要なら "AI Integration" 等
  });

  // 3) Claude / Gemini でポスト案生成
  const drafts = await generatePosts(query, chunks);

  // 4) Slack へ非同期返信
  await fetch(responseUrl, {
    method: "POST",
    body: JSON.stringify({ text: formatDrafts(drafts) })
  });
}
```

### 4-4. Notion → Supabase 同期ジョブ

```ts
// cron: 0 * * * *  （or Notion webhook）
const pages = await notion.databases.query({ database_id });
for (const page of pages.results) {
  if (already_indexed_and_unchanged(page)) continue;        // ★差分同期
  const chunks = chunkBySentence(page.properties["テキスト"]);
  for (const c of chunks) {
    const v = await embedDocument(c);                       // taskType=RETRIEVAL_DOCUMENT
    await supabase.from("thought_chunks").upsert({
      doc_id: page.id, title, text: c, embedding: v, ...
    });
  }
}
```

→ `last_edited_time` を見て差分同期するだけで、**API コストは桁で落ちる**。

---

## 5. 今回のローカル実装からの移行マッピング

| 今回（ローカル） | Slack 配信版 |
|---|---|
| `records.json`（手動投入） | **Notion → Supabase 同期ジョブ** |
| `embed.py` のチャンク + 埋め込み | 同期ジョブに移植（`RETRIEVAL_DOCUMENT`） |
| `embeddings_full.json` | **Supabase `thought_chunks` テーブル** |
| Python の `cosine()` 関数 | **pgvector の `<=>` 演算子**（HNSW で高速化） |
| Top-15 を print | **RPC `match_chunks()`** でフィルタ込み取得 |
| ポスト案を `.md` に書く | **Slack `response_url`** に POST |

---

## 6. 運用上の注意（落とし穴）

- **Slack は 3 秒で ACK** しないと "operation_timeout"。重い処理は ACK の後に `response_url` で返す。
- **Slack 署名検証**（`X-Slack-Signature`）は省略しない。public endpoint を晒すなら必須。
- **次元 3072 → 1536** に落とす（HNSW 上限・速度・容量すべて改善）。
- **同期は差分**（`last_edited_time`）で。全件再エンベディングはコストの無駄。
- **思考データは public に置かない**。Supabase の RLS で行レベル権限。
- **コスト感**：Gemini Embedding は約 $0.15 / 1M トークン。月 1,000 ページ更新でも数百円台で収まる。
- **メタデータ命**。`カテゴリー / タグ / 作成日時` をフィルタに使うと、データが増えても精度が落ちにくい。

---

## 7. 推奨スタート順

1. **Lv.3（Claude Code + Slack Web API）** で 1 日で疎通させる。
   → Slack Bot を立てる前に「考えDB → 投稿案」のロジックが本当に欲しい品質か見極める。
2. 良ければ **Lv.2（Supabase + Vercel + Slash Command）** に移植。
   → ハイブリッド検索＋差分同期で運用に乗せる。
3. 仕上げに **Reflection Loop**（実行ログを Notion / Supabase に書き戻し → 検索品質を自己改善）を載せる。

---

# 最終まとめ（30 秒で読み直す用）

- やってることは **「クエリをベクトル化して、ベクトル DB を近傍検索して、引いた文を LLM に食わせる」** だけ。これは**今回ローカルで動いた `embed.py` と完全に同じ流れ**。
- Slack 化で増える要素は **3 つだけ**：
  1. **トリガー**：Slack Slash Command（or Workflow / メンション）
  2. **置き場所**：JSON ファイル → **Supabase pgvector**
  3. **同期**：Notion 更新 → 差分エンベディング（cron / webhook）
- セミナーで強調された **「セマンティック × クエリ」のハイブリッド** は、Supabase の RPC 1 個（`where category=... order by embedding <=> ...`）で実装される。
- 始めるなら **Lv.3 → Lv.2** の順がコスパ最良。**Slack Bot から作り始めると確実に詰む**ので、まずパイプラインの品質を確認してから配信側を整える。
- 拡張の本丸は **Reflection Loop**：返したポスト案の良し悪しをログに残し、検索クエリの書き換え／チャンク戦略／プロンプトを自己更新していく。これが回り始めた瞬間、人間の手が外せる。

> ひとことで言うと：**「ローカルで動いたあの Python を、配置先（DB）と起動口（Slack）だけ差し替えれば本番化できる」**。新しい技術は要らない。

---

# 補講：Supabase は何者か（裏方の役割整理）

「Supabase（pgvector）って結局なに？ Notion API がセマンティック検索できないから経由する場所？」という疑問への答え。

## 結論を 3 行で

- **Notion 純正の API は文字列検索しかできない**（"AI" を含むページを引く、まで）。
- **意味で引きたい（"Notion AI への考え方"）なら、ベクトルを別の場所に置いて検索するしかない**。
- その置き場所が **Supabase（pgvector）**。完全に**裏方の装置**で、人間は直接触らない。

## 比喩：図書館で考える

```
[Notion DB]  ＝  本棚（本そのものが置いてある）
                 → タイトル・著者で探すのは得意
                 → "孤独について書いてある本" は探せない（人手で全部読むしかない）

[Supabase]   ＝  司書が裏で作る「意味インデックス・カード」
                 → 各ページの"意味"を数字（ベクトル）に変換して保管
                 → "孤独" っぽい意味の本を一瞬で並び替えられる
```

- 本（オリジナル）はずっと **Notion** にある。書き換えるのも Notion。
- Supabase はそれを**機械が検索できる形に焼き直したコピー**を持っているだけ。
- だから Notion を更新したら、Supabase 側のインデックスも作り直す必要がある（＝同期ジョブ）。

## 「なんで Supabase が必要？」を 4 ステップで

| 質問 | 答え |
|---|---|
| ① Notion API で文字列検索はできる？ | できる（"AI" を含むページ引く、はOK） |
| ② Notion API で**意味検索**はできる？ | **できない**。これがボトルネック。 |
| ③ Notion AI の中なら意味検索できるよね？ | できるが、**Notion の UI / Q&A から使う前提**。Slack Bot から呼べる API は公開されていない。 |
| ④ じゃあどこで意味検索するの？ | 自分でベクトル化して、自分の手元の DB（＝Supabase）で検索するしかない。 |

→ Supabase は **「Notion の代わり」ではなく「Notion が出来ない仕事を裏で肩代わりする黒子」**。

## データの流れで言うと

```
[書く人]         [真実のソース]        [機械用のコピー]      [検索する人]
   |                  |                      |                 |
あなたが書く  →   Notion DB     →    Supabase pgvector   ←  Slack Bot が引く
                   (人間が読む)        (機械だけが読む)
                       │
                       └─── 編集したら ───→ 同期ジョブで Supabase に再投入
```

- **Notion を見て書く**のはあなた（人間）
- **Supabase を見て検索する**のは Slack Bot（機械）
- **同じ内容を持つけど、用途が違うので分けてある**

## 「全部 Notion で済ませたい」場合の選択肢

| 方法 | できること | 限界 |
|---|---|---|
| **Notion AI の Q&A** | Notion 内で意味検索 → 回答 | Slack に流せない／API 未公開／プロンプト制御弱い |
| **Notion API + 全文 grep** | 単語マッチで近そうな所を引く | 意味検索ではない（"孤独" で "ぼっち" は引けない） |
| **Supabase pgvector**（今回の案） | Slack/任意のアプリから意味検索可能 | 同期ジョブを別で回す手間 |

→ **Slack に流したい時点で、Notion AI 単体では届かないので Supabase のような外部インデックスが要る**。

## ひとこと要約

> **Notion ＝ 表の世界（人間の書き場）**
> **Supabase ＝ 裏の世界（機械の検索場）**
> **2 つを定期的に同期させる小さな配達員（同期ジョブ）が要る**

Slack Bot は Notion を直接見るのではなく、Supabase に「意味で似てるやつ持ってきて」と頼み、その結果を context にしてポスト案を作って返す。

> **claudeからの補足**
> Supabase 以外でも役割は同じ（Pinecone / Qdrant / Weaviate / ローカル JSON など）。**「ベクトルを置いて、似てる順に取り出せる場所」**ならどれでもよい。Supabase を推したのは PostgreSQL ベースで SQL 併用でき、無料枠が広く、セミナーで言及されていたから。

---

# 補講2：データを「埋め込む側」と「メタデータ側」に分ける

「ラベルやカテゴリは確定値だからクエリで絞る、意味で引きたいテキストだけ embed する」という設計原則。実用に落とすとどうなるかの整理。

## 原則：データを 2 種類に分ける

```
[埋め込むもの]               [埋め込まないもの = メタデータ]
─────────────────         ──────────────────────────
意味で引きたい本文           確定値で絞りたい属性
・テキスト本文                ・カテゴリ（"AI Integration"）
・要約                        ・タグ（["副業", "Monetizing Skills"]）
                              ・作成日時（2026-03-21）
                              ・著者・status・URL・ID
                              ・数値（人気度スコア等）

意味のグラデーション          完全一致 or 範囲指定
"孤独"≒"ぼっち"≒"独り"        "AI Integration" = "AI Integration"
```

### なぜ混ぜちゃダメか

- カテゴリ "AI Integration" を embedding に混ぜると、"AIに統合する" "AIに溶け込む" 等のノイズと**意味で似てしまう**。確定値は確定値で扱うのが速くて正確。
- 日時は数学（範囲・比較）で扱うべきもの。意味ベクトルにすると順序が壊れる。
- メタデータは**桁で軽い**。最初に絞ればセマンティック検索の対象が 1/100 になり、速度も精度も上がる。

## スキーマ：列で分ける

```sql
create table thought_chunks (
  -- ★メタデータ（クエリ・フィルタ用）
  id           bigserial primary key,
  doc_id       text not null,
  title        text,
  category     text,                    -- "AI Integration" 等
  tags         text[],                  -- ["Monetizing Skills", ...]
  created_at   timestamptz,
  status       text,

  -- ★埋め込み対象（セマンティック検索用）
  text         text not null,           -- 実際の発言・思考
  embedding    vector(1536)             -- text を Gemini で焼いたベクトル
);

-- メタデータ側のインデックス（B-tree / GIN）
create index on thought_chunks (category);
create index on thought_chunks (created_at);
create index on thought_chunks using gin (tags);

-- 埋め込み側のインデックス（HNSW）
create index on thought_chunks using hnsw (embedding vector_cosine_ops);
```

## 検索クエリ：絞ってから意味で並べる

```sql
-- 「AI Integration カテゴリの、2026-03 以降の考え」から
-- "Notion AI への私見" に意味的に近いトップ 10
SELECT id, title, text,
       1 - (embedding <=> $qvec) as score
FROM thought_chunks
WHERE category = 'AI Integration'             -- ★①絞る
  AND created_at >= '2026-03-01'              -- ★②絞る
  AND status = '完了'                          -- ★③絞る
ORDER BY embedding <=> $qvec                  -- ★④意味で並べる
LIMIT 10;
```

PostgreSQL は **WHERE で絞ってから ORDER BY** を実行。ベクトル比較の対象が一気に減って、速度・精度・コストすべて改善。

## Slack コマンドへのマッピング

```
/post-draft <意味で探したい話題> [--category X] [--tag Y] [--since YYYY-MM-DD]
```

| Slack コマンド | SQL の WHERE | LLM への質問 |
|---|---|---|
| `/post-draft Notion AI` | （フィルタなし） | "Notion AI" に近いチャンク |
| `/post-draft Notion AI --category "AI Integration"` | `category = 'AI Integration'` | 同上 |
| `/post-draft 副業の単価 --tag "Monetizing Skills"` | `'Monetizing Skills' = ANY(tags)` | "副業の単価" に近いチャンク |
| `/post-draft 最近のNotion論 --since 2026-03-01` | `created_at >= '2026-03-01'` | "Notion 論" に近いチャンク（直近のみ） |

→ **「絞り込みは確定情報、最終ランキングは意味」** の役割分担が UI レベルでもキレイに見える。

## みずきデータでの具体例

### ① 「最近の自分の AI 観だけ」で投稿
```sql
WHERE category = 'AI Integration'
  AND created_at >= '2026-03-01'
ORDER BY embedding <=> embed('AIへの考え方')
```
→ 古い AI 観（2 月）が混ざらず、**今の自分**で書ける。

### ② 「マネタイズ系の話題だけ」で副業ポスト
```sql
WHERE 'Monetizing Skills' = ANY(tags)
ORDER BY embedding <=> embed('副業の始め方')
```
→ コンサル論や属人化の話に混ざらず、お金の話に絞れる。

### ③ 「Notion AI への懸念点」だけ引きたい
```sql
WHERE category = 'AI Integration'
ORDER BY embedding <=> embed('Notion AI 課金 懸念 限界')
```
→ ポジ語だけだと拾えない懸念ポイントが浮上。

### ④ 「完了した思考」だけ参照
```sql
WHERE status = '完了'
ORDER BY embedding <=> embed(任意のテーマ)
```
→ 思考途中の発言を排除、整った主張だけで投稿。

## メタデータは LLM 側にも渡す

検索で引いた**チャンクとメタデータをセットで** LLM に渡すと、ポストの質がさらに上がる。

```
[context to LLM]
- 2026-03-21 / カテゴリ: AI Integration
  本文: 「Notion AIは…ワークスペース内でのエキスパートとして…」
- 2026-03-17 / カテゴリ: AI Integration
  本文: 「Notion AIを使ったリサーチの利点について…」
- 2026-02-25 / カテゴリ: AI Integration
  本文: 「AIに頼りすぎることの危険性…」

[user prompt]
これらは私（みずき）が録音した自分の発言です。
最も新しい考え方を中心に、200 字の X ポストを 3 つ作ってください。
```

→ LLM は **「いつの発言」「どのカテゴリ」** を踏まえて、新しい主張を優先的に選んでくれる。**メタデータは検索にも LLM の文脈にも使う、二重活用**。

## ひとこと要約

> **メタデータ＝鋭い包丁（確定値で素早く絞る）**
> **エンベディング＝拡大鏡（意味の濃淡で並べ替える）**
> **両方を順番に使うのが、本物の RAG**。

埋め込みに何でも混ぜたら、包丁を捨てて拡大鏡だけで料理しているのと同じ。

---

# 補講3：Claude Code 単体ならどこまで足りるか／Supabase で何をどう分けるか

## Q1：Claude Code 上で使う分にはセマンティック検索 OK？

**Yes、Claude Code 単体運用なら自前 embedding 不要**。

Claude Code の Notion MCP（`mcp__claude_ai_Notion__notion-search`）は内部で Notion のベクトル検索エンジンを叩いている。

| やり方 | セマンティック検索 | 自前 embedding |
|---|---|---|
| **Claude Code + Notion MCP** | ◯（Notion 側で実行） | **不要** |
| **Slack Bot / 他アプリから呼ぶ** | × Notion 内部API は外部公開なし | **必要** |
| **細かい制御**（閾値・複数ベクトル・差分計算） | × 不可 | **必要** |

→ **「自分が Claude Code から触る分にはこれで足りる」「外部に流すなら自前」** が境目。
→ 今日エンベディングを自前で組んだ意味は、**Slack や他アプリにも流せる土台ができた**点にある。

> **claudeからの補足**：Notion MCP のセマンティック検索はブラックボックス。閾値も次元も選べない、なぜそのページが上位かの根拠も取れない。**Reflection Loop を回したい**なら、自前で持ってログを残せる仕組みのほうが強い。

## Q2：Supabase ではベクトル化したい要素を「選択」できる？

**Supabase が自動で選んでくれるわけではない**。**選ぶのはあなたの投入コード側**。

### 構造

```
[Notion 側 — 1 ページの中身]
  名前        : "Notion AIの真価:..."
  テキスト    : "Notion AIは、特に..."  ← メイン本文
  カテゴリー  : "AI Integration"
  タグ        : ["AI Integration", "Notion Consulting"]
  作成日時    : "2026-03-21"
  ステータス  : "完了"
        │
        │ ★ここで「人間が」決める
        │  どれを embed するか / どれをメタデータとして残すか
        ▼
[Supabase 側 — 1 行のレコード]
  ┌─────────────────────────────────────┐
  │ メタデータ列（plain column）          │
  │   title, category, tags, created_at,  │
  │   status, doc_id                      │
  ├─────────────────────────────────────┤
  │ 埋め込み対象列                        │
  │   text       = テキスト本文           │ ← embed API に送る素材
  │   embedding  = vector(1536)           │ ← API が返したベクトル
  └─────────────────────────────────────┘
```

つまり **Supabase のスキーマは "どこが意味検索か" を 1 個のベクトル列で決めるだけ**。
**何を embedding API に送るかは、あなたの投入コード側の責任**。

### 投入コード側で「選択」する

```python
page = notion.get_page(page_id)

# ★ここが "選択" の本体：何を embed するかをコードで決める
text_to_embed = page["テキスト"]                              # パターンA: 本文のみ
# text_to_embed = f"{page['名前']}\n{page['テキスト']}"       # パターンB: タイトル+本文
# text_to_embed = page["要約"]                                # パターンC: 要約のみ

vector = gemini.embed(text_to_embed, task_type="RETRIEVAL_DOCUMENT")

supabase.table("thought_chunks").insert({
    # メタデータ（embedしない、フィルタに使う）
    "doc_id":     page["id"],
    "title":      page["名前"],
    "category":   page["カテゴリー"],
    "tags":       page["タグ"],
    "created_at": page["作成日時"],
    "status":     page["ステータス"],

    # embed 対象＋ベクトル
    "text":       text_to_embed,        # 元テキストも残す（後で参照）
    "embedding":  vector,
}).execute()
```

→ **Supabase 側に "このカラムを embed して" という機能はない**。投入時にあなたがベクトル化してベクトル列に入れる。
→ 逆に言えば**自由度が高い**：本文だけ／タイトル+本文／要約だけ、好きな組み合わせを選べる。

### どれを embed するかの判断基準

| Notion フィールド | embed する？ | 理由 |
|---|:---:|---|
| **テキスト本文** | ◎ 必ず | 意味検索の本命 |
| **名前（タイトル）** | △ 状況による | 本文が短い時は混ぜる、長い時は不要 |
| **要約** | ○ 推奨 | 本文が長い時、要約だけ embed すると精度↑ |
| **カテゴリー** | × しない | 確定値、フィルタに使う |
| **タグ** | × しない | 同上 |
| **作成日時** | × しない | 範囲計算、ベクトルと相性悪い |
| **ステータス** | × しない | 確定値 |
| **URL / 関連 / ID** | × しない | 識別子、意味なし |

### 応用：複数のベクトル列を持つ

「タイトルでも検索したい / 本文でも検索したい」を両方やりたいなら**列を分ける**こともできる。

```sql
create table thought_chunks (
  ...
  title           text,
  title_embed     vector(1536),    -- ★タイトル用ベクトル
  text            text,
  text_embed      vector(1536),    -- ★本文用ベクトル
);
```

検索時に「タイトル類似度 30% + 本文類似度 70%」のような**マルチベクトル検索**ができる。

```sql
SELECT id,
       0.3 * (1 - (title_embed <=> $qvec))
     + 0.7 * (1 - (text_embed  <=> $qvec)) AS score
FROM thought_chunks
WHERE category = 'AI Integration'
ORDER BY score DESC
LIMIT 10;
```

→ ここまで来ると本格運用。最初は **本文 1 ベクトル**で十分。

## ひとこと要約

> **Supabase は "場所" を提供するだけ**。
> **何を embed するかは、Notion → Supabase に入れるあなたのコードで決める**。
> **メタデータとベクトルは "別の列" として共存**し、検索時に **WHERE で絞ってから ORDER BY embedding** で並べる。

Supabase は**引き出し付きの本棚**。「どの引き出しに何を入れるか」は、本を入れる**司書（あなたのスクリプト）の仕事**。

> **claudeからの補足**
> - 「**選択**」のレイヤーが 2 つあると整理するとブレない：
>   ① **どのフィールドを embed するか**（投入コードで決める）
>   ② **検索時にどのフィールドでフィルタするか**（クエリで決める）
> - Notion 側のフィールド名と Supabase 側の列名は揃えなくて OK。**英語名に統一**しておくと SQL を書きやすい（`title / category / tags / created_at`）。
> - 自動同期するなら **Notion → Webhook → Cloud Function → embed → Supabase**。Webhook がない Notion プランなら **cron で last_edited_time 差分**。

---

*Generated: 2026-04-26 / 関連メモ: エンべディングログ.md, ポスト案.md, 0425レポート.md*
