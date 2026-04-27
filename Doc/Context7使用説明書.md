# Context7 使用説明書

## 1. これは何か

**Context7はライブラリの最新公式ドキュメントをLLMに食わせるMCPサーバー**です。
LLMの学習カットオフ問題（古い情報・存在しない関数・ハルシネーション）を解決します。

提供：Upstash社
公式：https://context7.com

---

## 2. インストール状況

- **scope**：user（全プロジェクト共通）
- **形式**：MCPサーバー単体（プラグイン版ではない）
- **設定ファイル**：`~/.claude.json`
- **コマンド**：`npx -y @upstash/context7-mcp`

確認方法：
```bash
claude mcp list | grep context7
```

期待される出力：
```
context7: npx -y @upstash/context7-mcp - ✓ Connected
```

---

## 3. 基本の使い方

プロンプトの末尾に **`use context7`** を付けるだけ。

### 例1：新しいフレームワークの仕様を聞く
```
Next.js 15 App Routerのparallel routesの実装方法を教えて。 use context7
```

### 例2：バージョン指定
```
React Hook Form v7のresolverの書き方を教えて。 use context7
```

### 例3：マイナーOSS
```
Drizzle ORMでpostgresのトランザクションを書くサンプル。 use context7
```

---

## 4. 利用可能なツール

Claude Codeから自動で呼ばれる2つのツール。

| ツール名 | 役割 |
|---------|------|
| `mcp__context7__resolve-library-id` | ライブラリ名から内部IDを解決 |
| `mcp__context7__get-library-docs` | 解決したIDで最新ドキュメントを取得 |

通常は意識しなくてOK。`use context7`と書けば自動で順次呼ばれる。

---

## 5. 効果が高い場面

- **新しいフレームワーク**：Next.js 15, React 19, Vue 3.5, Astro最新版
- **API破壊変更が多いライブラリ**：LangChain, TailwindCSS, Convex, Drizzle
- **マイナーだがドキュメントがしっかりしているOSS**：tRPC, Hono, ElysiaJS
- **バージョン違いで挙動が変わるもの**

## 効果が薄い場面

- 自社内製ライブラリ（クロール対象外）
- ドキュメントが薄いマイナーOSS
- 設計思想・哲学的な議論
- "なぜか動かない"系のトラブルシュート

---

## 6. 料金・無料枠

**2026年1月の改定で大幅縮小**されたので注意。

| 項目 | 内容 |
|------|------|
| 月間リクエスト | **1,000回**（旧6,000回） |
| 時間あたり上限 | **60リクエスト/時** |
| API key | 不要（無料枠） |
| 有料プラン | レート上限緩和、商用サポート |

ヘビーユースだとすぐ枯れる。常用するなら有料プラン or ローカル代替（Context7ローカル版がOSSで存在）を検討。

---

## 7. 運用Tips

### TIP 1：使い分け
- 新規ライブラリを触るとき → ON
- 安定した枯れ技術 → OFF（トークン消費の無駄）

### TIP 2：トークン消費に注意
ドキュメント全文を注入するため、1リクエストでのコンテキスト消費が増える。

### TIP 3：プロンプトインジェクションの理論的リスク
外部から取ってきたドキュメントなので、悪意ある内容が混じる可能性は理論上ある（実害報告は今のところなし）。

---

## 8. アップグレード：プラグイン版

現在はMCPサーバー単体版。フル機能版（auto-trigger + docs-researcherサブエージェント）にアップグレードしたい場合：

Claude Codeに**自分で**入力：
```
/plugin marketplace add upstash/context7
/plugin install context7-plugin@context7-marketplace
```

プラグイン版になると：
- `use context7`を書かなくても自動発火
- 専用エージェントがドキュメント取得を担当（メインのコンテキストを汚さない）

---

## 9. アンインストール

```bash
claude mcp remove context7 --scope user
```

確認：
```bash
claude mcp list
```

---

## 10. トラブルシュート

| 症状 | 対処 |
|------|------|
| `use context7`が効かない | Claude Codeを再起動 |
| `Connection failed` | `npx -y @upstash/context7-mcp` を直接実行して動作確認 |
| レート制限エラー | 1時間待つ、または有料プラン検討 |
| 古い情報が返る | `/refresh` 後にretry、またはGitHub直リンク版を検討 |

---

## 11. 公式リソース

- 公式サイト：https://context7.com
- GitHub：https://github.com/upstash/context7
- Claude Code向けドキュメント：https://context7.com/docs/clients/claude-code

---

## メモ

- インストール日：2026-04-26
- バージョン：MCPサーバー単体版（プラグイン版ではない）
- 設置者：Claude Code（Opus 4.7）が `claude mcp add --scope user` で導入
