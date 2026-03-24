---
title: "TypeScriptのsatisfiesでドメイン定数の二重管理を防ぐ"
emoji: "🔒"
type: "tech"
topics: ["typescript", "nextjs", "設計"]
published: false
---

## はじめに

コードベースの中で同じ値が複数箇所に定義されている — いわゆる「定数の二重管理」問題。一方を更新しても他方が追従しないため、予期しないバグを生む。

本記事では、TypeScript 4.9で導入された`satisfies`演算子を使って、ドメイン定数の不整合をコンパイル時に検出するパターンを紹介する。

## 問題: 3箇所に散在したカテゴリ定義

```typescript
// schemas.ts（バリデーション用）
const CATEGORIES = ['肉類', '魚介類', '野菜', 'その他'] as const;

// page.tsx（UI表示用）— 独自定義
const CATEGORIES = ["肉類", "魚介類", "野菜", "その他"] as const;

// service.ts（DB挿入時）— リテラル直接使用
category: "未分類"  // ← enumに存在しない！
```

`service.ts` が `"未分類"` で挿入し、`schemas.ts` のenumに含まれていないため、その食材を更新しようとするとバリデーションエラーになった。

## 解決策1: Single Source of Truth + export

まず、定数を1箇所で定義してexportする:

```typescript
// schemas.ts
export const INGREDIENT_CATEGORIES = [
  '肉類', '魚介類', '野菜', '調味料', '乳製品', '穀物', 'その他', '未分類'
] as const;
```

フロントは独自定義を削除してimport:

```typescript
// page.tsx
import { INGREDIENT_CATEGORIES } from "@/lib/validators/schemas";
```

## 解決策2: satisfiesで型安全ガード

サービス層でリテラル文字列を使う場合、`satisfies`でenum内の値であることをコンパイル時に保証する:

```typescript
import { INGREDIENT_CATEGORIES } from "@/lib/validators/schemas";

category: "未分類" satisfies typeof INGREDIENT_CATEGORIES[number]
```

もし `"未分類"` がenumから削除された場合:

```
Type '"未分類"' does not satisfy the expected type '"肉類" | "魚介類" | ...'
```

コンパイルエラーになり、不整合が即座に検出される。

## asやキャストとの違い

```typescript
// ❌ as — 嘘をつける（実行時エラー）
category: "未分類" as IngredientCategory

// ❌ 型注釈 — 値を広げてしまう
const cat: IngredientCategory = "未分類"

// ✅ satisfies — 値の型を狭めたまま、制約を検証
category: "未分類" satisfies typeof INGREDIENT_CATEGORIES[number]
```

`satisfies`は値の型を変えずに「この型を満たすか」をチェックする。`as`と違って嘘をつけない。

## コーディング規約への追加

この問題の再発防止として、コーディング規約に以下を追加した:

> ドメイン値（カテゴリ、単位等のenum的な値の集合）は `src/lib/validators/schemas.ts` に定義し、他ファイルからはimportして使用する。リテラル文字列の直接使用を禁止。

## まとめ

- ドメイン定数は1箇所で定義してexport（Single Source of Truth）
- リテラル使用が避けられない場合は `satisfies` で型安全ガード
- コーディング規約で「リテラル禁止、import必須」を明文化
