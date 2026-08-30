# コーディング規約 (CODING_RULES.md) — ECMAScript / TypeScript / Node.js

> 迷ったら「**可読性**」と「**実行時の堅牢性**」を優先する。
>
> **前提**: Node は型注釈除去（type stripping）が stable な版以降。TypeScript・ESLint・Prettier・`node:test` を使う。バージョンの選び方は §8.1。
>
> **Node は `.ts` を型注釈の除去だけで実行する**（`tsconfig` を読まず、型検査もしない）。**実行が成功したことは型が正しいことの証明にならない。** §8 の**コンパイラのフラグ**が効くのは型チェックを回したときだけ。
>
> **非保証**: 本規約は設定ファイルを配布しない（§1.2）。§8 の要求事項が実際に効いているかは**各リポジトリで導入時に確かめる**（§8.3）。確認していないものを「機械が守っている」と扱わない。

---

## 1. 基本思想

二本柱で書く。

1. **Java ライクな堅牢性**: 明示的なブロック、厳密なエラーハンドリング、役割の分離（モジュール・境界・型による契約）。型システムは Java のインターフェース・ジェネリクス的な発想で使い、**コンパイラをペアプログラマーとして扱う**。
2. **Modern ES / TS の活用**: 型推論・Utility Types・async/await・ESM など、現代のイディオムを積極採用し、冗長な記述を避ける。

**型はコンパイル時のみ有効。** 外部入力（API レスポンス・設定値・`JSON.parse`）は型が付いていてもランタイムガードを省略しない。置く場所は**境界**（§1.1・§6.1）。

二本柱は次の節に展開される。

- 明示的なブロック → §5.1
- 厳密なエラーハンドリング → §6
- 決定的な後始末 → §5.5
- 役割の分離（モジュール・公開面・依存方向）→ §2.5〜§2.7
- 型による契約 → §2.8・§4

### 1.1 「明示」は境界に寄せる

