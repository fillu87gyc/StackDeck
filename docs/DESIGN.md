# StackDeck 設計ドキュメント

StackDeck は、複数の AI エージェント（Claude Code, Sonnet, Opus 等）が並行して開発する「スタック」（共通のベースブランチ上に連なる関連 PR のチェーン）を、Kanban 形式のターミナル UI（TUI）としてダッシュボード表示するツールである。左パネルにスタック一覧とそのトポロジー、中央にパイプライン状態（`CI All Green` → `Review Bot Done` → `Human Review` → `Merged`）の列、右パネルに選択中 PR の詳細（依存関係・チェック結果・レビュー依頼状況・操作ショートカット）を表示する。デプロイ（リリース）は独立した列ではなく、Merge 済みかつリリースタグが付いた PR カード上に 🚀 アイコン + タグとして表す。モックはこの体験の視覚的リファレンスであり、本ドキュメントの UI 仕様（§7）はモックの構成要素に開発上の名前を与えるためのものである。

将来的に同じコアを **Web サーバーとしても提供し**、ブラウザから同一体験を得られるようにすることが必須要件であり、本ドキュメントの技術選定・アーキテクチャはこの要件を起点に決定している。

---

## 目的と非目的

**StackDeck が目指すもの**

- 複数 AI エージェントが並行して積み上げる PR スタックの「今どこまで進んでいるか」を一画面で把握できるダッシュボード
- CI / レビューボット / 人間レビューという一連のパイプラインを Merge まで可視化し、Merge 後のリリース（Deploy）はカード上のマーカーとして扱う
- TUI をファーストクラスの体験としつつ、同一コアをブラウザからも利用できるようにする（day 1 設計）
- 読み取り中心（observe）のツールとして、GitHub 上の実状態を正確に反映する

**StackDeck が目指さないもの（非目的）**

- GitHub の PR UI そのものの置き換え（コードレビューのコメント作成・diff 表示などは GitHub / エディタに委ねる）
- v1 において AI エージェントを「起動・制御」するオーケストレーションツールになること（v1 は観測に徹する。§4, ロードマップ Phase 4 参照）
- 汎用 Git クライアントやマージツールになること（Merge/Deploy 操作は既存の CI/CD・GitHub API を薄く呼び出すのみ）
- マルチテナント SaaS になること（v1〜v4 はローカル実行・単一リポジトリ・単一ユーザーを前提とし、複数リポジトリ／複数ユーザー対応は Phase 6 で扱う）

---

## アーキテクチャ概要

TUI と将来の Web フロントエンドが同じ「読み取りモデル」を共有できるよう、GitHub 同期処理とドメインロジックをプレゼンテーション層から分離したレイヤード構成にする。

```
┌─────────────────────────────────────────────────────────────────┐
│                         GitHub API (REST / GraphQL v4)           │
│                         + Webhooks (pull_request, check_suite …) │
└───────────────────────────────┬───────────────────────────────────┘
                                 │ fetch / webhook events
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  Sync Worker                                                     │
│   - GraphQL による差分取得、REST による補完                       │
│   - Webhook 受信 + ポーリングのハイブリッド更新                    │
│   - スタック検出（base ブランチ連鎖 + Depends-on 規約）            │
└───────────────────────────────┬───────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  Core Domain（Stacks / PullRequests / Checks / Reviews / Agents） │
│   - ドメインモデルと整合性ルール（DAG 構築、状態遷移）              │
└───────────────────────────────┬───────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  Read Model（SQLite キャッシュ）                                  │
│   - 「キャッシュは使い捨て」= 削除しても Sync Worker が再構築可能   │
└───────────────────────────────┬───────────────────────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  API Layer（pydantic スキーマで契約化、localhost:8765）           │
└───────────┬─────────────────────────────────────┬─────────────────┘
            ▼                                     ▼
      ┌───────────┐                         ┌───────────────┐
      │    TUI     │                        │   Web（将来）   │
      └───────────┘                         └───────────────┘
```

