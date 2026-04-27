# Superpowers 使用説明書

## 1. これは何か

**Superpowersは「Claude Codeに開発規律を仕込む」プラグイン**です。
コードを書く前に設計→計画→TDD→レビューを自動で回し、長時間放置でも品質を保つことを目指します。

- 作者：Jesse Vincent (obra) / Prime Radiant
- GitHub Star：40,000+
- 採用：Anthropic公式マーケットプレイス（2026年1月〜）
- リポジトリ：https://github.com/obra/superpowers

---

## 2. インストール状況

- **scope**：user（全プロジェクトで有効）
- **バージョン**：5.0.7（インストール時点）
- **形式**：プラグイン（MCPサーバー＋スキル＋スラッシュコマンド＋エージェント）
- **マーケットプレイス**：obra/superpowers-marketplace

確認方法：
```bash
claude plugin list
```

期待される出力：
```
❯ superpowers@superpowers-marketplace
  Version: 5.0.7
  Scope: user
  Status: ✔ enabled
```

---

## 3. 何が起きるか（ビフォーアフター）

### Before（通常のClaude Code）
```
ユーザー：「○○機能を作って」
   ↓
Claude：いきなりコード生成
   ↓
動かない / 設計甘い / レビューなし
```

### After（Superpowers有効）
```
ユーザー：「○○機能を作って」
   ↓
Claude（自動でスキル発動）：
  ① brainstorming → 要件を質問で深掘り
  ② write-plan → 実装計画を文書化
  ③ TDD → 先にテストを書く（RED）
  ④ 実装（GREEN）→ リファクタ（REFACTOR）
  ⑤ requesting-code-review → セルフレビュー
  ⑥ Critical問題があれば自動巻き戻し
```

---

## 4. 主要スキル

### コア5スキル

| スキル | 発動タイミング | 役割 |
|--------|--------------|------|
| **brainstorming** | コード生成依頼前 | 設計を質問で固める。代替案を提示 |
| **write-plan** | 設計確定後 | 実装計画書を作成 |
| **execute-plan** | 計画承認後 | 計画通りに実装、逸脱を検知 |
| **test-driven-development** | コード生成中 | テスト先行を強制（守らないとコード削除） |
| **requesting-code-review** | タスク完了後 | セルフレビュー、Critical問題はブロック |

### サポートスキル

| スキル | 役割 |
|--------|------|
| skills-search | 状況に応じて適切なスキルを発見 |
| SessionStart context injection | セッション開始時に文脈を自動注入 |
| 他、計20以上のスキル | 詳細はGitHubのREADMEを参照 |

---

## 5. 使い方

### 方法A：スラッシュコマンド（明示的に発動）

```
/brainstorm
→ ふわっとしたアイデアを設計に落とすセッション開始

/write-plan
→ 実装計画書を作成

/execute-plan
→ 計画に沿って実装＋自動レビュー
```

### 方法B：自動発火（推奨）

何も意識しなくてOK。Claudeが文脈を読んで適切なスキルを自動で発動する。

```
ユーザー：「ログイン機能を作りたい」
   ↓（自動で）
Claude：「まずbrainstormingで設計を固めましょう。
        認証方式は？セッション管理は？...」
```

### 方法C：明示的指示

```
「TDDで実装して」
「先にbrainstormで要件を整理しよう」
「実装後にcode reviewを通して」
```

---

## 6. ワークフロー実例

### 例：「TODOアプリのバックエンドAPIを作って」

```
1. /brainstorm
   Claude：「データモデルは？認証は？永続化は？...」
   → 質問応答で設計を確定

2. /write-plan
   Claude：plan.md生成
   → API仕様、エンドポイント一覧、テスト戦略

3. /execute-plan
   Claude：
     - tests/auth.test.ts を先に書く（RED）
     - auth.ts を実装（GREEN）
     - リファクタ
     - 次のタスクへ
     - 全タスク完了後、自動code review
```

---

## 7. 向いている / 向いていないケース

### 向いている
- 本番投入する**業務アプリ・SaaSの開発**
- **長時間放置（1〜2時間）で自律実装させたい**
- TDD・コードレビュー文化を**身につけたい**
- 1人開発で**第三者レビューが不在**
- **品質>速度**のプロジェクト

### 向いていない
- 使い捨てスクリプト・プロトタイプ
- 1行修正・タイポ修正
- **コーディング以外**（執筆、リサーチ、動画編集等）
- とにかく速く動かしたい場面

---

## 8. 注意点・デメリット

| 観点 | 内容 | 対処 |
|------|------|------|
| **初動が遅い** | brainstorm→plan→TDDで工程増 | 小タスクは無効化 |
| **トークン消費増** | 各スキルで文脈膨張 | 月API料金増を見込む |
| **強制力が強い** | TDD守らないとコード削除 | 慣れるまではイラっとする |
| **発火のうるささ** | 何にでもスキルを使おうとする | 「普通に書いて」で抑制可 |
| **長文化** | 計画書やレビューで応答が長くなる | チャンク分けで対応 |

---

## 9. 一時的に無効化する方法

### 単発で抑制する
プロンプトに「**brainstormingなしで**」「**普通に書いて**」と指定。

### プラグイン全体を無効化
```bash
claude plugin disable superpowers@superpowers-marketplace
```

### 再有効化
```bash
claude plugin enable superpowers@superpowers-marketplace
```

---

## 10. 更新・アンインストール

### 最新版に更新
```bash
claude plugin update superpowers@superpowers-marketplace
```

### アンインストール
```bash
claude plugin uninstall superpowers@superpowers-marketplace
```

### マーケットプレイスごと削除
```bash
claude plugin marketplace remove superpowers-marketplace
```

---

## 11. ユーザーの利用状況に対する推奨

このマシンの主な利用形態：
- 動画編集（edit-short）
- 記事執筆（note-writer, pivot-script-writer）
- リサーチ（notion-topic-scout）
- 事業計画（business-plan）

**Superpowersはコーディング特化のため、上記用途では発動を抑制した方がスムーズ**。
コーディングタスクのときだけ恩恵を受ける運用が理想。

### 推奨運用パターン
1. デフォルトはenable状態のまま
2. 非コーディング作業時は「brainstorming不要」と明示
3. 本格コーディングプロジェクトでは何もせず自動発火に任せる

---

## 12. トラブルシュート

| 症状 | 対処 |
|------|------|
| スキルが発動しない | Claude Code再起動 |
| 発動がうるさすぎる | プロンプトで明示的に抑制 |
| `/brainstorm` が認識されない | `claude plugin list`で有効か確認 |
| バージョンが古い | `claude plugin update` |
| 想定外の挙動 | issueをGitHubで確認、ログを採取 |

---

## 13. 公式リソース

- GitHub（プラグイン本体）：https://github.com/obra/superpowers
- マーケットプレイス：https://github.com/obra/superpowers-marketplace
- 作者ブログ：https://blog.fsck.com/2025/10/09/superpowers/
- 日本語解説（Qiita）：https://qiita.com/nogataka/items/c2e73515e65533986421

---

## メモ

- インストール日：2026-04-26
- インストールバージョン：5.0.7
- 設置者：Claude Code（Opus 4.7）が `claude plugin install` で導入
- 反映タイミング：次回Claude Code起動時から