明示は無条件の善ではない。**境界は明示、内部は推論**。OpenJDK の公式スタイルガイドも同じ立場（[LVTI Style Guide](https://openjdk.org/projects/amber/guides/lvti-style-guide) の P4「Explicit types are a tradeoff」）。§4.1（公開関数の戻り値型）と §6.1（ランタイムガードは境界に置く）に展開される。

### 1.2 規約が上位、設定は従属

規約文で「禁止」と書いても運用では無言に破られるので、**機械に守らせる**。ただし向きを間違えないこと。

- **規約は完全な形で本文に持つ。** 設定に落とせたからといって本文からルールを削らない
- **設定は「本規約に完全準拠させる」ことを目的に、各リポジトリで作る**（§8）
- **規約を設定に合わせて調整しない。** 設定が規約を満たせないなら、それは設定の不足である

> 逆向き（「機械が検出できるものは規約文から削る」）にしてはいけない。未検証の主張が「削除」という不可逆な操作を許可する構造になり、**規則の実体がどこにも無くなる**。

### 1.3 適用範囲 — 実行モデルで切る

**ファイルの拡張子ではなく、そのコードを何が実行するかで決める。** 拡張子で切ると、切り落とした側が完全な死角になり、違反だらけのコードが警告ゼロで通る。

| 実行モデル | 対象 | 適用範囲 |
|---|---|---|
| **Node が直接実行する** | `.ts` / `.mts` / `.cts`、ESM の `.js` / `.mjs` | **全節** |
| **バンドラを通す** | ブラウザ・renderer 向け。`.tsx` を含む | **§2.2 と、§8 のうち Node 実行を前提とする要求を除く全節** |
| **既存の CommonJS** | `require()` ベースで `"type": "module"` が無いもの | **対象外** |

§3（命名）・§4（型システム）・§5（構文）・§6（エラー戦略）は実行モデルに依存しない。バンドラ経由のコードにも同じようにかかる。除外されるのは §2.2（相対 import の拡張子。バンドラは拡張子なしを解決する）だけである。**lint の対象には `.tsx` を必ず含める**（§8.1）。

CommonJS を対象外にするのは、§2.1・§2.2・§8 がいずれも成立しないため。混在するリポジトリでは **「lint / typecheck の対象から外す」か「移行対象として計画を立てる」かを最初に決める**。決めないまま本規約をロードすると、AI が既存 CJS を規約違反と判定して壊しにかかる。

### 1.4 読む順序

§1〜§7 が規則、§8 が機械強制の要求事項。**実装を始める前に §8 の設定を先に用意すること。** 先にコードを書いて後から設定を入れると、`"type": "module"`・テストファイルの配置・optional の扱いで広範囲に手戻りする。

---

## 2. モジュール構造と境界

### 2.1 ES Modules

`export` を使う。IIFE パターンは使わない。

```typescript
// ✅
export const HttpCore = { createTransport, withRetry, withLogger };

// ❌
const HttpCore = (() => { ... })();
```

**`package.json` に `"type": "module"` が必須。** 無いとモジュール解決が CommonJS と判定し、`export` 一つにつきエラーが出て何も書けない。

- `.mts` は常に ESM、`.cts` は常に CommonJS

### 2.2 相対 import には `.ts` 拡張子を書く

> **適用**: Node が直接実行するコードのみ（§1.3）。バンドラ経由のコードには適用しない。

Node は型注釈を除去して**実ファイル**を解決するため、`.js` では解決できない。

```typescript
// ✅
import { HttpCore } from './HttpCore.ts';
import type { Transport } from './httpTypes.ts';

// ❌ 拡張子なし（解決できない）
import { HttpCore } from './HttpCore';

// ❌ .js（実ファイルが無いため実行時に ERR_MODULE_NOT_FOUND）
import { HttpCore } from './HttpCore.js';
```

Node 公式ドキュメント: 「file extensions are mandatory in `import` statements and `import()` expressions: `import './file.ts'`, not `import './file'`」

ビルドして `dist/` を配布する場合は、出力時に `.js` へ書き換える設定を使う（§8.1）。書き換わるのは**相対パス・リテラル指定・非宣言ファイル**のみで、`paths` エイリアス・`#subpath imports`・パッケージ指定・変数を渡す動的 `import()`・`.d.ts` の中身は対象外。

**拡張子なしを書くとコンパイラが指摘してくるが、その提案（`.js` を付けよ）には従わない。** 相対 import は `.ts` と書く。

**テストランナーとバンドラの制約**: `.ts` 拡張子は `node --test` と Vitest では動くが、**Jest 系（`ts-jest` / `sfdx-lwc-jest`）では動かない**（出力時の書き換えと resolver が噛み合わない）。Jest 系を使うリポジトリでは §2.2 を採用しない。

### 2.3 型のみの import は `import type`

循環参照の回避、出力から型を消す、意図の明確化。

```typescript
// ✅
import type { Logger } from './LoggerFacade.ts';
import { HttpCore } from './HttpCore.ts';
import { fn, type FnParams } from './fn.ts';

// ❌ 型のみなのに通常 import
import { Logger } from './LoggerFacade.ts';
```

### 2.4 ファイルヘッダー

アプリケーションコードは不要（エントリの TSDoc に注力）。ライブラリ・ユーティリティは冒頭に概要を1行。`'use strict'` は ESM では不要。

**ファイル名の行とタグ（`@` で始まる行）の間は1行あける。**

```typescript
/**
 * HttpCore.ts
 *
 * @description HTTP通信の共通基盤（Transport・デコレータ・ユーティリティ）
 */
```

### 2.5 公開面を明示的に制御する

`index.ts` を唯一のエントリーポイントとし、公開する型・値を列挙する。

```typescript
// index.ts — 公開するものだけを列挙する
export { SalesforceApiClient } from './SalesforceApiClient.ts';
export type { BaseClient, Plugin, ResponseHandler } from './ApiClient.ts';

// 内部実装は export しない（HttpCore, LoggerFacade 等）
```

### 2.6 依存の方向を一方向に保つ

上位が下位に降りて接続する向きを崩さない。横断的な依存が生じたら設計を見直す。

```
consumers/  →  core/（型のみ）
core/       は consumers/ を import しない
```

### 2.7 型パラメータは検証とセットで使う

`unknown` を公開 API に漏らさないために型パラメータを引き上げる。**ただし検証を伴わない型パラメータは `as` と等価**で、型検査を全面通過して実行時に落ちるコードになる。

```typescript
// ❌ 検証なしの型パラメータ = unknown を無検査で T に洗浄している
const client = SalesforceApiClient.create<SoqlResult>(url, token);
const result = await client.get('/query');   // 型は SoqlResult、実体は {"error":"INVALID_SESSION"}
// → result.records.length で TypeError。コンパイラは何も言わない

// ✅ 境界で decode / parse する
const client = SalesforceApiClient.create<SoqlResult>(url, token, isSoqlResult);
```

検証を省く場合は §4.7 の許容枠の対象になる。**スキーマが本当に存在しない場合（任意 JSON のパース等）は `unknown` のままが正しい。** ジェネリクスで確定できるのは、契約に基づいて安全に絞れる場合に限る。

### 2.8 差し替え可能な依存はインターフェースで契約する

ロガー・トランスポート・時計のように利用者が差し替える依存は、**`unknown` で受けない**。インターフェースを公開し、何を実装すればよいかを型で示す。Java のインターフェースと同じ役割。

```typescript
// ✅ インターフェースで契約する
export interface Logger {
  trace(...args: unknown[]): void;
  debug(...args: unknown[]): void;
  info(...args: unknown[]): void;
  warn(...args: unknown[]): void;
  error(...args: unknown[]): void;
}

// ❌ 利用者が何を渡せばよいか分からない
export const create = (options: { logger?: unknown }) => { ... };
```

構造的部分型なので、利用者は既存のオブジェクトをそのまま渡せる（`{ logger: console }`）。アダプタ実装（`LoggerFacade` 等）は内部に留め、公開するのはインターフェースだけにする。

---

## 3. 命名規則

| スコープ / 役割 | 規則 | 例 |
|---|---|---|
| **真の定数** | `UPPER_SNAKE_CASE` | `const MAX_RETRY_COUNT = 5;` |
| **設定値オブジェクト** | `UPPER_SNAKE_CASE` + `as const` | `const HTTP_STATUS = { OK: 200 } as const;` |
| **名前空間・モジュールオブジェクト** | `PascalCase` | `HttpCore`, `SalesforcePlugins` |
| **再代入不可な変数** | `camelCase` | `const currentUser = auth.getUser();` |
| **型・interface・クラス** | `PascalCase` | `interface Transport`, `type HttpMethod` |
| **型パラメータ** | `T` / `TXxx` | 単純なら `T`、意味が要れば `TResult` |
| 短いスコープ（1〜3行） | 1文字変数（推奨） | `k`, `v`, `e`, `n` |
| 通常スコープ | **省略禁止** | `options`（not `opts`）、`response`（not `res`） |
| 未使用の引数 | `_` 接頭辞 | `(_event, _context) => {}` |

`Object.freeze` はランタイム凍結が本当に要る場合のみ。`as const` はコンパイル時の型情報が得られる。

```typescript
// ✅
const CONFIG = { DEFAULT_MAX_RETRIES: 3, DEFAULT_BASE_DELAY_MS: 500 } as const;
```

---

## 4. 型システム

### 4.1 必須事項

| 対象 | ルール | 詳細 |
|---|---|---|
| **strict モード** | **必須** | これなしの TypeScript は型チェックが緩く意味が薄い |
| **`any`** | **値の型としては禁止** | 外部データは `unknown` で受け型ガードで絞る（§4.6）。ジェネリック制約位置は例外（§4.7） |
| **`as`（型キャスト）** | **原則禁止** | まず `satisfies` を検討する（§4.2）。使う場合の書き方は §4.7 |
| **`!`（非nullアサーション）** | **原則禁止** | 理由は §4.3。使う場合は理由コメント必須（§4.7） |
| **公開関数の戻り値型** | **明示必須** | `export` する関数・メソッドは戻り値型を書く。内部実装は推論に任せてよい（§1.1） |
| **`enum`** | **禁止** | `as const`（§3）+ Union 型（§4.5）で代替 |

`any` / `as` / `!` は **コンパイラでは検出されない**。lint が必須（§8.1）。

「公開関数」の範囲は、**`export` されるオブジェクト経由で到達可能な関数を含む**。§2.1 のモジュールオブジェクト `export const HttpCore = { createTransport, ... }` を使う場合、メンバ関数それぞれに戻り値型が必要になる（到達不能な純粋な内部関数だけが対象外）。

### 4.2 `as` の前に `satisfies` を検討する

`satisfies` は型を広げずに検査だけする。`as` は型検査を黙らせるので、代替がある場面では使わない。

```typescript
// ❌ as: 型が widen し、プロパティ欠落も通る
const CONFIG = { retries: 3 } as Config;

// ✅ satisfies: Config を満たすか検査しつつリテラル型を保つ（欠落は検出される）
const CONFIG = { retries: 3, delay: 500 } satisfies Config;
```

### 4.3 null 安全 — 「デフォルト非null」を型で持っている

strict の null チェックを有効にすると、型は既定で非 null になり、null 許容は `T | null` / `T | undefined` として明示される。Java は同じことをアノテーションで後付けしている（Spring Framework は [JSpecify](https://docs.spring.io/spring-framework/reference/core/null-safety.html) を採用し、`@NullMarked` で「デフォルト非null」を宣言してビルド時に強制する）。

**`!` を書く行為は、この保証を手で捨てることに等しい。** だから原則禁止にする。

> **インデックスアクセスを厳しくするフラグについて（誤解しやすい）**: このフラグは `!` を不要にするのではなく、**インデックスアクセスを `T | undefined` にしてチェックを強制する**。結果として `!` を書きたくなる箇所が増える方向に働く（正規表現キャプチャ・`split(' ')[0]`・分割代入・カウンタループ）。そのため**このフラグ由来の箇所では `!` を理由コメント付きで許容する**（§4.7）。

**ループ後の確定値**は型システムが追えない頻出パターン。`?.` や事前チェックでは解けないので、値が必ずある形に組み替える。

```typescript
// ✅ 最終試行をループの外に出し、失敗を明示的に伝える
for (let i = 0; i < maxRetries; i++) {
  const response = await transport.fetch(url, options);
  if (response.ok) {
    return response;
  }
  await sleep(backoff(i));
}
const last = await transport.fetch(url, options);
if (!last.ok) {
  // リトライを使い切った事実を呼び出し側に伝える（§5.4）
  throw new HttpError('リトライ上限に達しました', last.status, await last.text());
}
return last;
```

### 4.4 `interface` vs `type`

| 用途 | 使うもの | 理由 |
|---|---|---|
| **公開APIの契約** | `interface` | Java のインターフェース的な発想。拡張・実装を想定（§2.8） |
| **Union 型** | `type` | `type HttpMethod = 'GET' \| 'POST' \| ...` |
| **Utility Types の組み合わせ** | `type` | `type Options = Partial<Config> & { logger?: Logger }` |
| **関数型** | `type` | `type Filter = (v: unknown) => unknown` |

### 4.5 閉じた階層は discriminated union で表す

Union 型は「型の並び」ではなく、**閉じた階層（sealed hierarchy）の宣言**として扱う。判別可能にするため、必ず共通のリテラルフィールド（判別子）を持たせる。

```typescript
// ✅ 閉じた階層
type Result<T> =
  | { readonly kind: 'ok';    readonly value: T }
  | { readonly kind: 'error'; readonly error: Error };
```

Java の対応物は `sealed interface` + `record`。この節の目的は §5.2 の網羅性検査を成立させること。

### 4.6 外部データは `unknown` で受ける

```typescript
// ✅
const body: unknown = JSON.parse(text);
if (typeof body === 'object' && body !== null && 'access_token' in body) {
  // ここでは body.access_token にアクセス可能
}

// ❌ any（型チェックを完全に無効化）
const body: any = JSON.parse(text);
```

### 4.7 例外の明文化

「原則禁止」だけでは運用で無言に破られる。**許容枠を明示する。**

| 許容する場面 | 対象 | 書き方 |
|---|---|---|
| 外部 API の広いユニオンを期待型へ絞る | `as` | 何も書かない（lint は発火しない） |
| テストのモック生成 | `as unknown as` | 何も書かない（§7.1 の緩和下で発火しない） |
| ジェネリック制約位置の `(...args: any[]) => any` | `any` | 抑制コメント（反変位置で `unknown` は代替不能） |
| インデックスアクセスを厳しくするフラグ由来の確定インデックス | `!` | 抑制コメント + 理由 |
| 型システムで表現できない合成（スプレッド等） | `as unknown as` | 通常のコメントで理由 |
| 上記以外 | `as` / `!` / `any` | 発火するなら抑制コメント + 理由、しないなら通常のコメントで理由 |

抑制コメントは `// eslint-disable-next-line <rule> -- 理由`（`--` 以降が理由）。**ルールが発火しない箇所に抑制コメントを書くと、§8.1 が要求する「不要になった抑制コメントの検出」に当たってビルドが壊れる。** 書く前に、そのコードで実際に何が発火するかを確かめる。

```typescript
// ✅ 発火しないので通常のコメント（型システムで証明不能: additionalMethods ∪ HttpMethods）
client = { ...additionalMethods, ...httpMethods, call, extend, use } as unknown as BaseClient<...>;

// ✅ 発火するので抑制コメント + 理由
// eslint-disable-next-line @typescript-eslint/no-non-null-assertion -- 直前の length 検査で確定
const first = xs[0]!;
```

### 4.8 Node が実行できない TypeScript 構文

型注釈除去は「型を消すだけ」なので、JavaScript コード生成を伴う構文は動かない。**これらは使わない。**

| 構文 | 備考 |
|---|---|
| `enum` | §4.1 で禁止済み |
| パラメータプロパティ `constructor(public readonly x: T)` | 書き方は §6.4 |
| runtime code を含む `namespace` | runtime code を含まないものは動く |
| `import A = B.C` / `export = X` | — |
| **デコレータ `@foo`** | **消去可能構文だけを許すフラグを素通りする**。lint で塞ぐ（§8.1） |
| **`accessor` フィールド** | 同上 |

**変換フラグ（`--experimental-transform-types`）は使わない。** これらを動かす experimental フラグは存在するが、採用すると「Node が読めるものだけを書く」という前提が崩れ、§8.1 の設定と二重管理になる。

デコレータと `accessor` はコンパイラを通過して **Node で SyntaxError** になる。しかも TypeScript 固有のエラーとしてではなく、パーサが構文として認識しない形で落ちる。

### 4.9 厳しいフラグを増やすときの原則

**厳しいフラグを追加するときは、§4.7 の許容枠を対で用意する。用意できないなら採用しない。**

- インデックスアクセスを厳しくするフラグは、許容枠（フラグ由来の `!`）とセットで採用する
- **optional プロパティを厳密にするフラグ（`exactOptionalPropertyTypes`）は採用しない。** `{ retries: maybe }`・`({ signal })`・`{ ...base, ...patch }` が落ち、§5.6 が推奨するスプレッドと §6.5 の `options = {}` に正面から当たる。回避策3つ（全 optional に `| undefined` を足す／条件付きスプレッド／`as`）はいずれもこの規約と衝突する。**コンパイラの初期化テンプレートはこのフラグを既定で入れてくるので、生成物をそのまま使わない**
- **宣言ファイルを単独生成可能にするフラグ（`isolatedDeclarations`）は採用しない。** `export` される値に明示的な型注釈を要求するため、§2.1 のモジュールオブジェクトが落ちる。§4.1 の目的は lint 側のルールで足りる（§8.1）。`.d.ts` を並列生成したいライブラリ層でのみ局所的に有効化してよい

---

## 5. 構文・スタイル

### 5.1 必須事項

| 対象 | ルール |
|---|---|
| **ブロック省略** | `if (x) return;` 禁止 |
| **一行化** | `if (x) { return; }` を1行に畳まない。本文は改行・インデント（**整形ツールの担当**。lint では検出できない） |
| **`forEach`** | **禁止** → `for...of`（制御が要る場合）または Iterator Helpers（変換のみ） |
| **`var`** | **禁止**（`const` / `let`） |
| **`switch` のフォールスルー** | **禁止**。各 `case` に `break` または `return` |
| **Yoda 条件** | ✅ `if (value === null)` / ❌ `if (null === value)` |

> **`switch` もブロックスタイルの対象**（`if` / `for` / `while` 限定と誤解されやすい）。`case x: break;` の一行にせず、`case` / `default` の本文を改行・インデントする。

**`forEach` を禁止する理由**: `forEach` は**コールバックの戻り値を仕様上まったく見ない**。非同期版の `forEach` は作られなかったので、ループ本体に後から `await` が必要になったとき **`for...of` はそのまま動き、`forEach` は黙って壊れる**（Promise を捨てて即座に完了し、副作用は後から届く）。改修時の取りこぼしが現実的なリスクなので、条件付きにせず一律で禁止する。

`map` / `filter` / Iterator Helpers は**戻り値を保持する**ので、await 忘れが `Promise<T>[]` として型に現れる。危険なのは戻り値を捨てる `forEach` 固有の性質であって、メソッドチェーン一般ではない。

### 5.2 `switch` の網羅性 — `default` を安易に書かない

型で網羅が証明できる場合、素の `default` は網羅検査を無効化するため有害。Java の同じ議論（[JEP 441](https://openjdk.org/jeps/441)）は「match-all clause は pernicious。網羅的な switch は match-all を持たない方がよい」と結論している。

**入力の性質で書き方を変える。**

| 入力 | 書き方 |
|---|---|
| 型で閉じられる（Union・判別子つき） | `default: return assertNever(x);` |
| 型で閉じられ、ランタイム保証が不要 | **`default` を完全に省略**（ケース追加時はコンパイルエラー） |
| 外部由来で型で閉じられない | 素の `default`（`throw` またはフォールバック） |

```typescript
type Mode = 'string' | 'json';

// ✅ 型で閉じられる
export const render = (mode: Mode): string => {
  switch (mode) {
    case 'string':
      return renderString();
    case 'json':
      return renderJson();
    default:
      return assertNever(mode);   // Mode に 'xml' を足すとコンパイルエラー
  }
};

// ✅ 外部由来
export const renderExternal = (raw: unknown): string => {
  switch (raw) {
    case 'string':
      return renderString();
    default:
      throw new Error(`unsupported mode from external input: ${String(raw)}`);
  }
};
```

`assertNever` は `never` を受けて必ず throw する共有ユーティリティとして1つ持つ（`src/shared/assertNever.ts`）。**外部入力に使ってはならない** — 型が閉じていないので網羅を証明できず、単に `throw` するだけの遠回りになる。既定を `assertNever` にする理由は**ランタイム保証も要るため**で、型を跨いだ実データ（`as` を通った値・realm 越え・古いビルドの永続データ）が来たときに throw させる。

> `assertNever` は最も広く import される層に置くので、**ランタイム依存（`node:util` 等）を入れない**。入れるとその層全体が Node 専用になる。

### 5.3 async / await

| 対象 | ルール |
|---|---|
| **非同期関数** | `async` / `await` に統一。`.then()` チェーンは使わない |
| **`Promise` の直接 `return`** | `await` 不要なら省略してよい |
| **並列実行** | 独立した処理は `Promise.all`（部分失敗を許すなら `allSettled`、最初の成功だけ要るなら `Promise.any` + `AggregateError`） |
| **エラーハンドリング** | `try` / `catch` で明示的に。Promise を握りつぶさない |
| **キャンセル・タイムアウト** | 外部 I/O は `AbortSignal` を受け取れる形にする |

```typescript
// ✅ 並列実行
const [users, channels] = await Promise.all([fetchUsers(client), fetchChannels(client)]);

// ✅ タイムアウトと外部キャンセルの合成（どちらも ES 標準）
const call = async (url: string, signal?: AbortSignal): Promise<Response> => {
  const timeout = AbortSignal.timeout(10_000);
  return fetch(url, { signal: signal ? AbortSignal.any([signal, timeout]) : timeout });
};

// ❌ .then() チェーン
transport.fetch(url, options).then(response => { ... });
```

### 5.4 沈黙の失敗を作らない

黙って空を返す・握りつぶす設計にしない。呼び出し側が気づけないものは、例外か、明示的な「空である理由」を返す。

```typescript
// ❌ 失敗を空配列に畳んで隠す
try {
  return await search(query);
} catch {
  return [];
}

// ✅ 呼び出し側が判断できる形にする
return { records: await search(query), truncated: false };
```

打ち切り・上限・サンプリングを行う場合は、**打ち切った事実を返す**（黙って削らない）。

### 5.5 リソース解放は `using` で行う

解放が要るもの（ファイルハンドル・接続・ロック・一時ディレクトリ）は **`using` / `await using` を第一選択**にする。`try` / `finally` は、獲得と解放が離れ、複数リソースでネストが深くなり、解放漏れが型にも lint にも現れない。

```typescript
// ✅ 宣言と解放が同じ行に紐づく
const readAll = async (path: string): Promise<string> => {
  await using handle = await open(path);
  return handle.readFile({ encoding: 'utf8' });
};

// ❌ 解放が離れる。early return を足した人が finally を見落とす
const handle = await open(path);
try {
  return await handle.readFile({ encoding: 'utf8' });
} finally {
  await handle.close();
}
```

- **破棄はスコープ末尾で、宣言と逆順**に走る
- **ループ内では反復ごとに破棄される**。`try` / `finally` を毎周ネストする必要がない
- 解放されるオブジェクトは `[Symbol.dispose]()`（同期）または `[Symbol.asyncDispose]()`（非同期）を実装する。自作リソースには実装を足す
- 個数が動的に決まる場合は `DisposableStack` / `AsyncDisposableStack` にまとめる（`use` / `defer` / `adopt`）

```typescript
class Connection {
  async [Symbol.asyncDispose](): Promise<void> {
    await this.close();
  }
}
```

**破棄中に例外が出たときの扱いは §6.3。** これを知らずに使うと本来の失敗原因を取り落とす。

### 5.6 推奨事項

| 対象 | アクション |
|---|---|
| **`== null`** | null / undefined の一括チェックに積極活用（lint の等価演算子ルールは null を除外して設定する） |
| **`??`** | 推奨（`0` / `false` / `''` が有効値なら `\|\|` ではなく `??`） |
| **`?.`** | 推奨 |
| 分割代入 / デフォルト引数 / スプレッド / アロー関数 / 一時変数の排除 | 推奨 |
| 非破壊の配列操作（`toSorted` / `with`）／分類（`Object.groupBy`・`Map.groupBy`）／Iterator Helpers／集合演算／`RegExp.escape`／`Promise.withResolvers`／`Promise.try`／`Array.fromAsync`／Import Attributes | ES 標準のものは使ってよい |
| `import.meta.dirname` / `import.meta.filename` | `__dirname` / `__filename` の ESM 置き換え |

> **`lib` は実行環境より先を行く。** 型が通っても実行環境に無い API がある（コンパイラの `lib` は TypeScript のバージョンに追随する）。**新しい API を使うときは、実行環境の下限を満たすかを個別に確認し、満たさないものは使わない。** 確認結果は下限マーカー（§8.5）として付録 A に足す。

> **`Object.groupBy` の注意**: 戻り値のキーの存在が型で保証されないので `?? []` で受ける。キー集合が閉じているなら `Map.groupBy`。ここで `!` や `as` に流れると §4.1 と衝突する。

---

## 6. エラー戦略と TSDoc

### 6.1 エラーの投げ分け

| クラス | 用途 |
|---|---|
| **`TypeError`** | **境界の関数**での fail-fast 型バリデーション |
| **`Error`** | ドメインエラー（API 通信の失敗・期待するリソースが無い等） |
| **カスタムエラークラス** | 追加情報を持たせたい場合（§6.4） |

```typescript
if (!instanceUrl) {
  throw new TypeError('instanceUrl には空でない string を指定してください');
}
```

**ランタイムガードは境界に置く**（§1.1）。公開 API・外部入力の受け口・プロセス境界が対象で、**内部の非公開関数には置かない**（型で担保されているものを二重に検査するとノイズになる）。

メッセージの期待型は自然言語で列挙してよい。`null` や複数型を許容する場合は「`... には Date または null を ...`」のように明示する。

### 6.2 例外連鎖 — `cause` で情報を落とさない

```typescript
// src/shared/toError.ts
export function toError(e: unknown): Error {
  return Error.isError(e) ? e : new Error(String(e), { cause: e });
}
```

`catch (e)` の `e` は `unknown`。**`as Error` を散在させず `toError()` で正規化する。** `instanceof Error` ではなく **`Error.isError`** を使う（`instanceof` は realm を跨ぐと誤判定する — `node:vm`・worker・別 realm 由来の Error）。

境界をまたぐときに情報を畳む場合も、`cause` で元を残す。

```typescript
// ✅ 元の例外を捨てない
throw new Error(`gBizINFO の取得に失敗しました: ${response.status}`, { cause: original });

// ❌ message に畳んで元を捨てる
throw new Error(`HTTP ${original.status}: ${original.message}`);
```

> **`cause` が消える経路**: `cause` は非 enumerable なので **JSON 直列化で確実に消える**（`JSON.stringify(err)` → `{}`）。`structuredClone` なら保持される。HTTP / IPC など JSON を経由する境界では、`cause` チェーンを明示的に平坦化してフィールドに詰め直す。

### 6.3 破棄中の例外 — `SuppressedError` の向きに注意する

`using`（§5.5）で本体と破棄の両方が失敗すると、投げられるのは `SuppressedError` になる。**フィールドの向きが直感と逆である。**

| フィールド | 中身 |
|---|---|
| `error` | **破棄時**の例外（後から起きた方） |
| `suppressed` | **本体**の例外（本来の失敗原因） |

つまり素朴に `e.message` を読むと**後始末の失敗を掴み、本来の原因は `suppressed` に埋まる**。§6.2 の `cause` と同じ「情報を落とすな」の問題なので、境界でログ・整形する箇所では両方を辿る。

```typescript
// ✅ 本来の原因まで辿る
if (e instanceof SuppressedError) {
  logger.error('本体の失敗', e.suppressed);
  logger.error('後始末も失敗', e.error);
}
```

### 6.4 カスタムエラークラスの書き方

`name` はクラスフィールド（`override readonly`）で宣言する。**パラメータプロパティは使わない**（§4.8）。

```typescript
export class HttpError extends Error {
  override readonly name = 'HttpError';
  readonly status: number;
  readonly body: unknown;

  constructor(message: string, status: number, body: unknown, options?: ErrorOptions) {
    super(message, options);
    this.status = status;
    this.body = body;
  }
}
```

### 6.5 TSDoc

型は型定義で表現するため `@param` に型注釈は不要。

- `@param name - 説明` — **ダッシュ ` - ` 必須**、型注釈なし
- `@returns` / `@throws`（送出する場合）/ 設計上の制限事項

```typescript
/**
 * Salesforce API クライアントを作成する
 *
 * @param instanceUrl - 組織固有の My Domain URL (例: https://yourorg.my.salesforce.com)
 * @param options - オプション設定
 * @returns Salesforce API クライアント
 * @throws {TypeError} instanceUrl が空文字の場合
 */
export const create = (instanceUrl: string, options: CreateOptions = {}): SalesforceApiClient => { ... };
```

---

## 7. テスト

**ランナーは `node:test`**（`.ts` を直接実行でき、追加依存が要らない）。アサーションは `node:assert/strict`。モックの既定は**依存注入**（§2.7 の decode / parse を引き回す設計と一致）で、`t.mock.fn` / `t.mock.method` はフラグ不要。**モジュール単位のモックは既定では存在しない**（experimental フラグが必要）ので、それに依存する設計にしない。

| 対象 | 方針 |
|---|---|
| **純関数・共有ロジック** | 直接ユニットテストする |
| **境界（I/O・フレームワーク越し）** | 依存をモックしてテストする |
| **検証範囲** | 正常系だけでなく**エラー経路も検証する** |
| **バルク・境界値** | 上限付近・空・1件・大量を検証する |

**テストは `src/` 配下に `*.test.ts` として同居させる。** 別ディレクトリに置くと lint のプロジェクト解決が対象範囲外と判定して構文解析エラーになる（§8.1）。配布物からテストを除くのはビルド用の設定を分けて行う。

### 7.1 テストコードは本体と同じ制約をかけない

テストは「壊れたら気づく」ためのコードで、本体と同じ制約をかけると表現力が落ちる。

| | 本体 | テスト |
|---|---|---|
| `as unknown as` | 理由コメント必須 | **許容**（モック生成） |
| `any` とその周辺（`any` 値の呼び出し・メンバアクセス・引数・戻り値・不要なアサーション） | 禁止 | **一式まとめて許容** |
| `async` だが `await` なし | 検出 | **許容**（`async () => value` のモック） |
| 重複 | 避ける | **許容**（可読性優先） |

**`any` を許容するなら周辺ルールも一式で緩める。** `any` の宣言だけ許して呼び出しやメンバアクセスを禁じると、**宣言できるが使えない**状態になる。

**実行可能性のガードは緩めない。** デコレータと `accessor`（§4.8）はテストでも Node で動かないので、表現力の問題ではない。テスト向けに構文制限ルールを丸ごと無効化すると**この2つのガードも一緒に消える**（lint は緑になり実行時に SyntaxError）。緩めたいものだけを名指しで外す。

> **緩められるのは lint 側の軸だけ。** コンパイラのフラグはファイル単位で緩められないので、未使用変数や到達不能コードの検査はテストにも同じ厳しさでかかる。分けたい場合はテスト用の設定を別に持つしかない（コストと引き換え）。

### 7.2 段階適用

scaffold 段階は最小構成でよい。共有ロジックや実装本体を書く段でこの章の構成へ寄せる。段階の切り替え時期を曖昧にしないため、どちらの段階かを PR に書く。

---

## 8. 機械強制の要求事項

**設定ファイルは本規約が配布しない。** 各リポジトリで、**本規約に完全準拠させることを目的として**作る（§1.2）。以下は「設定が満たすべき要求事項」であり、設定そのものではない。

### 8.1 設定が満たすべきこと

**コンパイラ**

- strict モード（§4.1）
- 消去可能構文だけを許す（§4.8）
- import / export の構文を保持する（§2.3）
- 相対 import の拡張子を出力時に書き換える（§2.2 とビルド配布の両立）
- インデックスアクセスを厳しくする（§4.3。`!` 禁止の強制ではない）
- `switch` のフォールスルーを検出（§5.1）／`override` を必須にする（§6.4）
- **標準ライブラリと型定義の範囲を明示する** — 既定で DOM が混入し、Node プロジェクトなのに `document` / `window` が型を通る。範囲を絞ると `console` も消えるので実行環境の型定義が必須
- **モジュール解決を Node の現行方式にし、`package.json` に `"type": "module"` を置く**（§2.1）
- **対象ファイルの指定は `.ts` / `.mts` / `.cts` の3パターンで書く** — lint の対象範囲と食い違うと、`.mts` を置いた瞬間に lint が落ちるうえコンパイラは最初からそのファイルを見ない
- ビルド出力からテストを除くのは**別の設定ファイル**で行う — 除外指定に入れると lint のプロジェクト解決がテストを見失う

**採用しないもの**: optional プロパティを厳密にするフラグ／宣言ファイルを単独生成可能にするフラグ（理由は §4.9）。規則に対応しない基盤設定は各リポジトリの裁量。

**lint**

- **`.ts` 系・`.js` 系・`.tsx` 系のすべてを対象にする。** 片方にしかルールを置かないと**もう片方が完全な死角になる**（違反だらけのファイルが警告ゼロで通る）。**「どれが共通か」を列挙で持たない** — 列挙は必ず実際のルール集合から遅れ、列挙外が共通でないという誤った含意を作る
- **`.js` 系には実行環境のグローバルを与える** — `console` / `process` が未定義エラーになり、設定ファイル自身が落ちる
- **型情報を使う「厳格」水準のプリセットを有効化する** — 「推奨」水準では `!` を禁じるルールが含まれず、**`any` 禁止だけが効いて片方が静かに死ぬ**（§4.1・§5.3・§7.1）
- **`switch` の網羅性検査と、公開関数の戻り値型を要求するルールを明示的に足す** — 一般的な「厳格」プリセットには**含まれていない**（§4.1・§5.2）
- **デコレータ・`accessor` を構文で禁じる** — コンパイラは素通りするので lint しか止められない（§4.8）
- **相対 import の `.js` を静的・動的の両方で禁じる** — コンパイラは `.js` を `.ts` に写像して**通してしまう**。動的 `import()` は静的 import 用のルールでは捕まらない（§2.2）
- `forEach` / `var` / ブロック省略 / Yoda 条件を禁じる（§5.1）
- **`switch` のフォールスルーを lint 側でも禁じる** — コンパイラの検出は `.js` を見ない
- **等価演算子は厳格に。ただし `null` は除外**（§5.6 の `== null`）
- **未使用の変数・引数・catch 変数のすべてで `_` 接頭辞を除外する** — 除外の指定はそれぞれ別なので、変数だけ設定すると §3 の `(_event, _context) => {}` が弾かれる
- **不要になった抑制コメントを検出する**（§4.7 の許容枠が化石化する）
- **テストランナーの `test()` を「戻り値を捨ててよい呼び出し」として登録する**（Promise の握りつぶし検査が全テストで発火する）
- **プロジェクト解決を使う場合、コンパイラの対象範囲と揃える**（範囲外のファイルが構文解析エラーになる）

**整形**: 一行化（§5.1）は lint では検出できないので、整形ツールが担当する。コード例の書式（シングルクォート）に合わせた設定を置く。

**バージョンの選び方**: 具体的な版を本規約に固定しない（すぐ腐る）。満たすべき条件は次の2つ。

- **型情報を使う lint が動く組み合わせを選ぶ。** 型チェッカーと lint プラグインの対応バージョンがずれると、**lint がロード時に失敗して1件も走らなくなる**（型チェックだけは通るので緑に見える）。**型チェッカーの最新版が lint 側の対応範囲より先に進んでいることは珍しくない。** 速い方を採るか lint を維持するかは、この規約を機械強制できるかどうかで決める（§1.2）
- **Node は型注釈除去が stable な版以降。** LTS の切り替え時期に baseline を見直す

固定した版とその理由は `package.json` の隣（コメント・`.node-version` 等）に書き、**本規約には書かない**。

### 8.2 誰も守っていないもの

設定を入れても**検出されない**。規約文の側で守る。

- `any` / `as` / `!` — コンパイラでは検出されない。lint が必須（§4.1）
- インデックスアクセスを厳しくするフラグは **`!` 禁止を強制しない**（チェックを強制するだけ。§4.3）
- デコレータ / `accessor` — 消去可能構文フラグを**素通りする**（§4.8）
- 相対 import の `.js` — コンパイラは**通す**（§2.2）
- 一行化 — lint では**検出できない**（§5.1）
- 公開関数の戻り値型を要求するルールは、**`export` されるオブジェクト経由で到達可能な関数も対象にする**（§4.1）
- `SuppressedError` の向き（§6.3）・命名規則（§3）・境界の規律（§2.5〜2.8）・許容枠の理由の妥当性（§4.7）— 機械は判定できない

### 8.3 検証義務

**§8.1 の各要求につき、違反コードを1件書き、束ねたコマンド（下記）を実行して落ちることを確かめる。** ツール単体で確認すると、**ツールは正しいのにコマンドに繋がっていない**という穴を見逃す。確認していない要求を「機械が守っている」と扱わない。設定を変えたとき・依存を更新したときも同じ確認をする。

**警告ゼロで通ることは「守られている」の証拠にならない** — ルールを緩めても同じく緑になる。

範囲の切り方:

```
自分のコード         警告ゼロを強制する
借り物・生成物・CJS   lint / typecheck の対象から外す（§1.3）
テストコード         lint 側の軸だけ緩める（§7.1）
```

**抑制コメントは借り物コードには使えない**（同期や再生成で上書きされる）。設定ファイル側で範囲を切る。「警告ゼロ」が崩れる主因は自分が書いていないコードの警告で、逃げ道の無い全面適用は一度崩れると戻せない。範囲を先に決める。

`lint` / `format` / `typecheck` / `test` は1本のコマンドに束ね、**ローカルと CI で同じものを実行する**。AI エージェントに作業させる場合は、完了報告の前にそれを通すことを `AGENTS.md` 側に書く。

### 8.4 本規約に追記するときのゲート

本規約は常時ロードされる。**次の5つを全部満たさないものは書かない**（満たさないなら1行に圧縮するか、書かない）。

1. **再現性** — 本規約が既に与えている規則・例に従って踏んだ問題か（独自に組んだ構成の問題ではないか）
2. **普遍性** — §1〜§8 の主経路を辿る全員が踏むか（特定のオプション経路だけで踏むものではないか）
3. **恒久性** — 規則同士の構造的な相互作用か（今のツールのバージョン状況という時限的事実ではないか）
4. **プローズ充足性** — 一文で防げるか（正しさが手順の細部に依存し、文章では再現できないものではないか）
5. **頻度** — 繰り返し踏むか（一度きりの偶発ではないか）

**「正しいが行動を変えない行」が最大のコストである。** 検証した事実の記録・変更履歴・網羅的な機能列挙・実行すれば分かること・モデルが既に知っていることは書かない。

**実装の詳細は規約に置かない。** 共有ユーティリティの中身・そのエッジケース処理の理由は、そのファイルのコメントに置く。規約が持つのは「いつそれを使うか」まで。

### 8.5 意図的に持たないもの

以下は検討のうえで**持たない**と決めたもの。親切心で戻さないこと。戻すときは、なぜ前回外したのかを先に確認する。

| 持たないもの | 理由 |
|---|---|
| **設定ファイルの原本**（tsconfig / lint / 整形 / package.json の全文） | 本規約と一緒に検証されないので腐る。半年後にコピーした人に古い設定を配る害が、コピペできる利便より大きい。§8.1 の要求と §8.3 の検証義務で代替する |
| **ピン止めしたバージョン番号**（「TypeScript は 5.8 を使う」） | 最も速く腐り、しかも腐ったことに気づく機構が文書の内側に無い。満たすべき条件（§8.1 末尾）に変換し、固定値は `package.json` の隣に置く |
| **変更履歴・移行の経緯** | 常時ロードされる文書が持つ意味がない。git 履歴の仕事 |
| **「どれが機械強制されているか」の網羅表** | 検証手段を持たない文書がこれを掲げると、誤った安心を作る。持つのは §8.2（**誰も守っていないもの**）だけ |
| **「どれが共通ルールか」の列挙**（§8.1） | 列挙は必ず実際のルール集合から遅れ、列挙外が共通でないという誤った含意を作る |
| **本文側での要件の要約** | 参照先が変わると要約だけが古いまま残る。要件の中身は単一の節が持ち、他は参照だけにする |
| **レビュー済み・検証済みの表示** | 読み手の行動を変えない。代わりに冒頭の非保証と §8.3 の検証義務を置く |

> **例外 — 下限マーカーは持ってよい。** 「この API は Node 26 以降」のような**機能の可用性の下限**は、ピン止めと違って腐らない（「Node 26 で入った」は永久に真）。§5.6 で新しい API を採るときは、この形で下限を書き残す（付録 A）。**「Node 26 を使う」は下限マーカーではなくピン止めなので書かない。**

---

## 付録 A. 実行環境の下限と baseline の更新

規則ではなく、規則を運用するための材料。§5.6（新しい API を使う前の確認）と §8.1（LTS 切り替え時に baseline を見直す）が要求する作業の実体をここに置く。

### A.1 下限マーカー

本規約の baseline は「型注釈除去が stable な Node」（§8.1）。**それより後に入った機能は、下限を満たす環境でしか使えない。** 確認済みの下限を記録する。

| 機能 | 下限 |
|---|---|
| `using` / `await using` / `DisposableStack` / `SuppressedError`（§5.5・§6.3） | baseline |
| `Error.isError`（§6.2） / `RegExp.escape` / `Promise.try` / `Array.fromAsync` / `Object.groupBy`（§5.6） | baseline |
| `Uint8Array` の base64 / hex 変換 | **Node 25+** |
| `Temporal` | **Node 26+** |
| `Map.getOrInsert` / `getOrInsertComputed` | **Node 26+** |
| `Iterator.concat` | **Node 26+** |

**この表は網羅ではない。** ここに無い新しい API は、使う前に自分で実行環境を確認し、行を足す。**型が通ったことを可用性の根拠にしない**（§5.6）。

### A.2 baseline を上げるときの手順

1. §1.3 の適用範囲と §8.1 の baseline 条件を更新する
2. A.1 で**下限が満たされた行を消す**（マーカーの役目が終わる）
3. **解禁された機能を自動で採用しない。** 採るなら §8.4 のゲートを通し、規則の形にしてから本文に書く
4. §8.3 の検証をやり直す（依存を更新したときと同じ扱い）

### A.3 次の baseline 更新で決めること

- **`Temporal` を日時の第一選択にするか。** `Date` は可変・月が 0 始まり・タイムゾーンを持たない。`Temporal` は不変で、日付・時刻・タイムゾーンを別の型に分ける。§1 の「Java ライクな堅牢性」に照らすと `java.time` と同じ位置づけにあたるため、**採否を先送りせず A.2 の手順3で判断する**