**共有の要（シーム）**: TUI・Web のどちらも「API Layer」だけを参照し、Core Domain / Sync Worker には直接触れない。v1 では TUI プロセス内で API Layer をインプロセス呼び出しするが、スキーマは pydantic モデルとして契約化しておくため、後日 API Layer を別プロセス・別ホストへ切り出しても呼び出し側のコードはほぼ変更不要になる。これが §5 の移行戦略の技術的根拠である。

---

## 技術選定

### 比較表

| 観点 | Python + Textual | Rust + Ratatui | Go + Bubble Tea | TypeScript + Ink |
|---|---|---|---|---|
| モックの視覚様式への忠実度 | 高（CSS 風スタイリングで角丸ボーダー・アクセントカラーを宣言的に記述） | 中〜高（`BorderType::Rounded` 等はあるが色・レイアウトは手組み） | 高（Lip Gloss が角丸ボーダー・グラデーション配色に強く、Charm 製品群の見た目の完成度が高い） | 中（yoga による Flexbox はあるが、装飾系の定番ライブラリは薄い） |
| 将来の Web 化のしやすさ | **最高**：`textual-serve`（自社謹製・MIT）で数行の変更なく既存 TUI をブラウザ配信可能。`textual-web` は公開 URL 配信までカバーするが beta（セッション非永続、レイテンシ課題あり） | 低：公式の Web 配信手段なし。WASM バックエンド（`ratatui-wasm-backend`, `egui_ratatui` 等）はコミュニティ実験段階 | 中：`wish` で SSH 配信は一級品だが、ブラウザ配信には ttyd/xterm.js 等の外部ブリッジが別途必要 | 低：Ink 自体はターミナル専用。DOM 版に流用できる設計ではない |
| シングルバイナリ配布 | 不可（Python ランタイム前提。`uvx`/`pipx` で擬似的に一発起動は可能） | 最高（`cargo build --release` で完結） | 最高（`go build` で完結、クロスコンパイルも容易） | 不可（Node ランタイム前提） |
| GitHub API との相性 | 良好（`httpx` + `gql` で GraphQL、REST は素の HTTP クライアントで容易） | 中（`octokit-rs`/`octocrab` はあるが機能追随がやや遅れる） | 良好（`google/go-github` が高品質で型安全、広く使われる） | 最高（`octokit.js` は GitHub 公式リファレンス実装） |
| Kanban 風レイアウトのエコシステム | 良好（`DataTable`/`Grid` レイアウトで多列 UI を組みやすい） | 中（列レイアウトの実例はあるが自前実装が多い） | **最高**：Charm 公式チュートリアル "kancli" がまさに Bubble Tea + Bubbles でのカンバンボード構築例 | 中（`ink-select-input` 等はあるが多列ダッシュボードの定番例が薄い） |
| 個人〜小規模チームでの保守負荷 | 低（型ヒント + pytest + 広いエコシステムで十分回る） | 中（所有権・非同期まわりの学習コストがやや高い） | 低（言語がシンプルで並行処理も書きやすい） | 中（React の心的モデルは持ち込めるが、CLI ツールとしての周辺整備＝引数解析等は自前が必要） |

### 決定

**採用: Python + Textual**

理由: StackDeck の必須要件は「同じコアを later で Web サーバーとして配信する」ことであり、これを最小の実装コストで満たせるのは `textual-serve` を持つ Textual だけである。他候補（Ratatui, Bubble Tea）はいずれもブラウザ配信を自前でブリッジする必要があり、Web 化のコストがアーキテクチャ全体の見積もりを支配してしまう。加えて StackDeck はチーム（人間レビュアー＋複数の可視化対象エージェント）に見せるダッシュボードであり、"URL を共有すればインストール不要で見られる" という配布モデルの価値が、単一バイナリ配布の価値を上回ると判断した。視覚様式（角丸ボーダー・列ごとのアクセントカラー）も Textual の CSS ライクなスタイルシートで十分再現できる。

