---
title: "YouCam の image-to-image API で「筋トレを続けた未来の自分」を生成するアプリを作る"
emoji: "💪"
type: "tech"
topics: ["youcam", "nextjs", "supabase", "typescript", "生成ai"]
published: true
---

> 本記事は「YouCam API を活用した実装事例とアイデア」をテーマにした投稿です。美容・ファッション領域が主戦場の YouCam API を、あえて **筋トレ継続のモチベーション維持** に応用します。毎日のボディ写真を1枚記録し、その写真を起点に「このまま続けたら数ヶ月後どうなるか」を **生成AI（image-to-image）で可視化**。到達日が来たら、実際の自分とあのとき描いた未来を並べて **「答え合わせ」** できる Web アプリを、ローカルで動くところまで手を動かしながら作っていきます。

筆者は YouCam API を初めて触りました。本記事は「ドキュメントを読みながら詰まった箇所」も含めて、**手元で再現できる粒度**で残すことを目的にしています。コードはモック動作（API キーなし）から始められるので、キー発行を待たずに最後まで通せます。

![選択した記録と、描いた未来（1ヶ月後）を並べた比較ビュー](/images/youcam-compare.png)
*履歴で記録を選び、生成した未来予測をクリックすると「選択した記録 ↔ 描いた未来」を並べて確認できる*

---

## TL;DR

- **毎日1枚**のボディ写真を記録し、その写真を起点に **継続期間（1ヶ月 / 3ヶ月）** の「未来の自分」を生成する。生成画像は **到達日（起点日 + 期間）のスロット** に保存され、到達日が来たら **実際の自分と並べて答え合わせ** できる。
- 使った API は YouCam の **AI Image Generator（image-to-image）**。`model: "youcam-image-v2"` に **テキストプロンプト** を渡し、筋肉の陰影・カットごと生成する。
- スタックは **Next.js（App Router） + Supabase local（Docker） + YouCam API**。認証は **単一 API キーを `Bearer`** で渡すだけ。
- API は **「File API → タスク作成 → ポーリング」の非同期ジョブ型**。結果 URL は失効するので **自前の Storage に保存** する。
- 設計の肝は、「実際の記録」と「未来予測」を別テーブルに分け、予測を **`(到達日, 期間)` で一意** なスロットに upsert すること。日付は **月末丸め**（1/31 + 1ヶ月 = 2/28）で計算する。
- image-to-image は「同一人物を保て」と言いすぎると体型がほとんど変わらない。**変化を明示的に要求** して初めて筋肉がつく。これがいちばんの学び。

---

## なぜ image-to-image なのか（Body Reshape では失敗した）

筋トレ実践者の一番のモチベーションは「続けた先の身体」です。最初、YouCam の **AI Body Reshape API**（ウエストや腕を補正するボディリシェイプ）で実装しました。が、結果は微妙でした。

理由は明確で、**Body Reshape は画像の輪郭を変形させる処理**（Photoshop の「ゆがみ」に近い）だからです。腕を太くしても輪郭が膨らむだけで、**力こぶの陰影や腹筋のカットといった「筋肉の質感」は生成されません**。細身の人は「細身のまま膨らむ」だけになります。

そこで、**テキストで指示できる生成AI画像編集 = image-to-image** に切り替えました。これなら `muscular, well-defined abs` のように **筋肉そのものを描き起こせます**。代わりに「同一人物に見えるか」「どれくらい変化させるか」という別の課題が出てくるので、そこはプロンプトで作り込みます。

| 方式 | 何をする処理か | 筋肉の質感が出るか |
| ---- | ---- | ---- |
| AI Body Reshape | 輪郭の変形（太さ・くびれの補正） | × 膨らむだけ |
| image-to-image | テキスト指示で画像を再生成 | ○ 陰影・カットごと生成 |

---

## 作るもの

```
[毎日のボディ写真(1日1枚)]
        ↓ その日の写真を起点に
[継続期間: 1ヶ月 / 3ヶ月] を選んで生成
        ↓ image-to-image
[未来の自分] を「到達日(起点日+期間)」のスロットに保存
        ↓ 到達日が来たら
[実際のその日 ↔ あのとき描いた未来] を並べて「答え合わせ」
```

