# プラグインシステム設計規約（分離版・取り込み判断は保留）

> **この文書の位置づけ**
>
> 本体のコーディング規約（ECMAScript / TypeScript / Node.js）から**意図的に切り出した**もの。
> 「クライアント本体に対してプラグインで機能を足す」という**特定のアーキテクチャを採る場合にのみ**成立する規則で、
> 全リポジトリが辿る主経路ではないため、本体規約の追記ゲート（普遍性）を満たさない。
>
> 出所は `libraries/nodejs/CODING_RULES.md`（main）の §7〜§8。本体規約へ取り込むかは後で決める。
> **採用するリポジトリでは、本体規約に加えてこの文書もロードする。**
>
> 相対 import の拡張子は本体規約に合わせて `.ts` で書いている（Node が実ファイルを解決するため）。

---

## 1. 適用条件

次のすべてに当てはまるときだけ、この文書を適用する。

- **薄い汎用ライブラリ**として設計しており、業務ロジックをコアに置かない
- コアに対して**後から機能を合成する**拡張点（`.use()` 等）を公開している
- 拡張の作者と、コアの作者が分かれうる

当てはまらない場合、この文書の規則は不要な間接層を増やすだけになる。

---

## 2. ライブラリ設計の規則

### 2.1 公開APIの境界

`index.ts` を唯一のエントリーポイントとして、利用者に見せる型・値を明示的に制御する。

```typescript
// index.ts — 公開するものだけを列挙する
export { SalesforceApiClient } from './SalesforceApiClient.ts';
export { SlackApiClient, SlackWebhookClient, SlackApiError } from './SlackClient.ts';
export type { BaseClient, Plugin, ResponseHandler } from './ApiClient.ts';
export type { Logger } from './LoggerFacade.ts';

// 内部実装は export しない（HttpCore, createClient, withBearerAuth, LoggerFacade 等）
```

### 2.2 依存の方向を一方向に保つ

```
plugins/   →  ApiClient.ts / httpTypes.ts（型のみ）
clients/   →  ApiClient.ts / HttpCore.ts / httpTypes.ts
plugins/ は clients/ を import しない
clients/ は plugins/ を import しない
```

横断的な依存が生じた場合は設計を見直す。

---

## 3. プラグインシステムの規則

### 3.1 Plugin 型

プラグインは `Plugin` 型として一級市民で定義する。クラスは使わない。

```typescript
// ApiClient.ts
export type Plugin<TResponse, TNew extends object> =
  (client: BaseClient<TResponse>) => TNew;
```

### 3.2 Plugin は純粋関数

プラグイン（ファクトリ）はステートレスな純粋関数とする。ファクトリ自体は副作用を持たない（返すメソッドが行う I/O は除く）。

汎用プラグインは外部依存を持たない。ただし bulk 系のように CSV パース等の外部ライブラリや `.use()` 非対応の直接呼び出しを要するものは例外とし、理由を §3.5 の要領でコメントに明示する。

```typescript
// ✅ 純粋関数
const greetPlugin: Plugin<unknown, { greet(): string }> =
  (_client) => ({ greet: () => 'hello' });

// ✅ 設定を受け取る場合はファクトリ（プラグインを返す関数）
const timeoutPlugin = (ms: number): Plugin<unknown, { withTimeout(): ... }> =>
  (client) => ({ ... });

// 使い方は統一される
client.use(greetPlugin);
client.use(timeoutPlugin(3000));
```

### 3.3 Plugin セットの設計

関連するプラグインをまとめる場合は `as const` でプラグインセットとして提供する。
型パラメータは利用者まで届ける（本体規約「型パラメータは検証とセットで使う」に従い、**検証を伴わない型パラメータは `as` と等価**である点に注意する）。

```typescript
// plugins/salesforce.ts
const soql = <TRow = unknown>(): Plugin<unknown, {
  query(q: string): Promise<{ records: TRow[]; totalSize: number; done: boolean }>;
}> => (client) => ({
  query: (q) => client.get('/query', { q }) as Promise<...>,
});

const sobject = <TRecord = unknown>(type: string): Plugin<unknown, {
  findById(id: string): Promise<TRecord>;
  create(data: Partial<TRecord>): Promise<{ id: string }>;
  update(id: string, data: Partial<TRecord>): Promise<void>;
  delete(id: string): Promise<void>;
}> => (client) => ({ ... });

export const SalesforceApiClientPlugins = { soql, sobject } as const;
```

利用例:

```typescript
type Account = { Id: string; Name: string };

const sf = SalesforceApiClient.create(url, token)
  .use(SalesforceApiClientPlugins.soql<Account>())
  .use(SalesforceApiClientPlugins.sobject<Account>('Account'));

const res = await sf.query('SELECT Id, Name FROM Account');
// res.records: Account[]  ← 型が付く
```

> **セット全体への `satisfies` は使えない。** 引数なしのプラグイン（`soql`）には
> `satisfies Record<string, (...args: never[]) => Plugin<unknown, object>>` が適用できるが、
> 必須引数を持つプラグイン（`sobject(type: string)`）は `never[]` を満たせない。
> セット全体に適用するには引数ありのプラグインをファクトリパターンから外す必要があり、
> 設計トレードオフになる。`as const` で足りる。

### 3.4 プラグインセットと疎結合

プラグインセット（`plugins/`）はコアの型のみに依存し、各クライアント実装を import しない。
API クライアント固有の知識（エンドポイントパス等）はプラグイン内に閉じ込め、コアに漏らさない。

```typescript
// ✅ plugins/salesforce.ts
import type { Plugin } from '../ApiClient.ts'; // コアのみ

// ❌ クライアント実装を import してはいけない
import { SalesforceApiClient } from '../SalesforceApiClient.ts';
```

### 3.5 `as` キャストはプラグイン内に閉じ込める

`client.get()` の戻り値は `TResponse`（場合によっては `unknown`）。これをドメイン型に変換するための `as` キャストはプラグイン実装の内部に留め、利用者には型付きの結果のみを公開する。

キャストの理由は、**なぜその形が保証されるのか**をコメントに書く（本体規約の許容枠に従う）。

```typescript
query: (q) =>
  // BaseClient<unknown> の get 戻り値を SoqlResult<TRow> にキャスト
  // SF /query エンドポイントは必ずこの形を返すことが SF API 仕様で保証される
  client.get('/query', { q }) as Promise<SoqlResult<TRow>>,
```

---

## 4. 本体規約との関係

この文書は本体規約を**上書きしない**。競合したら本体規約が優先する。

本体規約側に既に取り込まれており、**ここでは繰り返さない**もの:

| 内容 | 移った先 |
|---|---|
| Logger をインターフェースで公開する（`logger: unknown` を使わない） | 本体「ライブラリ境界」 |
| 型パラメータは検証とセットで使う（検証なしの型パラメータ = `as`） | 本体「モジュール構造と境界」 |
| 型で証明できない合成（スプレッド等）は理由をコメントに書く | 本体「例外の明文化」 |