**次点: Go + Bubble Tea + Lip Gloss（+ wish）**

Charm 公式のカンバンボード構築チュートリアル "kancli" が本プロジェクトのユースケースに驚くほど近く、Lip Gloss の配色・角丸ボーダー表現力もモックの美観に最も近い。またシングルバイナリ配布は開発者向け CLI ツールとして訴求力が高い。**もし StackDeck が「チームに配るダッシュボード」ではなく「個人が使う高速な CLI」として再定義されるなら、配布容易性とパフォーマンスを優先して Go + Bubble Tea を選び直すのが正解になる。** その場合の Web 化は `wish`（SSH）+ ttyd/xterm.js ブリッジ、または Core Domain を切り出した完全な shared-core 構成（§5 の Path A）を最初から採用する。

### 技術選定（コンポーネント別）

| コンポーネント | 採用 | 次点 | 次点が有利になる条件 |
|---|---|---|---|
| TUI フレームワーク | Textual | Bubble Tea | Web 配信より単一バイナリ配布と描画性能を優先する場合 |
| 内部 API 層 | FastAPI（pydantic スキーマで契約化、v1 はインプロセス呼び出し） | HTTP を挟まない素の Python モジュール境界 | Phase 0-1 のスピードを最優先し、shared-core 化を当面考えない場合 |
| DB | SQLite + SQLModel（SQLAlchemy + pydantic） | 素の `sqlite3` + 手書きリポジトリ層 | 依存を極小化し、マイグレーションツールの学習コストを避けたい場合 |
| GitHub クライアント | `httpx`（async）+ `gql`（GraphQL v4） | `PyGithub`（REST 専用、同期） | GraphQL のクエリ設計コストを払いたくない、初速優先の個人開発 |
| テスト | pytest + Textual 公式スナップショットテスト（`App.run_test()`）+ `respx` で GitHub HTTP モック | 標準 `unittest` + 手動モック | 依存追加を避けたい最小構成の場合 |
| パッケージング | `uv` + `pyproject.toml`（`uvx stackdeck` で実行） | `PyInstaller` によるシングルバイナリ化 | 「Python 不要」を配布上の必須要件にする場合（Phase 6 でのストレッチゴールとして再検討） |

---

## Web モード戦略

**主軸: Path B（TUI 自体を Web 配信する）＝ `textual-serve`**

- v1〜Phase 5 の主軸として、Textual アプリをそのまま `textual-serve` でブラウザ配信する。ユーザー体験は「ターミナルをそのままブラウザで見る」形であり、モックが想定する monospace な TUI の質感をそのまま維持できる。
- 実装コストが極めて小さい（アプリ本体を変更せず、配信プロセスを追加するだけ）ため、Phase 5 の立ち上げが速い。
- 既知の制約: `textual-serve` はローカル/社内配信向け（サブプロセス起動 + WebSocket 通信）であり、公開 URL での配信や真のマルチユーザー分離は `textual-web`（beta、セッション非永続）の範囲になる。モバイル最適化やレスポンシブなレイアウトは持たない。

**フォールバック: Path A（Core を切り出し、TUI と SPA を薄いクライアントにする）**

- API Layer（§3）がすでに pydantic スキーマで契約化されているため、Path A への移行は「新しい HTTP/WebSocket クライアント（React 等の SPA）を追加するだけ」で済む。Core Domain・Read Model・Sync Worker には手を入れない。
- Path A を選ぶべきタイミング: (1) 非技術者ステークホルダー向けにモバイル対応・レスポンシブなレイアウトが必要になったとき、(2) `textual-serve` のセッション/レイテンシ制約が実運用で問題になったとき、(3) マルチユーザー・マルチリポジトリ対応（Phase 6）で認証・パーミッションを Web 側に持たせる必要が出たとき。

