<p align="center">
  <img src="https://raw.githubusercontent.com/idlms9/tada-frontend/main/tada-logo.png" alt="TADA" width="260" />
</p>

<h3 align="center">AI絵日記サービス</h3>

<p align="center"><b>日記を書くと、AIが一日を1枚のステッカーにして残してくれる</b></p>

<p align="center">カレンダー・日記 担当 ｜ チーム開発(5名・約2ヶ月)</p>

<p align="center">
  <img src="https://img.shields.io/badge/JAVA_17-007396?style=for-the-badge&logo=openjdk&logoColor=white" />
  <img src="https://img.shields.io/badge/SPRING_BOOT_3.5-6DB33F?style=for-the-badge&logo=springboot&logoColor=white" />
  <img src="https://img.shields.io/badge/SPRING_DATA_JPA-59666C?style=for-the-badge&logo=spring&logoColor=white" />
  <img src="https://img.shields.io/badge/JUNIT_5-25A162?style=for-the-badge&logo=junit5&logoColor=white" />
  <img src="https://img.shields.io/badge/NEXT.JS-000000?style=for-the-badge&logo=nextdotjs&logoColor=white" />
  <img src="https://img.shields.io/badge/TYPESCRIPT-3178C6?style=for-the-badge&logo=typescript&logoColor=white" />
  <img src="https://img.shields.io/badge/TAILWIND_CSS-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white" />
  <img src="https://img.shields.io/badge/TANSTACK_QUERY-FF4154?style=for-the-badge&logo=reactquery&logoColor=white" />
</p>

<p align="center">
  <a href="https://tada-frontend-seven.vercel.app/"><b>▶ Demo</b></a> ・
  <a href="https://github.com/idlms9/tada-frontend"><b>Frontend</b></a> ・
  <a href="https://github.com/idlms9/tada-was"><b>Backend</b></a>
</p>

## 私の担当

<table>
  <tr>
    <td width="170"><b>🖥️ フロントエンド</b></td>
    <td>
      ・月間カレンダー画面(CSS Grid自作・年月ピッカー)<br>
      ・日記の作成・詳細モーダル<br>
      ・ゴミ箱パネルとドラッグ&amp;ドロップ削除<br>
      ・API連携・エラー処理
    </td>
  </tr>
  <tr>
    <td><b>⚙️ バックエンド</b></td>
    <td>
      ・日記CRUD API(作成・取得・更新・ゴミ箱・復元)<br>
      ・カレンダーAPI(月別のステッカー取得)<br>
      ・30日後の自動削除バッチ、未使用画像の自動清掃<br>
      ・AI再分析の連携(本文更新時)<br>
      ・人物認識(Curator)との同期イベント連携
    </td>
  </tr>
</table>

## 設計で考えたこと

<table>
  <tr>
    <td width="250"><b>🛡️ 中途半端を残さない</b></td>
    <td>日記・ステッカー・人物認識を<br>1つのトランザクションで保存し、<br>失敗したらDB変更を全て取り消す</td>
  </tr>
  <tr>
    <td><b>🔄 安心して使える</b></td>
    <td>1日1件・新規作成は1日5回まで<br>消してもゴミ箱から戻せる<br>戻す日に日記があれば確認して入れ替え</td>
  </tr>
  <tr>
    <td><b>⏱️ DBを占有しない</b></td>
    <td>AIの呼び出しはトランザクションの外で行い、<br>結果を検証してから<br>短いトランザクションで保存する</td>
  </tr>
</table>

## アーキテクチャ

### システム構成

```mermaid
flowchart LR
  U["User"] --> FE["Frontend<br/>Next.js / Vercel"]
  FE -->|"REST API"| BE["Backend<br/>Spring Boot / Railway"]
  FE -->|"analysis"| N8N["n8n"]
  BE -->|"sticker / re-analysis"| N8N
  BE --> DB[("Supabase<br/>PostgreSQL + pgvector")]
  BE --> ST[("Supabase<br/>Storage")]
  BE -->|"embedding"| V["Voyage AI"]
  BE -->|"event"| CU["Curator"]
  N8N --> G["Gemini"]
  N8N --> C["Cloudflare<br/>Workers AI"]

  classDef mine fill:#FFEDD5,stroke:#F97316,stroke-width:2px,color:#3F312E;
  classDef team fill:#F4F4F5,stroke:#A1A1AA,color:#3F312E;
  class FE,BE mine;
  class U,N8N,DB,ST,V,G,C,CU team;
```

<table>
  <tr>
    <td width="150"><b>🟠 オレンジ</b></td>
    <td>私の担当(画面、日記・カレンダーAPI)</td>
  </tr>
  <tr>
    <td><b>⚪ グレー</b></td>
    <td>チームの他のメンバーの担当<br>(AI処理・DB・インフラ・人物認識など)</td>
  </tr>
