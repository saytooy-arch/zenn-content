---
title: "vitestのサイレントPASS問題 — passWithNoAssertionsを知らないと品質ゲートが空洞化する"
emoji: "🧪"
type: "tech"
topics: ["vitest", "テスト", "品質管理", "CI"]
published: true
---

## TL;DR

vitestはデフォルトで **アサーション0件のテストをPASS** にする。`if (!ready) { return; }` のようなサイレントスキップパターンがあると、テストが何も検証していないのにPASS判定され、品質ゲートが空洞化する。`passWithNoAssertions: false` を設定すれば防げる。

## 何が起きたか

飲食店向け原価計算SaaSの開発プロジェクトで、ITテスト（結合テスト）7件が全てPASSしていた。しかし後日調査すると、**テスト実行時にdev serverが起動していなかった**。

テストコードはこう書かれていた:

```typescript
const serverAvailable = await checkServer();
if (!serverAvailable) {
  return; // ← ここでテスト関数が終了
}

// 以下のアサーションは一度も実行されない
const response = await fetch('/api/...');
expect(response.status).toBe(200);
```

`return` でテスト関数が正常終了すると、vitestは **アサーションが0件でもPASS** と判定する。SKIPですらない。テスト結果レポートには7件全PASSと表示され、品質ゲートの署名まで通ってしまった。

## なぜPASSになるのか

vitestの `passWithNoAssertions` 設定がデフォルトで `true` だからだ。

```typescript
// vitest.config.ts のデフォルト動作
export default defineConfig({
  test: {
    passWithNoAssertions: true, // ← これがデフォルト
  },
});
```

この設定が `true` の場合:
- `expect()` が一度も呼ばれなくてもPASS
- `return` による早期終了でもPASS
- 空のテスト関数 `it('todo', () => {})` もPASS

Jestも同じデフォルト動作をする。

## 対策

### 1. `passWithNoAssertions: false` を設定

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    passWithNoAssertions: false,
  },
});
```

これでアサーション0件のテストはFAILになる。

### 2. サイレント `return` パターンを禁止

テストの前提条件が満たされない場合は、黙って `return` するのではなく明示的にスキップまたは失敗させる:

```typescript
// NG: サイレントreturn（PASSになる）
if (!serverAvailable) {
  return;
}

// OK: 明示的にスキップ
if (!serverAvailable) {
  it.skip('server not available');
  return;
}

// OK: 明示的にFAIL
if (!serverAvailable) {
  throw new Error('Dev server is not running. Start it with `npm run dev`.');
}
```

### 3. CIで最低アサーション数をチェック

テスト結果のJSONレポートを解析し、アサーション数が0のテストがないか確認するスクリプトを追加する。`passWithNoAssertions: false` で十分だが、二重チェックとして有効。

## 教訓

- **テストが通っている ≠ テストが検証している**。アサーション0件のPASSは「何もチェックしていない」と同義
- vitestやJestを使う場合は、プロジェクト初期に `passWithNoAssertions: false` を設定すべき
- テスト前提条件の不足は、サイレントに握りつぶすのではなく、明示的にFAILまたはSKIPにすべき
- 品質ゲートがテスト結果のPASS/FAIL数だけを見ている場合、この問題は検出できない。アサーション数も確認する仕組みが必要

## 参考

- [Vitest Configuration - passWithNoAssertions](https://vitest.dev/config/#passwithnoassertions)
- [Jest Configuration - passWithNoTests](https://jestjs.io/docs/configuration#passwithnotests-boolean)