**移行の道筋**: API Layer を最初から「HTTP で呼べる形」の契約として設計しておく（v1 ではインプロセス関数呼び出しでも、スキーマとエンドポイント境界は HTTP API として定義しておく）ことで、Path B → Path A への切り替えは「配信方式の追加」であり「作り直し」にならない。

---

## GitHub 連携

- **読み取り: GraphQL v4 を主軸、REST を補完**。PR・チェック・レビュー・依存 PR を 1 クエリでまとめて取得できるため REST の N+1 を避けられ、GraphQL のポイント制レート制限（5,000 pt/h、REST は 5,000 req/h）の下でも効率が良い。Merge 実行・Deployment ステータス更新など GraphQL がカバーしない操作は REST を使う。
- **鮮度: Webhook 受信を主軸 + ポーリングを安全網として併用**。`pull_request`, `check_suite`, `pull_request_review`, `deployment_status` などのイベントを Webhook で即時反映し、配信漏れに備えて 5〜15 分間隔の低頻度ポーリングで整合性を取る（ハイブリッド方式）。ローカル開発では Webhook 受信に `smee.io` 等のリレーが必要になる点に注意。
- **認証**: GitHub App を推奨（Webhook 署名検証、インストールトークンによる細粒度権限、レート制限枠の恩恵）。個人利用・v1 の立ち上げ簡易化のためには PAT (Personal Access Token) をフォールバックとして許容する。
- **スタック検出の規約**: **base ブランチの連鎖を正とする**。同一リポジトリ内の Open PR 群について `base.ref` を辿り、A → B → C のように base が前段の head と一致する PR 列を機械的に DAG として構築する。これは `git --update-refs`, ghstack, spr, Graphite のいずれのワークフローで作られたスタックでも追加の規約なしに機能する。ただし base 連鎖だけでは判別できないケース（同じ base を共有するが本当は独立した PR、あるいは意図的な非線形依存）のために、PR 本文に **`Depends-on: #123`** という明示的な行を書く規約を上乗せし、これを base 連鎖より優先度の高いオーバーライドとして扱う。Graphite 独自の内部メタデータ（ローカル git config やクラウド側で管理され、GitHub API からは安定して読み取れない）には依存しない。

---

## AI エージェント連携

- **エージェント帰属の規約**: **ブランチ名プレフィックス `agent/<agent-name>/<slug>`**（例: `agent/claude-sonnet/auth-refactor`）を第一の判定材料とする。エージェント側の自動化にラベル付けを覚えさせる必要がなく、PR を見るだけで判別できるため取りこぼしが少ない。ラベル `agent:<name>` は、ブランチ命名規約に従っていない PR（人間が途中から引き継いだ場合など）向けの第二判定材料として併用する。
- **StackDeck はエージェントを「起動」しない（v1 は observe-first）**。"Review Bot を再実行" のようなアクションはエージェントの Webhook/API を呼び出す必要があり、認可・冪等性・失敗時のリトライ設計など別レイヤーの複雑さを持ち込む。v1〜Phase 3 は GitHub 上に既に存在する状態を観測するだけに留め、エージェント起動を伴うアクションは Phase 4 でオプトインの機能として慎重に追加する。

---

## 永続化・状態

- **恒久化すべきもの**: (1) キャッシュされた PR/Check/Review スナップショット、(2) ユーザー設定（テーマ、表示列のフィルタ等）、(3) 直近のビュー状態（選択中スタック・スクロール位置など、再起動後の体験を滑らかにするため）。
- **DB**: SQLite + SQLModel を採用。「キャッシュは使い捨て」という境界を明確にする＝ PR/Check/Review テーブルはいつ消しても Sync Worker が GitHub から再構築できる設計とし、ユーザー設定テーブルとは物理的に区別する（別テーブル、将来的には別ファイルへの分離も検討）。
- マイグレーションは Alembic（SQLModel と親和性が高い）で管理する。

---

