# 学習ログ 要件定義

> 何年何月何日に、何を、どれくらい学習したかを記録する個人用 Web アプリ。
> Next.js + Supabase + Vercel で、認証・CRUD・RLS・集計を一通り作る学習用プロジェクト。

## 1. 技術スタック（固定）

- フロント／サーバー：Next.js（App Router）＋ TypeScript
- DB／認証：Supabase（Postgres ＋ Supabase Auth）。Next.js との接続は公式の `@supabase/ssr` を使う
- デプロイ：Vercel
- 認証は Supabase Auth のメール＋パスワード。**自前の users テーブルにパスワードを保存しない**
- アプリから `service_role` キーは使わない（anon キー＋ログイン中ユーザーのセッションだけで動かす。RLS を素通りさせないため）
- タイムゾーンは JST。「今日」「今週」「今月」はすべて JST 基準で判定する
- グラフのライブラリは任意（Recharts など軽いもの）

## 2. 画面一覧

| 画面 | パス | 未ログインで開いたとき |
| --- | --- | --- |
| 新規登録 | `/signup` | 表示する |
| ログイン | `/login` | 表示する |
| 記録の一覧 | `/` | `/login` へリダイレクト |
| 記録の追加 | `/logs/new` | `/login` へリダイレクト |
| 記録の編集 | `/logs/[id]/edit` | `/login` へリダイレクト |

- 詳細画面は作らない（詳細＝編集画面）
- ログイン済みで `/login` `/signup` を開いたら `/` へリダイレクト
- リダイレクトはサーバー側でセッションを確認して行う（クライアントで隠すだけにしない）
- ログイン中の画面には共通ヘッダーにログアウトボタンを置く。押すとセッションが切れて `/login` へ

## 3. データ

### 3-1. `study_logs` テーブル（1 テーブルのみ）

| カラム | 型 | 制約 | 意味 |
| --- | --- | --- | --- |
| id | uuid | PK、default `gen_random_uuid()` | |
| user_id | uuid | NOT NULL、FK `auth.users(id)`、default `auth.uid()` | 登録したユーザー |
| studied_on | date | NOT NULL | 学習した日（JST の日付） |
| minutes | integer | NOT NULL、1〜1440 | 学習時間（分） |
| subject | text | NOT NULL、固定の選択肢のみ（CHECK） | 科目 |
| memo | text | 任意、500 文字まで | メモ |
| created_at | timestamptz | NOT NULL、default `now()` | |
| updated_at | timestamptz | NOT NULL、更新時に自動更新 | |

- 同じ日に複数件登録できる（ユニーク制約なし）
- 開始・終了時刻は持たない。学習時間は「分」だけ
- `user_id` はフォームから送らない。DB の default（`auth.uid()`）で入れる

### 3-2. 科目（固定の選択肢）

- v1 の科目：**【要記入：3〜6 個】**
- 選択肢はコードの定数（例：`lib/subjects.ts`）と DB の CHECK 制約の両方に置き、変えるときは両方直す
- 自由追加は v1 ではやらない

### 3-3. RLS（4 操作すべて）

`study_logs` で RLS を有効化し、`authenticated` ロールに対して次の 4 ポリシーを作る。

| 操作 | 条件 |
| --- | --- |
| SELECT | `auth.uid() = user_id` |
| INSERT | WITH CHECK `auth.uid() = user_id` |
| UPDATE | USING ＆ WITH CHECK `auth.uid() = user_id` |
| DELETE | USING `auth.uid() = user_id` |

「見える」だけでなく「他人のは追加も変更も削除もできない」まで含める。SQL は付録 A。

### 3-4. Supabase Auth の設定

- メール確認（Confirm email）：**オフ**。登録したらそのままログイン状態になる
- パスワード最小文字数：8（アプリ側のバリデーションと合わせる）
- パスワード再設定の画面は v1 では作らない

## 4. 集計ルール

- 今週：今日を含む **月曜〜日曜**（JST）。`studied_on` がこの範囲の記録の `minutes` 合計
- 月別：`studied_on` の年月で `minutes` を合計
- 科目別：当月の記録を `subject` ごとに合計
- 分の表示は「X時間Y分」（90 → 1時間30分、45 → 45分、0 → 0分）
- 「今日」の初期値は JST で作る。`new Date().toISOString().slice(0, 10)` は UTC の日付になるので使わない

## 5. 画面ごとの完了条件

### 5-1. 新規登録（`/signup`）

- メールアドレスとパスワード（8 文字以上）で登録できる
- 成功したらそのままログイン状態になり `/` へ遷移する
- 登録済みのメールで登録しようとしたらエラーを表示し、登録しない
- ログイン画面へのリンクがある

### 5-2. ログイン（`/login`）

- 登録済みのメールとパスワードの組み合わせでログインでき、`/` へ遷移する
- 組み合わせが違う場合はログインできず、「メールアドレスまたはパスワードが違います」と表示する（どちらが違うかは出さない）
- 新規登録画面へのリンクがある

### 5-3. 記録の一覧（`/`）