</table>

私が実装したのは、バックエンドからn8nを呼び出す連携部分と、人物認識へイベントを渡す部分です。
画面は1つのページに、モーダルやパネルを重ねる構成です(ページ遷移なし)。

### 日記を更新するときの流れ

```mermaid
sequenceDiagram
  participant FE as Frontend
  participant S as DiaryService<br/>(no transaction)
  participant N as n8n
  participant U as DiaryContentUpdater<br/>(@Transactional)
  participant C as Curator
  participant E as Search index

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

<table>
  <tr>
    <td width="150"><b>🔗 人物認識</b></td>
    <td>同じトランザクションで実行<br>失敗したら日記の更新も取り消される<br>(対象はDB変更のみ)</td>
  </tr>
  <tr>
    <td><b>✅ 検索用データ</b></td>
    <td>コミット後に実行<br>失敗しても日記の保存には影響しない</td>
  </tr>
</table>

### ゴミ箱から復元するときの流れ

```mermaid
sequenceDiagram
  participant FE as Frontend
  participant D as DiaryService<br/>(@Transactional)
  participant DB as PostgreSQL
  participant C as Curator

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

<table>
  <tr>
    <td width="150"><b>🔁 入れ替え</b></td>
    <td>既存をゴミ箱へ送る処理と復元を<br>1つのトランザクションで実行<br>途中で失敗しても中途半端にならない</td>
  </tr>
  <tr>
    <td><b>🗑️ 完全削除</b></td>
    <td>利用者の操作では行わない<br>30日経過後に自動削除バッチが実行</td>
  </tr>
</table>

## 担当API

| メソッド | パス | 内容 |
|---|---|---|
| POST | /api/diaries | 日記の最終保存 |
| GET | /api/diaries/{id} | 日記の取得 |
| PUT | /api/diaries/{id} | 日記の更新 |
| DELETE | /api/diaries/{id} | ゴミ箱へ移動 |
| POST | /api/diaries/{id}/restore | ゴミ箱から復元 |
| GET | /api/diaries/trash | ゴミ箱の一覧 |
| GET | /api/diaries/can-create | 作成可否の確認 |
| POST | /api/diaries/generate-sticker | ステッカー生成 |
| DELETE | /api/diaries/{id}/permanent | 完全削除(画面なし) |
| GET | /api/calendar | 月別のステッカー取得 |

## 課題と対応

### ① @Transactionalが効かない

<table>
  <tr>
    <td width="90"><b>🔴 問題</b></td>
    <td>本文更新で、AI再分析の待ち時間も<br>DB接続を占有する恐れ</td>
  </tr>
  <tr>
    <td><b>🟡 原因</b></td>
    <td>同じクラス内の呼び出しでは<br><code>@Transactional</code> が効かない(プロキシを通らない)</td>
  </tr>
  <tr>
    <td><b>🔵 対応</b></td>
    <td>DB更新を別クラス(<code>DiaryContentUpdater</code>)に分け、<br>AI呼び出しはトランザクションの外へ</td>
  </tr>
  <tr>
    <td><b>🟢 結果</b></td>
    <td>AI待ちでDBを占有せず、<br>人物認識が失敗したら更新も取り消される</td>
  </tr>
</table>

### ② 復元の同時操作(排他制御)

<table>
  <tr>
    <td width="90"><b>🔴 問題</b></td>
    <td>「確認→復元」の間に別の操作が入ると、<br>制約違反エラーになる恐れ</td>
  </tr>
  <tr>
    <td><b>🟡 原因</b></td>
    <td>確認と復元の間に、別の処理が入る余地があった<br>(レビューで発見)</td>
  </tr>
  <tr>
    <td><b>🔵 対応</b></td>
    <td>復元する日記と、同じ日の既存の日記に<br>悲観ロック(FOR UPDATE)をかけた</td>
  </tr>
  <tr>
    <td><b>🟢 結果</b></td>
    <td>同時に操作しても、エラーにならず<br>日記は1件のまま(DB制約とロックの二重で防御)</td>
  </tr>
</table>

## テスト

| 対象 | 検証内容 | 件数 |
|---|---|---:|
| DiaryService | 作成・更新・ゴミ箱・復元 | 18 |
| DiaryContentUpdater | 本文更新・イベント発行 | 5 |
| StickerCleanupService | 未使用画像の清掃 | 5 |
| StickerObjectCleanupClient | ストレージ連携 | 5 |
| DiaryAnalysisClient | n8n連携 | 3 |

JUnit・Mockitoで作成。実行:`./gradlew test`

## ディレクトリ構成

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