## スタイリング / テーマ

- Textual の CSS ライクなスタイルシート（`.tcss`）は、角丸ボーダー（`border: round`）・列ごとのアクセントカラー・Nerd Font グリフのフォールバック（Unicode 代替）を宣言的に記述でき、モックの美観を高い忠実度で再現できる。
- **デザイントークンを単一の情報源（Single Source of Truth）として YAML/JSON で定義**し、そこから (1) Textual 用の `.tcss` 変数、(2) 将来の Web SPA 用の CSS カスタムプロパティ（`--col-ci-green` 等）の両方を生成する。TUI と Web で色の見た目がずれることを構造的に防ぐ。
- 色トークンの命名は §7 に列挙する（`col-ci-green` など）。

---

## データモデル

| エンティティ | 主なフィールド | 備考 |
|---|---|---|
| `Stack` | `id`, `name`（例: `stack/auth-refactor`）, `base_branch`, `agent_id`, `created_at`, `pr_ids[]`（DAG 順） | base ブランチ連鎖 + `Depends-on` オーバーライドから構築 |
| `PullRequest` | `id`, `number`, `title`, `branch`, `base_branch`, `author_agent_id`, `status`（列挙: ci_green / review_bot_done / human_review / merged）, `depends_on[]`, `blocks[]`, `labels: [Label]`, `release_tag?: string`, `created_at`, `updated_at`, `url` | `status` は列（Kanban のカラム）と 1:1 対応。`status = merged` かつ `release_tag` が設定されている場合にカード上へ 🚀 表示（§7）。Deploy 自体は独立した列ではなく `Deploy` エンティティ（後述）で追跡する |
| `Check` | `id`, `pr_id`, `name`（build/test/lint/typecheck/security-scan）, `conclusion`, `started_at`, `completed_at`, `details_url` | GraphQL の `checkSuites` から取得 |
| `Review` | `id`, `pr_id`, `reviewer`（レビュアーの名前/ハンドル）, `reviewer_type`（`human` / `review_bot`）, `state`（`pending` / `approved` / `changes_requested` / `commented`）, `summary`, `requested_at`, `responded_at?`, `age`（`pending` の場合のみ意味を持つ派生値 = `now - requested_at`） | レビューボットのサマリも同じ形で保持。`age` は永続化せず参照時に計算する派生フィールド |
| `Label` | `key`, `value?`, `source`（`"human"` または `"agent:<name>"`）, `created_at` | AI エージェントが付与するラベル（例: `agent:refactor`, `risk:low`, `area:auth`）とヒト由来ラベルを区別して保持。`PullRequest.labels` として多対一 |
| `Agent` | `id`, `name`（例: `claude-sonnet-4.5`）, `kind`（`claude-code`/`other`）, `first_seen_at` | ブランチプレフィックス／ラベルから解決 |
| `Deploy` | `id`, `pr_id`, `environment`, `status`, `release_tag`, `deployed_at`, `url` | Deployment API/Webhook 由来。UI 上はパイプラインの列ではなく、Merge 済みカードの 🚀 マーカーとして表現される（§7） |

**スタック検出の正準規約（再掲）**: base ブランチ連鎖を第一級の信号とし、PR 本文の `Depends-on: #123` 行を明示的なオーバーライドとして扱う。

---

## UI 仕様

### レイアウト（モック準拠、部品名を定義）

