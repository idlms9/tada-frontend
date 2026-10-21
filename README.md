## アーキテクチャ

### システム構成

```mermaid
flowchart LR
  U["ユーザー"] --> FE["フロントエンド<br/>Next.js (Vercel)"]
  FE -->|"REST API / JWT"| BE["バックエンド<br/>Spring Boot (Railway)"]
  FE -->|"日記の1次分析<br/>(サーバールート)"| N8N["n8n (Railway)"]
  BE -->|"ステッカー生成<br/>本文更新時の再分析"| N8N
  BE --> DB[("Supabase<br/>PostgreSQL + pgvector")]
  BE --> ST[("Supabase Storage<br/>ステッカー画像")]
  BE -->|"検索用 embedding"| V["Voyage AI"]
  N8N --> G["Gemini<br/>タイトル・キーワード・人物抽出"]
  N8N --> C["Cloudflare Workers AI<br/>ステッカー画像生成"]

  classDef mine fill:#FFEDD5,stroke:#F97316,stroke-width:2px;
  class FE,BE mine;
```

オレンジの枠が私の担当範囲です(カレンダー・日記の画面、日記・カレンダーAPI)。
画面は1つのページに、モーダルやパネルを重ねる構成です(ページ遷移なし)。

### 日記を更新するときの流れ

```mermaid
sequenceDiagram
  participant FE as フロントエンド
  participant S as DiaryService.updateDiary<br/>(トランザクション外)
  participant N as n8n(AI再分析)
  participant U as DiaryContentUpdater.apply<br/>(@Transactional)
  participant C as 人物認識(Curator)
  participant E as 検索用データ更新

  FE->>S: PUT /api/diaries/{id}
  S->>N: 本文が変わった場合のみ再分析
  N-->>S: 抽出結果
  S->>U: DB更新を委譲(別クラス)
  U->>U: 日記を更新
  U->>C: 人物抽出イベント(同期)
  C-->>U: 失敗したら例外 → 全体ロールバック
  U-->>FE: コミットして応答
  U-)E: コミット後に非同期で実行
```

### ゴミ箱から復元するときの流れ

```mermaid
sequenceDiagram
  participant FE as フロントエンド
  participant D as DiaryService.restoreDiary<br/>(@Transactional)
  participant DB as PostgreSQL
  participant C as 人物認識(Curator)

  FE->>D: POST /api/diaries/{id}/restore
  D->>DB: 復元対象を悲観ロック(FOR UPDATE)
  D->>DB: 同じ日の既存の日記も悲観ロック
  alt 既存の日記があり replace=false
    D-->>FE: 409(確認を促す)
  else 既存なし、または replace=true
    D->>DB: 既存をゴミ箱へ(replace時)→ 対象を復元
    D->>C: 状態変更イベント(同期・集計を再計算)
    D-->>FE: 復元した日記
  end
```

### ディレクトリ構成

```
バックエンド  src/main/java/com/tada/tada/
├── global/     設定・セキュリティ・共通例外・イベント
├── auth/       ログイン・トークン
├── diary/      日記(私の担当)
├── calendar/   カレンダー(私の担当)
├── search/     検索
└── curator/    人物認識

フロントエンド  domains/
├── calendar/ diary/ sticker/ search/ curator/ auth/ guest/
└── shared/     共通部品(api-client, token-store など)
```