UI は **「今日」**（その日の記録 + 到達した答え合わせ）と **「履歴」**（過去の記録を選んで未来予測の生成・比較）の2タブ構成です。

![今日タブ。上が今日の記録、下が到達日が今日になった予測との答え合わせ](/images/youcam-today.png)
*今日タブ。上が今日の記録、下が「到達日が今日」になった予測との答え合わせ（この例は 5/9 起点の1ヶ月後が 6/9 に到達）*

---

## 技術スタック

| レイヤー | 採用技術 |
| ---- | ---- |
| フロント | Next.js 16（App Router） / React 19 / Tailwind CSS v4 |
| バックエンド | Next.js Route Handler（サーバー側で YouCam を呼ぶ） |
| 認証 / DB / ストレージ | Supabase local（Docker, `supabase start`） |
| 生成AI | YouCam AI Image Generator / image-to-image（`youcam-image-v2`） |
| テスト | Vitest（日付計算・プロンプトのユニットテスト） |
| パッケージ管理 | pnpm |

ポイントは、**生成AI の API キーは絶対にブラウザへ出さず、Next.js のサーバー側（Route Handler）だけで扱う** ことです。フロントは `fetch("/api/...")` を叩くだけにします。

---

## YouCam image-to-image API の全体像

サーバーは `https://yce-api-01.makeupar.com`、認証は API キーを `Authorization: Bearer <KEY>` で渡すだけです。API キーは [API コンソール](https://yce.makeupar.com/api-console/en/api-keys/) で取得します。フローは **3 ステップの非同期ジョブ型** です。

### ① File API（アップロード）

`POST /s2s/v2.0/file/image-to-image/youcam`

ファイルのメタ情報を宣言すると、**`file_id` と presigned アップロード URL** が返ります。

```jsonc
// リクエスト
{ "files": [{ "content_type": "image/jpg", "file_name": "selfie.jpg", "file_size": 50000 }] }

// レスポンス（抜粋）
{ "files": [{
  "file_id": "U8aq...",
  "requests": [{ "url": "https://.../presigned", "headers": { "Content-Type": "image/jpg" }, "method": "PUT" }]
}] }
```

:::message alert
返ってきた `requests[].url` に **自分で PUT して本体を送る** 必要があります。File API を呼ぶだけではアップロードされません。ここは最初に詰まりやすいポイントです。
:::

### ② タスク作成

`POST /s2s/v2.0/task/image-to-image/youcam`

`model` / `prompt` / `negative_prompt` と入力画像（`src_file_ids`）を渡すと `data.task_id` が返ります。

```jsonc
// リクエスト
{
  "model": "youcam-image-v2",
  "prompt": "Edit the person in Image 1 ... clearly more muscular ... same person.",
  "negative_prompt": "different person, face change, distorted face, ...",
  "src_file_ids": ["U8aq..."]
}
// レスポンス
{ "status": 200, "data": { "task_id": "grH0..." } }
```

- `prompt` は最大 800 文字、`negative_prompt` は最大 500 文字（中英対応）。
- `size`（`"1024*1024"` 形式, 512〜2048）や `prompt_extend`（`true` で自動最適化）も指定できます。

### ③ ポーリング

`GET /s2s/v2.0/task/image-to-image/youcam/{task_id}`

`data.task_status` が `running` → `success` / `error` になるまで監視し、成功時の結果画像 URL を取得します。

:::message
結果 URL は一定時間で失効します。そのまま DB に保存しても後で表示できなくなるので、**結果は自前の Supabase Storage にコピー保存** します。
:::

---

## アーキテクチャ

ポイントは **「Next.js のサーバー側がオーケストレーターになる」** ことです。API キーは Route Handler 内だけで扱い、ブラウザには一切出しません。

```
┌─────────────────────────────────────────────┐
│  ブラウザ (Next.js / React)                  │
│  今日タブ(撮影/答え合わせ)・履歴タブ(選択/比較/生成) │
└───────────────┬─────────────────────────────┘
        │ /api/photos (記録)  /api/predict (生成)  /api/timeline (取得)
        ▼
┌─────────────────────────────────────────────┐
│  Next.js Route Handler                        │
│  ── ここでだけ YouCam キーを扱う ──            │
│  期間→プロンプト変換 / File→Task→Poll          │
└──────┬───────────────────────┬──────────────┘
   ▼                       ▼
┌──────────────────┐  ┌────────────────────────┐
│ YouCam image2image │  │ Supabase Local (Docker) │
│ API               │  │ Storage(原本/結果)・DB   │
└──────────────────┘  └────────────────────────┘
```

---

## セットアップ

ここから手を動かします。前提は Node.js / pnpm / Docker（Supabase local 用）/ Supabase CLI が入っていることです。

### 1. プロジェクト作成

```bash
pnpm create next-app@latest you-can   # App Router / TypeScript / Tailwind を選択
cd you-can
pnpm add @supabase/supabase-js @supabase/ssr
pnpm add -D vitest
```

### 2. Supabase local を起動

```bash
supabase init          # supabase/ ディレクトリを作成
supabase start         # Docker でローカルスタックを起動
supabase status        # API URL / anon key / service_role key が表示される
```

`supabase status` の出力にある URL と各キーを、次の `.env.local` に貼ります。

### 3. 環境変数

`.env.local` を作成します。**まずはモード `YOUCAM_USE_MOCK=true`（モック）** で UI と Supabase 連携を通しで確認し、API キーを取得したら `false` に切り替えて実生成へ進む流れです。

```bash
# ----- YouCam API -----
YOUCAM_API_KEY=                      # API コンソールで取得（モックの間は空でよい）
YOUCAM_API_BASE=https://yce-api-01.makeupar.com
YOUCAM_USE_MOCK=true                 # true: YouCam を呼ばずダミー結果を返す / false: 実APIを叩く

# ----- Supabase (local) -----
NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon key>
SUPABASE_SERVICE_ROLE_KEY=<service_role key>   # サーバー専用・絶対に公開しない
```

:::message alert
`SUPABASE_SERVICE_ROLE_KEY` と `YOUCAM_API_KEY` には **`NEXT_PUBLIC_` を付けない**こと。付けるとクライアントバンドルに焼き込まれて漏洩します。
:::

---

## データモデル（記録と予測を分ける）

このアプリの設計の中心は、**「実際の記録」と「未来予測」を別テーブルに持つ** ことです。マイグレーションを `supabase/migrations/` に置きます。

```sql
-- 実際の記録: 1日1枚 (photo_date を UNIQUE にして保証)
create table public.daily_photos (
  id uuid primary key default gen_random_uuid(),
  photo_date date not null unique,
  image_path text not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

-- 未来の自分: (reveal_date, period) ごとに1枚
create table public.predictions (
  id uuid primary key default gen_random_uuid(),
  source_photo_date date not null,        -- 起点となった記録の日付
  period text not null,                   -- 1m / 3m
  reveal_date date not null,              -- 起点日 + 期間 (到達日)
  image_path text,                        -- 生成成功後に入る
  prompt text,
  status text not null default 'running', -- running / success / error
  error text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (reveal_date, period)            -- 到達日 × 期間で一意のスロット
);
```

- **`daily_photos.photo_date` を UNIQUE** にすることで「1日1枚」を DB レベルで保証します。同じ日に撮り直したら upsert で上書きされます。
- **`predictions` は `(reveal_date, period)` で UNIQUE**。同じ到達日でも「1/1 起点の3ヶ月後（→ 4/1）」と「3/1 起点の1ヶ月後（→ 4/1）」は period が違うので、別スロットとして共存できます。同一スロットへの再生成は upsert で上書きします。
- 認証なしの単一ユーザー前提なので RLS は張らず、**DB アクセスはサーバー（service_role）から**だけ行います。

Storage バケットは原本用 `originals`（`{photo_date}.{ext}`）と結果用 `results`（`{reveal_date}_{period}.{ext}`）の2つを作ります。

```sql
insert into storage.buckets (id, name, public)
values ('originals', 'originals', true), ('results', 'results', true)
on conflict (id) do nothing;
```

マイグレーションを当てます。

```bash
supabase db reset   # ローカル DB を初期化してマイグレーションを再適用
```

---

## 到達日の計算 ― 月末丸めをテストで守る

「起点日 + 期間 → 到達日」の計算がこのアプリの肝です。素朴に `Date` の月加算を使うと **`1/31 + 1ヶ月 = 3/3`** になってしまうため、**月末は翌月の月末に丸める**（1/31 → 2/28、うるう年は 2/29）整数演算で実装します。タイムゾーン事故を避けるため `YYYY-MM-DD` 文字列のまま計算するのがコツです。

```ts:src/lib/dates.ts
import type { Period } from "./prompt";

const PERIOD_MONTHS: Record<Period, number> = { "1m": 1, "3m": 3 };

/** year の month(1-based) の日数。Date.UTC を使い決定的に求める。 */
export function daysInMonth(year: number, month: number): number {
  // day=0 は「前月の最終日」= 当月(1-based)の最終日。
  return new Date(Date.UTC(year, month, 0)).getUTCDate();
}

/** iso(YYYY-MM-DD) に months ヶ月を加算。月末は翌月末に丸める。 */
export function addMonths(iso: string, months: number): string {
  const [y, m, d] = iso.split("-").map(Number);
  const total = m - 1 + months;            // 0-based の月インデックスで加算
  const ny = y + Math.floor(total / 12);
  const nm = (total % 12) + 1;             // 1-based に戻す
  const nd = Math.min(d, daysInMonth(ny, nm)); // 月末クランプ
  const mm = String(nm).padStart(2, "0");
  const dd = String(nd).padStart(2, "0");
  return `${ny}-${mm}-${dd}`;
}

export const revealDate = (sourceDate: string, period: Period) =>
  addMonths(sourceDate, PERIOD_MONTHS[period]);
```

ここは壊れると「答え合わせの日」がずれるので、Vitest で守ります。

```ts:src/lib/dates.test.ts
import { expect, test } from "vitest";
import { addMonths, revealDate } from "./dates";

test("月末丸め", () => {
  expect(addMonths("2026-01-31", 1)).toBe("2026-02-28"); // 平年
  expect(addMonths("2024-01-31", 1)).toBe("2024-02-29"); // うるう年
});

test("到達日", () => {
  expect(revealDate("2026-01-01", "3m")).toBe("2026-04-01");
});
```

---

## プロンプト設計 ―「変化を明示する」のがいちばんの学び

期間を image-to-image の編集プロンプトに変換します。コツは2つで、**①体型の変化を明示的に要求する**、**②顔・体勢・背景など同一性は保つ** です。

```ts:src/lib/prompt.ts
export const PERIODS = [
  { value: "1m", label: "1ヶ月", en: "1 month",
    phrase: "a subtle but already visible improvement in muscle tone and firmness" },
  { value: "3m", label: "3ヶ月", en: "3 months",
    phrase: "a clearly more toned body with a visible increase in lean muscle" },
] as const;

export type Period = (typeof PERIODS)[number]["value"];
export const isPeriod = (v: string): v is Period => PERIODS.some((p) => p.value === v);

export function buildPrompt(period: Period): string {
  const term = PERIODS.find((p) => p.value === period)!;
  const prompt = [
    `Edit the person in Image 1 to realistically show how their body would look after ${term.en} of consistent, intense full-body strength training and a clean diet.`,
    `Clearly and visibly transform the physique to show ${term.phrase}. The increase in muscle size and definition and the change in body composition MUST be clearly noticeable compared to the original photo — do not keep the body almost unchanged.`,
    `Keep the same face, identity, hair, body pose, framing, background and lighting. Photorealistic photo of the same real person; only the muscularity, muscle definition and body fat change.`,
  ].join(" ");
  return prompt.length > 800 ? prompt.slice(0, 800) : prompt; // prompt は最大800文字
}

export function buildNegativePrompt(): string {
  return [
    "different person, face change, distorted face",
    "low quality, blurry, disfigured, bad anatomy, extra limbs, extra fingers",
  ].join(", ");
}
```

:::message
最初は「同一人物・同じ構図を厳密に保て」という制約を強くしすぎて、**実 API なのに体型がほとんど変わりませんでした**。`MUST be clearly noticeable ... do not keep the body almost unchanged` のように **変化を明示的に命じる** 一文を足して初めて筋肉がつきました。同一性（顔・体勢・背景）は別文で守ります。
:::

なお当初は 1ヶ月 / 3ヶ月 / 6ヶ月 / 1年 の4段階でした。しかし **6ヶ月・1年は変化が誇張・不自然になりがち** で「もし続けたら」の説得力が落ちたため、最終的に **1ヶ月・3ヶ月の2段階に絞り込みました**。期間ごとに `phrase` をマッピングして変化量を制御しています。

---

## YouCam クライアント ― モックと実 API を切り替える

API キーが無い段階でも UI と Supabase 連携を完成させられるよう、呼び出しを **インターフェースで抽象化** して環境変数で切り替えます。

```ts:src/lib/youcam/types.ts
export type SimulateInput = {
  image: Buffer;            // 原本画像のバイト列
  contentType: string;      // image/jpeg | image/png
  prompt: string;           // 編集プロンプト
  negativePrompt?: string;
};
export type SimulateResult = {
  resultImage: Buffer;
  resultContentType: string;
  youcamTaskId: string | null;
};
export interface YouCamClient {
  runImageEdit(input: SimulateInput): Promise<SimulateResult>;
}
```

```ts:src/lib/youcam/index.ts
import { mockClient } from "./mock";
import { realClient } from "./real";
import type { YouCamClient } from "./types";

// 明示的に "false" のときだけ実API。未設定・"true" は安全側でモック。
const useMock = process.env.YOUCAM_USE_MOCK !== "false";
export const youcam: YouCamClient = useMock ? mockClient : realClient;
export const isMock = useMock;
export * from "./types";
```

モックは **原本をそのまま結果として返す** だけです（UI に「モック表示」バッジを出す）。なので「モックなのに変わらない」のは正常動作で、実 API に切り替えて初めて生成が走ります。

```ts:src/lib/youcam/mock.ts
export const mockClient: YouCamClient = {
  async runImageEdit(input) {
    await new Promise((r) => setTimeout(r, 1200)); // 生成の待ち時間を模す
    return { resultImage: input.image, resultContentType: input.contentType, youcamTaskId: null };
  },
};
```

実 API の中身が、前述の3ステップです。`fetch` の `body` に Node の `Buffer` を直接渡すと TypeScript の `BodyInit` 型に合わないため、`new Uint8Array(buffer)` に変換しています。また、**生成系 API は「ドキュメントの構造」と「実レスポンス」がズレることがある** ので、結果 URL やファイル情報は **複数の場所を探索しつつ、想定外なら実 JSON をそのままエラーに出す** ようにしておくと、デバッグが一気に楽になります。

```ts:src/lib/youcam/real.ts
const API_BASE = process.env.YOUCAM_API_BASE ?? "https://yce-api-01.makeupar.com";
const authHeader = () => `Bearer ${process.env.YOUCAM_API_KEY}`;

// ① File API: file_id 取得 + presigned URL へ本体を PUT
async function uploadFile(input: SimulateInput): Promise<string> {
  const initRes = await fetch(`${API_BASE}/s2s/v2.0/file/image-to-image/youcam`, {
    method: "POST",
    headers: { Authorization: authHeader(), "Content-Type": "application/json" },
    body: JSON.stringify({
      files: [{ content_type: "image/jpg", file_name: "selfie.jpg", file_size: input.image.byteLength }],
    }),
  });
  const initData = await initRes.json();
  // レスポンスのラップ違いに備えて複数の場所を見る
  const files = initData.files ?? initData.data?.files ?? initData.result?.files;
  const fileInfo = files?.[0];
  const req = fileInfo?.requests?.[0];
  if (!fileInfo?.file_id || !req?.url) {
    throw new Error(`File APIのレスポンス構造が想定外です: ${JSON.stringify(initData)}`);
  }
  // 取得した presigned URL に本体をアップロード（これをしないとアップロードされない）
  await fetch(req.url, { method: req.method ?? "PUT", headers: req.headers, body: new Uint8Array(input.image) });
  return fileInfo.file_id;
}

// ② タスク作成 → task_id
async function createTask(fileId: string, input: SimulateInput): Promise<string> {
  const res = await fetch(`${API_BASE}/s2s/v2.0/task/image-to-image/youcam`, {
    method: "POST",
    headers: { Authorization: authHeader(), "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "youcam-image-v2",
      prompt: input.prompt,
      negative_prompt: input.negativePrompt ?? "",
      src_file_ids: [fileId],
    }),
  });
  const data = await res.json();
  if (!data.data?.task_id) throw new Error(`task_id を取得できませんでした: ${JSON.stringify(data)}`);
  return data.data.task_id;
}

// ③ ポーリング → 結果画像URL
async function pollTask(taskId: string): Promise<string> {
  const startedAt = Date.now();
  while (Date.now() - startedAt < 300000) { // 生成AIは時間がかかるため長めに
    const res = await fetch(`${API_BASE}/s2s/v2.0/task/image-to-image/youcam/${taskId}`, {
      headers: { Authorization: authHeader() },
    });
    const d = (await res.json()).data ?? {};
    if (d.task_status === "success") {
      // 結果URLの場所も揺れるので候補を探索する
      const url = d.url ?? d.results?.url ?? (Array.isArray(d.results) ? d.results[0]?.url : undefined);
      if (!url) throw new Error(`結果画像URLが見つかりません: ${JSON.stringify(d)}`);
      return url;
    }
    if (d.task_status === "error") throw new Error(`${d.error} ${d.error_message}`);
    await new Promise((r) => setTimeout(r, 3000));
  }
  throw new Error("YouCam タスクがタイムアウトしました。");
}

export const realClient: YouCamClient = {
  async runImageEdit(input) {
    const fileId = await uploadFile(input);
    const taskId = await createTask(fileId, input);
    const resultUrl = await pollTask(taskId);
    // 結果URLは失効する。自前のStorageへ保存するため取得しておく。
    const res = await fetch(resultUrl);
    const buf = Buffer.from(await res.arrayBuffer());
    return { resultImage: buf, resultContentType: res.headers.get("content-type") ?? "image/jpeg", youcamTaskId: taskId };
  },
};
```

---

## API ルート

サーバー側でキーを扱うので、3 本の Route Handler に役割を分けます。`Buffer` と Supabase を使うため、いずれも `export const runtime = "nodejs"` を付けます。

### /api/photos ― 実写真の保存（1日1枚 upsert）

画像と日付（省略時は今日）を受け取り、Storage と `daily_photos` を **`photo_date` で upsert** します。同じ日に撮り直すと自然に上書きされます。

```ts:src/app/api/photos/route.ts
export const runtime = "nodejs";

export async function POST(req: NextRequest) {
  const form = await req.formData();
  const file = form.get("image");
  const date = String(form.get("date") ?? "") || todayISO();
  // ...バリデーション（File 型 / 日付形式 / MIME は jpg|png）...

  const buf = Buffer.from(await (file as File).arrayBuffer());
  const supabase = createServiceClient();
  const imagePath = `${date}.${EXT[contentType]}`;

  await supabase.storage.from("originals").upload(imagePath, buf, { contentType, upsert: true });
  const row = await supabase
    .from("daily_photos")
    .upsert({ photo_date: date, image_path: imagePath }, { onConflict: "photo_date" })
    .select().single();

  const base = supabase.storage.from("originals").getPublicUrl(imagePath).data.publicUrl;
  // 上書き時のブラウザキャッシュ回避にバージョンを付ける
  const imageUrl = `${base}?v=${new Date(row.data.updated_at).getTime()}`;
  return NextResponse.json({ date, imageUrl, updatedAt: row.data.updated_at });
}
```

### /api/predict ― 未来の自分を生成（スロットへ upsert）

起点写真の日付（`sourceDate`）と **複数の期間** を受け取り、起点写真を元に各期間を生成して **`(reveal_date, period)` のスロット** へ保存します。先に `running` でスロットを確保し、**一部の期間が失敗しても他は止めない** よう期間ごとに `try/catch` します。過去日を `sourceDate` に渡せば「もしも続けていたら」も同じ経路で作れます。

```ts:src/app/api/predict/route.ts
export async function POST(req: NextRequest) {
  const { sourceDate, periods } = await req.json();
  // ...バリデーション（日付形式 / periods.filter(isPeriod) が1つ以上）...

  const supabase = createServiceClient();
  const photo = await supabase.from("daily_photos")
    .select("image_path").eq("photo_date", sourceDate).maybeSingle();
  if (!photo.data) return NextResponse.json({ error: "その日の写真がありません" }, { status: 422 });

  const dl = await supabase.storage.from("originals").download(photo.data.image_path);
  const srcBuf = Buffer.from(await dl.data!.arrayBuffer());

  const results = [];
  for (const period of validPeriods) {
    const reveal = revealDate(sourceDate, period);
    const prompt = buildPrompt(period);
    // 先に running でスロットを確保（衝突時は上書き）
    await supabase.from("predictions").upsert(
      { source_photo_date: sourceDate, period, reveal_date: reveal, prompt, status: "running", image_path: null, error: null },
      { onConflict: "reveal_date,period" },
    );
    try {
      const out = await youcam.runImageEdit({ image: srcBuf, contentType, prompt, negativePrompt: buildNegativePrompt() });
      const resultPath = `${reveal}_${period}.${EXT[out.resultContentType] ?? "jpg"}`;
      await supabase.storage.from("results").upload(resultPath, out.resultImage, { contentType: out.resultContentType, upsert: true });
      await supabase.from("predictions").update({ status: "success", image_path: resultPath }).eq("reveal_date", reveal).eq("period", period);
      results.push({ revealDate: reveal, period, status: "success" /* imageUrl は ?v= 付きで返す */ });
    } catch (e) {
      await supabase.from("predictions").update({ status: "error", error: String(e) }).eq("reveal_date", reveal).eq("period", period);
      results.push({ revealDate: reveal, period, imageUrl: null, status: "error", error: String(e) });
    }
  }
  return NextResponse.json({ results, isMock });
}
```

### /api/timeline ― 画面描画データ

`daily_photos`（撮影日の昇順）と `predictions`（到達日順）をまとめて返すだけの GET です。今日タブ・履歴タブの両方がこれ 1 本で描画します。結果画像 URL には **キャッシュ回避の `?v=` を付与** します（再生成で同じパスを上書きするため）。

---

## UI（今日タブ / 履歴タブ）

サーバー側がキーを扱うので、フロントは `fetch("/api/...")` するだけです。`page.tsx` で `/api/timeline` を取得し、`isMock` が真ならバッジを出して、今日タブ・履歴タブへ `data` を渡します。

- **今日タブ**：その日の写真を **撮影（アップロード）** し、**到達日が今日の予測があれば「実際の今日 ↔ 描いた未来」を並べて答え合わせ**。
- **履歴タブ**：過去の記録を **選択** すると、その記録を起点とする **未来予測の一覧** と **生成フォーム** が出る。予測をクリックすると **「選択した記録 ↔ その予測」を並べて比較** できる。

![履歴タブ。記録は撮影日の昇順に並び、カードを選ぶと下に詳細が出る](/images/youcam-history.png)
*履歴タブ。記録は撮影日の昇順に並ぶ*

![記録を選択した状態。未来予測の一覧と、1ヶ月／3ヶ月を選んで生成するフォーム](/images/youcam-history-detail.png)
*記録を選択した状態。その記録の未来予測（クリックで比較）と、1ヶ月／3ヶ月を選んで生成するフォーム*

---

## 動かす

```bash
supabase start          # Docker でローカルスタック起動
supabase status         # URL / anon key / service_role key を .env.local へ
pnpm dev                # http://localhost:3000
pnpm test               # 日付計算・プロンプトのユニットテスト
```

まず `YOUCAM_USE_MOCK=true`（モック）で UI と Supabase 連携を通しで確認し、API キー取得後に `YOUCAM_USE_MOCK=false` にして実生成へ切り替えます。モックの間は「撮影 → 生成 → 答え合わせ」の流れは動きますが、生成結果は原本のままです（バッジで明示）。

---

## ハマりどころまとめ

実際に詰まった箇所です。**生成系 API は「ドキュメントの構造」と「実レスポンス」がズレることがある** ので、想定外のときに **実レスポンスをそのままエラーに出す** と一気に解決します。

| # | 症状 | 対処 |
| ---- | ---- | ---- |
| 1 | Body Reshape で筋肉がつかない | あれは輪郭変形ツール。筋肉の質感を出すなら image-to-image に切替 |
| 2 | `File API のレスポンス構造が想定外です` | `files` 直下か `data.files` 配下か揺れる。複数候補を見る + 実 JSON をエラーに出す |
| 3 | `結果画像 URL が見つかりません` | 結果 URL が `data.url` とは限らない。`data.results.url` 等も探索 + 実 JSON 出力 |
| 4 | 入力画像のフィールド名が不明 | image-to-image は複数画像参照。`src_file_ids`（配列）で渡す |
| 5 | 実 API なのに体型が変わらない | 「同一人物を保て」が強すぎ。変化を明示的に命じる一文を追加 |
| 6 | 再生成したのに古い画像が表示される | Storage は同じパスを upsert 上書きする。公開 URL に `?v={updated_at}` を付けてキャッシュ回避 |
| 7 | `1/31 + 1ヶ月` が `3/3` になる | `Date` の素朴な月加算をやめ、月末は翌月末にクランプする整数演算 + テスト |
| 8 | File を呼んだのにアップロードされない | 返却 `url` に自分で PUT する |
| 9 | 結果がいつの間にか消える | 結果 URL は失効する。Storage に保存する |
| 10 | キー露出 | `service_role` と YouCam キーはサーバー側のみ（`NEXT_PUBLIC_` を付けない） |
| 11 | `fetch` の body 型エラー | Node の `Buffer` を直接渡さず `new Uint8Array(buffer)` に変換 |

---

## 横展開：他のアイデア

同じ「YouCam をサーバー側でオーケストレーションし、結果を自前ストレージに保存する」骨格は、そのまま他の API でも流用できます。

- **肌分析 × 運動記録**：トレ前後の自撮りを肌分析 API でスコア化し、運動の効果を可視化する
- **顔分析 × コンディション判定**：朝の自撮りから、その日のトレ強度をレコメンドする
- **バーチャル試着 × フィットネスウェア**：理想体型の生成画像にウェアを試着させる
- **生成AI動画**：毎日の記録から Before → After のモーフィング動画を作る

---

## まとめ

- 「筋肉をつけた未来」を出すなら、輪郭変形の **Body Reshape ではなく、テキスト駆動の image-to-image** が正解。
- YouCam の生成 API は **単一 API キー認証の非同期ジョブ型（File → Task → Poll）**。結果は失効するので **永続化を自前で** 行う。
- 「記録」と「予測」を分け、予測は **`(到達日, 期間)` のスロットに upsert**。日付は **月末丸め** でテストごと守る。
- 生成系は **サーバー側でオーケストレーション** し、キーをクライアントに出さない。
- 品質は **プロンプト設計が9割**。「変化を明示」しつつ「同一人物性」を守り、期間の刻みでリアリティを調整する。
- **モック → 実 API の2段構え** にしておくと、キー発行を待たずに UI と DB 連携を完成させられる。

---

## 参考

- [YouCam API コンソール / API キー発行](https://yce.makeupar.com/api-console/en/api-keys/)
- [YouCam AI API 機能一覧](https://yce.perfectcorp.com/ai-api)
- [Perfect Corp 開発ドキュメント](https://docs.perfectcorp.com/develop/introduction)