```
┌ HeaderBar ─────────────────────────────────────────────────────────────────┐
│ Stacks:3  PRs:12  Agents:2  Deploys:5   API: localhost:8765   Mode: TUI     │
│ Filter: [ label ▾ ]   Group by: [ label ▾ ]     j/k Move  h/l Switch  ...   │
├───────────────┬──────────────────────────────────────────────┬─────────────┤
│ StackListPanel│               PipelineBoard                  │PRDetailPanel│
│               │  CI All Green │ Review Bot Done │ Human Review│ Merged     ││
│ stack/auth-.. │  ┌──────────┐ │  ┌───────────┐  │ ┌──────────┐│┌──────────┐││ branch: ...
│  StackTopo    │  │#123 title│ │  │#122 title │  │ │#121 title││#120 title│││ author: ...
│  Graph        │  │[risk:low]│→│  │[area:auth]│ →│ │👥1/2·⏱3h ││🚀 v1.2.0 │││ Depends on/
│               │  └──────────┘ │  └───────────┘  │ └──────────┘│└──────────┘││ Blocks ...
├───────────────┴──────────────────────────────────────────────┴─────────────┤
│ FooterBar: 最終更新 12:34:56   j/k Move  Enter Open   1/3 pages             │
└──────────────────────────────────────────────────────────────────────────┘
```

- **`StackListPanel`**（左パネル）: スタック一覧。各行に `agent_id`, `created_at`, `base_branch`、および `StackTopologyGraph`（縦方向のミニ DAG）を表示。
- **`PipelineBoard`**（中央）: 列 `CI All Green` / `Review Bot Done` / `Human Review` / `Merged` の 4 列から成る Kanban（デプロイ専用の列は持たない）。各列は `PRCard`（`id`, `title`, ラベルチップ、レビューサマリチップ、必要に応じて 🚀 マーカー）を保持し、同一スタックに属するカードは列を跨いで矢印（`StackFlowArrow`）で接続される。
- **`PRDetailPanel`**（右パネル）: 選択中 PR の詳細。`branch`, `author`（エージェント名）, `created_at`/`updated_at`, `Depends on` / `Blocks`（DAG エッジ）, `status`, 個別チェック結果（build/test/lint/typecheck/security-scan）, ラベル一覧, `ReviewRequestStatus`（レビュー依頼状況、後述）, アクションショートカット（Edit / View on GitHub / Checkout / Merge / Deploy）。
- **`HeaderBar`**（ヘッダー）: 集計（Stacks / PRs / Agents / **Deploys** — Merge 済みかつリリース済み（released）の PR 数を指す）、ラベルバー（`Filter: [ label ▾ ]  Group by: [ label ▾ ]`）、バックエンド API エンドポイント（mock では `localhost:8765`）、現在のモード（TUI / Web）、キーバインド凡例。
- **`FooterBar`**（フッター）: 最終更新時刻、コンテキストに応じたキーヒント、ページネーション。

### レビュー依頼状況（`ReviewRequestStatus`）

`PRDetailPanel` は依頼済みレビュアーごとに以下を一覧表示する。

- レビュアー名/ハンドル（`Review.reviewer`）
- 状態: `Pending` / `Approved` / `Changes requested` / `Commented`
- 依頼からの経過時間（`Pending` の場合のみ）: `requested_at` からの経過を `⏱ 3h` / `⏱ 2d` のように表示し、応答があれば `responded_at` を示す

`PRCard`（カンバン上のカード）ではこれを 1 行に圧縮し、**レビュアー数（承認済み/依頼総数）と最も長く待たされている `Pending` の経過時間**だけを表示する: `👥 1/2 · ⏱ 3h`。この圧縮表示は `Review Bot Done` / `Human Review` 列のカードにのみ出す（`CI All Green` 列やレビュー未依頼の PR では非表示）。

### カード内表示ルール

カードは「今いる列」で自明な情報を繰り返さない。実装者は以下のルールに従うこと。

- `CI All Green` 列のカードに `✓ CI Green` のようなチップを重ねて表示しない（列自体がそれを示している）。
- `Review Bot Done` 列のカードに `Review Bot ✓` を重ねて表示しない。
- `Human Review` 列のカードに `In Review` を重ねて表示しない。
- カード本文に出してよいのは次の要素のみ: PR `id`、`title`、ラベルチップ（色付き、`Label.source` に応じて人間/エージェントを区別可能にする）、レビュー依頼状況の圧縮チップ（`👥 1/2 · ⏱ 3h`、`Review Bot Done`/`Human Review` 列のみ）、および `Merged` 列でリリース済みの場合の 🚀 マーカー + リリースタグ（例: `🚀 v1.2.0`）。

