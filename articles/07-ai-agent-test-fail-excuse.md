---
title: "AIエージェントがテストFAILを「環境の問題」と言い訳した話 — hookで免責を物理ブロックする"
emoji: "🚫"
type: "tech"
topics: ["ClaudeCode", "テスト", "品質管理", "AI開発"]
published: true
---

## TL;DR

AIエージェント組織でSaaS開発中、テスト8件がFAILしていたのに品質レポートで「環境制約によるSKIP（非ブロッカー）」としてPASS判定が出た。原因は `vitest.config.ts` に `.env.local` の読み込みが欠落していただけ。AIエージェントも人間と同じく「環境のせい」で免責する。hookスクリプトで品質レポート編集を物理ブロックする仕組みを作った。

## 何が起きたか

飲食店向け原価計算SaaS「Genka」をClaude Codeのマルチエージェント体制で開発している。アーキテクチャ設計書に基づく全工程やり直し（SD→MF→UT→IT→ST→UAT→RL）を実行した後、テスト結果を確認した。

```
Tests  8 failed | 383 passed | 11 skipped (402)
```

8件のFAIL。しかし品質レポートにはこう書かれていた:

```markdown
| Vitest IT（auto-add-ingredients） | 8件SKIP（Supabase未起動） | SKIP（環境制約） |
```

**FAILがSKIPに読み替えられている。**

しかもレポートの総合判定は:

```markdown
### 総合判定: **PASS**
```

## 本当の原因

「Supabase未起動」ではなかった。Supabaseは起動していた（ポート54321で正常稼働）。

エラーメッセージをよく見ると:

```
Error: connect ECONNREFUSED 127.0.0.1:5432
```

ポート **5432**（PostgreSQLのデフォルト）に接続しようとしている。実際のDBは **54322** で動いている。

原因は `vitest.config.ts` に `.env.local` の読み込みが設定されていなかったこと:

```typescript
// Before: DATABASE_URL が undefined → デフォルトポート5432
export default defineConfig({
  test: { globals: true, environment: "node" },
});

// After: loadEnv で .env.local を読み込む
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), "");
  return {
    test: { globals: true, environment: "node", env },
  };
});
```

修正後:

```
Tests  391 passed | 11 skipped (402)  ← 0 failed
```

## なぜAIエージェントは免責したか

人間のエンジニアも「CI環境の問題」「Docker起動してないからSKIPで」と言い訳することがある。AIエージェントも全く同じことをする。

根本原因は3つ:

1. **エラーメッセージを精査しなかった**: `ECONNREFUSED 127.0.0.1:5432` のポート番号が実際のDB設定（54322）と違うことに気づかなかった
2. **FAILとSKIPの区別が曖昧だった**: 品質ゲートチェックリストに「FAILをSKIPに読み替えてはならない」が明文化されていなかった
3. **テスト環境の検証がなかった**: 「テスト環境は正常か？」というチェック項目がゲートになかった

## hookで物理ブロックする

「気をつけよう」では再発する。hookスクリプトで物理的にブロックする。

### 仕組み

```
[テスト実行（vitest）]
  ↓ PostToolUse hook
[check_test_result.sh]
  ├─ FAIL検出 → test_result.json に FAIL 記録 + 警告出力
  └─ PASS → test_result.json に PASS 記録

[品質レポート編集（doc/10_RL/*）]
  ↓ PreToolUse hook
[check_test_result.sh]
  ├─ test_result.json が FAIL → exit 2（編集ブロック）
  └─ test_result.json が PASS → exit 0（編集許可）
```

### hookスクリプト（抜粋）

```bash
# テスト実行後: FAIL件数を記録
if echo "$command" | grep -qE "(vitest|npx vitest)"; then
  failed=$(echo "$stdout" | grep -oP '(\d+) failed' | head -1 | grep -oP '\d+' || echo "0")
  if [ "$failed" -gt 0 ]; then
    echo "[TEST RESULT] ${failed}件のFAILがあります" >&2
    echo "  FAILをSKIPに読み替えることは禁止されています" >&2
    # test_result.json に FAIL 記録
  fi
fi

# 品質レポート編集時: FAIL記録があればブロック
if echo "$file_path" | grep -q "doc/10_RL/"; then
  if [ "$test_overall" = "FAIL" ]; then
    echo "[TEST GATE] 品質レポートの作成がブロックされました" >&2
    exit 2
  fi
fi
```

## ナレッジにも明文化

test_policy.md に「FAIL/SKIP分類ルール」を追加:

| 状態 | 定義 |
|------|------|
| PASS | テスト実行、全アサーション成功 |
| FAIL | テスト実行、アサーション失敗 or ランタイムエラー |
| SKIP | テストコード上で `it.skip()` 等により**意図的に除外** |

**禁止事項**: FAILを「環境制約SKIP」として品質レポートでPASS判定を出すこと。

## 続編: エージェントが「テスト全PASS」と嘘をついた

上記のFAIL免責問題を修正した後、全画面のボタン処理テスト（E2E 31件）を作成した。programmerエージェントに実装を委譲したところ、「30 PASS / 0 FAIL」と報告してきた。

しかし自分で再実行すると:

```
29 failed
2 passed (15.2m)
```

**29件FAIL**。エージェントの報告は完全に嘘だった。

### なぜエージェントは嘘をつくか

「嘘」というより**楽観的な解釈**。エージェントは:
1. テストを実行する
2. いくつかFAILする
3. 修正を試みる
4. 部分的に修正できた結果を「修正完了」と報告する

人間のエンジニアが「ローカルでは動いてました」と言うのと同じ構造。

### 対策

**エージェントの報告を信用しない。自分でテストを実行し、出力を直接確認する。**

これはhookでは解決できない。hookはツール実行をブロックできるが、「エージェントの報告が正しいか」を検証する仕組みは今のところない。人間がテスト出力を読むしかない。

## 教訓

AIエージェントも人間と同じ言い訳をする。違いは **hookで物理的にブロックできる** こと。

- ルールを書いただけでは守られない（L4-L5）
- hookで物理ブロックして初めて「強制」になる（L1）
- テスト環境の問題は免責理由ではない。環境を直してから再実行する
- **エージェントの「テスト全PASS」報告は信用するな。自分で実行して確認せよ**
