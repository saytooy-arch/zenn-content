---
title: "Next.js + Zodで「入力内容を確認してください」を卒業する — ApiError.detailsパターン"
emoji: "🔍"
type: "tech"
topics: ["nextjs", "zod", "typescript", "ux"]
published: true
---

## はじめに

「入力内容を確認してください」— フォーム送信時にこのメッセージだけ表示されて、どのフィールドが問題なのかわからない経験はないだろうか。

本記事では、ZodのsafeParseが返すフィールド単位のエラー情報を、フロントエンドまで正しく伝搬させるパターンを紹介する。

## 問題: APIはdetailsを返しているのにフロントが捨てている

### APIルートハンドラー（正しい）
Zodの `safeParse` は失敗時に `error.issues` 配列を返す。これをフィールドごとのdetailsに変換してレスポンスに含めるのは一般的なパターン:

```typescript
const parsed = Schema.safeParse(body);
if (!parsed.success) {
  const details: Record<string, string> = {};
  for (const issue of parsed.error.issues) {
    const key = issue.path.join(".");
    if (!details[key]) details[key] = issue.message;
  }
  return NextResponse.json({
    success: false,
    error: { code: "VALIDATION_ERROR", message: "入力内容を確認してください", details }
  }, { status: 400 });
}
```

### ApiErrorクラス（問題あり）
しかし、フロントのAPIクライアントがdetailsを保存していなければ意味がない:

```typescript
// Before: detailsが捨てられている
export class ApiError extends Error {
  code: string;
  status: number;
  constructor(message: string, code: string, status: number) {
    super(message);
    this.code = code;
    this.status = status;
  }
}
```

## 解決: ApiErrorにdetailsプロパティを追加

```typescript
export class ApiError extends Error {
  code: string;
  status: number;
  details?: Record<string, string>;

  constructor(message: string, code: string, status: number, details?: Record<string, string>) {
    super(message);
    this.name = "ApiError";
    this.code = code;
    this.status = status;
    this.details = details;
  }
}
```

apiFetchでdetailsを型安全に取り出す:

```typescript
if (!res.ok) {
  const message = json.error?.message ?? "エラーが発生しました";
  const code = json.error?.code ?? "UNKNOWN";
  const details =
    json.error?.details != null && typeof json.error.details === "object"
      ? (json.error.details as Record<string, string>)
      : undefined;
  throw new ApiError(
    typeof message === "string" ? message : "エラーが発生しました",
    code, res.status, details
  );
}
```

## フロントでのフィールド単位エラー表示

```tsx
} catch (err) {
  if (err instanceof ApiError) {
    if (err.details && Object.keys(err.details).length > 0) {
      const fieldErrors = Object.entries(err.details)
        .map(([field, msg]) => `${field}: ${msg}`)
        .join("\n");
      setError(`${err.message}\n${fieldErrors}`);
    } else {
      setError(err.message);
    }
  }
}
```

`whitespace-pre-line` で改行を表示:

```tsx
<p className="text-sm text-red-600 bg-red-50 p-3 rounded-lg whitespace-pre-line">{error}</p>
```

## 効果: エラー詳細が根本原因発見を加速した

この改善を入れた直後、ユーザーから「category: 有効なカテゴリを選択してください」という具体的なエラーが報告された。これにより、DBに`未分類`というenum外の値が存在していた根本原因を即座に特定できた。

汎用メッセージのままだったら、原因特定にもっと時間がかかっていただろう。

## まとめ

- Zodの `safeParse` は豊富なエラー情報を返してくれる
- APIクライアントでdetailsを捨てずに保存する
- フロントでフィールド単位のエラーを表示する
- エラー情報の透明化は、デバッグ効率を劇的に向上させる