- 上部に「今週の合計」を表示（4 の定義）
- 月ごとの合計学習時間の棒グラフ（直近 6 ヶ月、記録がない月は 0 で表示）
- 当月の科目別合計（表か横棒グラフ）
- 記録一覧：当月の記録を `studied_on` 降順（同日は `created_at` 降順）で表示。各行に日付・科目・学習時間・メモ（先頭 40 文字）
- 前月／次月に移動できる（デフォルトは当月）
- 各行から編集画面（`/logs/[id]/edit`）へ
- 「追加」ボタンから `/logs/new` へ
- 記録が 0 件のとき「まだ記録がありません」と追加ボタンを出す
- 自分の記録しか出ない

### 5-4. 記録の追加（`/logs/new`）

- 入力項目：日付（初期値は今日・JST、今日より後は不可）、科目（選択肢、必須）、学習時間（分・整数・1〜1440、必須）、メモ（任意・500 文字まで）
- 保存すると `study_logs` に 1 行追加され、`user_id` にログイン中のユーザーが入る
- 保存後は `/` に戻り、追加した記録が一覧に出る
- バリデーションエラーは項目のそばに日本語で出し、保存しない
- キャンセルで `/` に戻る

### 5-5. 記録の編集（`/logs/[id]/edit`）

- 存在しない id、自分の記録でない id は 404 扱い（一覧へ戻る導線を出す）
- 日付・科目・学習時間・メモをすべて変更できる。バリデーションは追加と同じ
- 保存すると `updated_at` が更新され、`/` に戻る
- 「削除」ボタン → 確認ダイアログ → OK で物理削除 → `/` に戻る
- 他人の記録は見えない・変えられない・消せない

## 6. 共通（非機能）

- スマホ対応必須：幅 375px で横スクロールなしに全画面が使える。入力欄とボタンは指で押せる大きさ（高さ 44px 目安）
- 表示は日本語
- 公開 URL（Vercel）で動く。Supabase の URL と anon キーは Vercel の環境変数に置き、リポジトリに入れない
- エラーは画面に日本語で出す（白画面・console だけにしない）

## 7. 完成の基準（受け入れテスト）

アカウント A と B を用意し、**公開 URL** で順に確認する。

1. A で新規登録 → そのままログイン状態で `/` が開く
2. 間違ったパスワードで A にログイン → ログインできず、エラーが出る
3. A で記録を 3 件追加（うち 2 件は同じ日・別科目）→ 一覧に 3 件出て、今週の合計が 3 件の分数の合計になる
4. A で 1 件編集（科目と分数を変える）→ 一覧とグラフに反映される
5. A で 1 件削除（確認ダイアログが出る）→ 一覧から消える
6. A でログアウト → `/` を開くと `/login` にリダイレクトされる
7. B で新規登録 → 一覧は 0 件で、A の記録は見えない
8. B で A の記録の編集 URL（`/logs/[Aのid]/edit`）を直接開く → 404 扱いで、編集も削除もできない
9. Supabase ダッシュボードの Table Editor で、A と B の行がそれぞれの `user_id` で入っている
10. スマホ幅（375px）で 1〜8 がすべてできる

## 8. 作る順番

| 段階 | 作るもの | 通すテスト |
| --- | --- | --- |
| 1 | Supabase プロジェクト、テーブル＋RLS（付録 A）、新規登録・ログイン・ログアウト、追加、一覧（文字だけ）、Vercel にデプロイ | 1, 2, 3（合計は後回し）, 6, 7, 9 |
| 2 | 編集・削除 | 4（一覧の反映）, 5, 8 |
| 3 | 今週の合計、月別グラフ、科目別内訳、前月／次月 | 3 の合計、4 のグラフ |
| 4 | スマホ幅の調整 | 10 |

段階 1 で一度「公開 URL で 2 アカウントが分離している」状態まで持っていく。グラフは最後。

## 9. v1 でやらないこと

- パスワード再設定
- 科目の自由追加（固定の選択肢のみ）
- 詳細画面
- 開始・終了時刻の記録
- エクスポート、目標設定、連続日数、通知などの機能

---

## 付録 A：テーブルと RLS の SQL

Supabase の SQL Editor でそのまま実行する。科目の CHECK は自分の科目に書き換える。

```sql
-- テーブル
create table public.study_logs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users(id) on delete cascade,
  studied_on date not null,
  minutes integer not null check (minutes between 1 and 1440),
  subject text not null check (subject in ('科目A', '科目B', '科目C')), -- 自分の科目に書き換える
  memo text check (memo is null or char_length(memo) <= 500),
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create index study_logs_user_date on public.study_logs (user_id, studied_on desc);

-- updated_at の自動更新
create or replace function public.set_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end $$;

create trigger study_logs_set_updated_at
  before update on public.study_logs
  for each row execute function public.set_updated_at();

-- RLS
alter table public.study_logs enable row level security;

create policy "select own" on public.study_logs
  for select to authenticated using (auth.uid() = user_id);

create policy "insert own" on public.study_logs
  for insert to authenticated with check (auth.uid() = user_id);

create policy "update own" on public.study_logs
  for update to authenticated
  using (auth.uid() = user_id) with check (auth.uid() = user_id);

create policy "delete own" on public.study_logs
  for delete to authenticated using (auth.uid() = user_id);
```