### ラベルによるフィルタ／グルーピング

AI エージェントは PR に自らラベル（例: `agent:refactor`, `risk:low`, `area:auth`）を付与できる。`HeaderBar` のラベルバーから、これらのラベル（人間由来・エージェント由来を問わず）でスタック/PR を絞り込み（`Filter`）、あるいはグループ化（`Group by`）できる。

### キーバインド（原文どおり + ラベル操作）

- `j/k` … Move
- `h/l` … Switch
- `Enter` … Open
- `r` … Refresh
- `q` … Quit
- `?` … Help
- `f` … Filter（ラベルによる絞り込み）
- `g` … Group by（ラベルによるグルーピング）

### 色トークン

| トークン | 用途 |
|---|---|
| `col-ci-green` | `CI All Green` 列のアクセント |
| `col-review-blue` | `Review Bot Done` 列のアクセント |
| `col-review-orange` | `Human Review` 列のアクセント |
| `col-merged-purple` | `Merged` 列のアクセント |
| `col-deploy-teal` | 列アクセントではなく、Merge 済みカード上の 🚀 リリースマーカー + リリースタグの配色（独立した列を持たないため用途を変更） |

---

## ロードマップ

### Phase 0 — Skeleton
- リポジトリ雛形（`pyproject.toml`, `uv` 設定, ディレクトリ構成）
- CI（lint + pytest を GitHub Actions で実行）
- Textual "hello board"（`HeaderBar`/`PipelineBoard`/`FooterBar` の空実装が起動する）
- SQLite マイグレーション基盤（Alembic 初期スキーマ）
- 設定ローダー（GitHub トークン、対象リポジトリ、ポーリング間隔）

**Exit criteria**: `uvx stackdeck` で空のボードが起動し、CI が green である。

### Phase 1 — Read-only TUI MVP
- 単一リポジトリの Open PR を GraphQL で取得
- `PipelineBoard` に実データを描画（列へのマッピングは CI 状態のみで簡易判定）
- `PRDetailPanel` の基本表示（チェック結果・レビュー状況）
- 全キーバインド（`j/k/h/l/Enter/r/q/?`）の実装
- 書き込み系操作は一切なし

**Exit criteria**: 実リポジトリの PR 一覧が正しい列に表示され、詳細パネルが正しいチェック/レビュー情報を出す。

### Phase 2 — Stack awareness
- base ブランチ連鎖によるスタック検出
- `Depends-on:` オーバーライドの解析
- `StackTopologyGraph`（DAG 描画）
- 依存関係を踏まえた列配置（前段が Merge されるまで後段を適切な状態として表示）

**Exit criteria**: 3 段以上のスタックが `StackListPanel` に正しいトポロジーで表示され、依存未解決の PR が誤った列に出ない。

### Phase 3 — Live sync
- Webhook 受信エンドポイント（`pull_request`, `check_suite`, `pull_request_review`, `deployment_status`）
- Webhook + 低頻度ポーリングのハイブリッド更新ロジック
- ブランチプレフィックス／ラベルによるエージェント帰属の解決

**Exit criteria**: GitHub 上の変更が（ポーリング周期を待たずに）数秒以内にボードへ反映される。

### Phase 4 — Actions
- `Checkout`（ローカルへのブランチ取得）
- `Merge`（GitHub API 経由）
- `Deploy`（既存 CI/CD トリガーの起動）
- `Edit PR`（タイトル・base ブランチ等の変更）
- （オプトイン、要検討）エージェント起動アクション（例: Review Bot 再実行）の実験的追加

**Exit criteria**: TUI から Merge/Deploy を実行し、結果が GitHub 側の実状態と一致することを確認できる。

### Phase 5 — Web mode
- `textual-serve` によるブラウザ配信の有効化（Path B）
- API Layer のスキーマ固定（Path A への切替に備えた契約の凍結）
- 認証なしのローカル/社内配信での動作確認

**Exit criteria**: 同一の Textual アプリがターミナルとブラウザの両方で同じ操作感で動作する。

### Phase 6 — Multi-repo / multi-user hardening
- 複数リポジトリの同時監視
- 認証（GitHub OAuth 等）とユーザーごとのビュー
- レート制限予算管理（複数ユーザー/リポジトリでの GraphQL ポイント配分）
- （必要であれば）Path A への本格移行、または PyInstaller によるシングルバイナリ配布の再検討

**Exit criteria**: 3 リポジトリ × 複数ユーザーでレート制限エラーを起こさずに安定稼働する。

---

## リスクと未決事項

- **GraphQL レート制限の見積もり誤差**: PR 数・チェック数が多いリポジトリでは 1 クエリのポイントコストが想定より高くなる可能性があり、Phase 3 のポーリング間隔設計時に実測が必要。
- **エージェント帰属の信頼性**: ブランチプレフィックス規約はエージェント側の実装に依存するため、規約に従わない PR（人間の手動介入など）で帰属が誤判定される可能性がある。
- **`textual-serve` の実運用制約**: サブプロセス起動 + WebSocket 方式のため、同時アクセス数やセッション永続性の限界が実運用でどこまで持つか未検証。`textual-web`（beta）への切替タイミングの判断基準が必要。
- **Webhook 配信の信頼性**: ローカル/社内環境での Webhook 受信には外部リレー（smee.io 等）や公開エンドポイントが必要になり、ポーリングとの二重管理コストが生じる。
- **`Depends-on:` 規約の遵守**: 人間・エージェント双方がこの規約を確実に守るとは限らず、base 連鎖だけでは解決できない複雑な依存関係が見逃される可能性がある。
- **Web 化時の視覚体験の差**: Path B（TUI をそのまま配信）はレスポンシブでもモバイル最適でもないため、非技術者ステークホルダーへの訴求力が限定的になりうる。Path A への移行タイミングの見極めが必要。
- **Merge/Deploy アクションの誤操作リスク**: Phase 4 で書き込み系操作を追加する際、TUI 上の誤操作（キー押し間違い等）が本番環境に影響しうるため、確認ダイアログや取り消し不能操作の明示が必要。

---

## 参考リンク

- [GitHub - Textualize/textual-serve](https://github.com/Textualize/textual-serve)
- [GitHub - Textualize/textual-web](https://github.com/Textualize/textual-web)
- [textual-serve-asgi · PyPI](https://pypi.org/project/textual-serve-asgi/)
- [GitHub - ratatui/ratatui Wasm support discussion #640](https://github.com/ratatui/ratatui/discussions/640)
- [GitHub - NfNitLoop/ratatui-wasm-backend](https://github.com/NfNitLoop/ratatui-wasm-backend)
- [GitHub - charmbracelet/wish](https://github.com/charmbracelet/wish)
- [GitHub - charmbracelet/bubbletea](https://github.com/charmbracelet/bubbletea)
- [Bubble Tea v2: 10x Faster Terminal UIs for Go Developers](https://byteiota.com/bubble-tea-v2-10x-faster-terminal-uis-for-go-developers/)
- [GitHub - vadimdemedes/ink](https://github.com/vadimdemedes/ink)
- [Rate limits and node limits for the GraphQL API - GitHub Docs](https://docs.github.com/en/enterprise-cloud@latest/graphql/overview/rate-limits-and-node-limits-for-the-graphql-api)
- [Rate limits for the REST API - GitHub Docs](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
- [The stacking workflow — stacking.dev](https://www.stacking.dev/)
- [Best Practices For Reviewing Stacked PRs - Graphite](https://graphite.com/docs/best-practices-for-reviewing-stacks)
