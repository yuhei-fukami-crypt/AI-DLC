# Phase 2: Blueprint プレイブック

> Phase 1 の成果物をもとに、技術に依存しない機能設計とアーキテクチャ設計を行い、実装計画（Bolt Plan）を策定する。

## 前提条件

- Phase 1 が完了していること（aidlc-docs/aidlc-state.md で確認）
- 要件が承認され、デザインファイルとUnit分割が確定していること
- `aidlc-docs/inception/` 配下の全成果物が存在すること

## GitHub Projects v2 設定

Step 3 で GitHub Projects v2 を操作する際は、`aidlc-docs/config/github-projects.json` を読み込み、以下の変数として使用する:

| 変数               | JSON パス                            |
| ------------------ | ------------------------------------ |
| `$PROJECT_ORG`     | `.org`                               |
| `$PROJECT_NUMBER`  | `.project_number`                    |
| `$PROJECT_ID`      | `.project_id`                        |
| `$REPO`            | `.repo`                              |
| `$FIELD_STATUS`    | `.fields.BoltStatus`                 |
| `$FIELD_INTENT`    | `.fields.Intent`                     |
| `$FIELD_BOLT`      | `.fields.Bolt`                       |
| `$FIELD_LOCK`      | `.fields.Lock`                       |
| `$OPT_READY`       | `.options.BoltStatus.ready`          |
| `$OPT_BLOCKED`     | `.options.BoltStatus.blocked`        |
| `$OPT_IN_PROGRESS` | `.options.BoltStatus["in-progress"]` |
| `$OPT_DONE`        | `.options.BoltStatus.done`           |
| `$OPT_LOCKED`      | `.options.Lock.locked`               |
| `$OPT_UNLOCKED`    | `.options.Lock.unlocked`             |

## Step 0: GitHub Projects v2 セットアップ

> `aidlc-docs/config/github-projects.json` がプレースホルダーのままの場合、このステップで実際の値を設定する。既にセットアップ済みの場合はスキップする。

### AIの行動

1. **`aidlc-docs/config/github-projects.json` を読み込み、セットアップ済みか確認する**
   - 値が `<...>` プレースホルダーのままであれば、以下の手順でセットアップする
   - 既に実際の値が入っていればスキップする

2. **人間に以下の情報を質問する**
   - GitHub Organization 名（例: `stadium`）
   - リポジトリ名（例: `stadium/stan-monorepo`）
   - GitHub Projects v2 のプロジェクト番号（GitHub Projects の URL から確認可能）

3. **GitHub CLI で Projects v2 の情報を取得する**

   ```bash
   # プロジェクト ID を取得
   gh project view $PROJECT_NUMBER --owner $ORG --format json | jq -r '.id'

   # カスタムフィールドの ID を取得
   gh project field-list $PROJECT_NUMBER --owner $ORG --format json
   ```

4. **フィールドが存在しない場合は作成する**

   フィールド `BoltStatus`, `Intent`, `Bolt`, `Lock` が存在しない場合、`gh project field-create` で作成する:

   ```bash
   # BoltStatus（SINGLE_SELECT）
   gh project field-create $PROJECT_NUMBER --owner $ORG \
     --name "BoltStatus" --data-type "SINGLE_SELECT" \
     --single-select-options "ready,blocked,in-progress,done"

   # Intent（TEXT）
   gh project field-create $PROJECT_NUMBER --owner $ORG \
     --name "Intent" --data-type "TEXT"

   # Bolt（TEXT）
   gh project field-create $PROJECT_NUMBER --owner $ORG \
     --name "Bolt" --data-type "TEXT"

   # Lock（SINGLE_SELECT）
   gh project field-create $PROJECT_NUMBER --owner $ORG \
     --name "Lock" --data-type "SINGLE_SELECT" \
     --single-select-options "locked,unlocked"
   ```

   作成後、`gh project field-list` で各フィールドの ID とオプション ID を取得する。

5. **取得した情報で `aidlc-docs/config/github-projects.json` を書き込む**

   手順 2〜4 で得た値をテンプレートに埋め込み、ファイルを更新する:

   ```jsonc
   {
     "project_name": "<人間に確認したプロジェクト名>",
     "org": "<ORG>",
     "repo": "<ORG>/<REPO>",
     "project_number": "<PROJECT_NUMBER>",
     "project_id": "<gh project view で取得した ID>",
     "fields": {
       "BoltStatus": "<field-list から取得した BoltStatus の ID>",
       "Intent": "<field-list から取得した Intent の ID>",
       "Bolt": "<field-list から取得した Bolt の ID>",
       "Lock": "<field-list から取得した Lock の ID>",
     },
     "options": {
       "BoltStatus": {
         "ready": "<option ID>",
         "blocked": "<option ID>",
         "in-progress": "<option ID>",
         "done": "<option ID>",
       },
       "Lock": {
         "locked": "<option ID>",
         "unlocked": "<option ID>",
       },
     },
   }
   ```

   - `gh project field-list` の JSON 出力から各フィールドの `id` と、SINGLE_SELECT フィールドの `options[].id` を抽出して埋める
   - 書き込み後、ファイルを読み直して値が正しく設定されていることを確認する

### 完了条件

- `aidlc-docs/config/github-projects.json` が実際の値で設定されている（プレースホルダーが残っていない）
- 必要なカスタムフィールド（BoltStatus, Intent, Bolt, Lock）が GitHub Projects v2 に存在する
- `aidlc-docs/aidlc-state.md` の Step 0 チェックボックスを `[x]` にする

---

## フェーズ開始時の成果物読み込み

Phase 2 の各ステップに入る前に、以下の成果物を一括で読み込む:

- `aidlc-docs/inception/intent.md`
- `aidlc-docs/inception/requirements.md`
- `aidlc-docs/inception/user-stories.md`
- `aidlc-docs/inception/qa-plan.md`
- `aidlc-docs/inception/units.md`
- `aidlc-docs/inception/` 配下のデザインファイル（.pen）
- `aidlc-docs/blueprint/functional-design.md`（存在する場合 — 追加開発時）
- `aidlc-docs/blueprint/architecture.md`（存在する場合 — 追加開発時）

## デザインファイルの扱い

- Phase 1 で作成したデザインファイル（.pen）は **UI・画面構成の参照** として使用する
- プロダクションコードはゼロから構築する（初回のみ。追加開発時は既存コードに追加）

## 設計ファイルの構成

設計ファイルは **全体設計** と **Unit設計** の2層構造で管理する。

```
blueprint/
  functional-design.md       ← 全体: プロダクト全体のドメインモデル・ビジネスルール
  architecture.md            ← 全体: プロダクト全体のAPI・データモデル・インフラ
  units/
    {unit-name}/
      functional-design.md   ← Unit: この Unit で追加・変更したスコープ
      architecture.md        ← Unit: この Unit で追加・変更したスコープ
      bolt-plan.md           ← Unit: この Unit の Bolt 分割計画
```

|                | 全体設計                                 | Unit設計                    |
| -------------- | ---------------------------------------- | --------------------------- |
| 目的           | プロダクトの現在の全体像                 | 今回のスコープの明確化      |
| 更新タイミング | Unit設計生成後すぐに統合（統合後に承認） | 各ステップで生成            |
| 使う場面       | 整合性チェック、新規開発時の参照         | Bolt Plan・Execution の入力 |
| 内容           | 累積（全エンティティ、全API）            | 差分（追加・変更のみ）      |

### Unit名の決定

- Unit名は Inception の Intent から導出する（例: `voting-basic`, `voting-ranking`）
- 人間に確認: 「今回の Unit名は `{unit-name}` でよいですか？」

### 複数 Unit の進め方

`inception/units.md` に複数の Unit が定義されている場合、**Step 1〜3 を Unit ごとに繰り返す**。

1. **開始時**: `inception/units.md` から Unit 一覧と実行順序を読み込む
2. **Unit ループ**: 各 Unit について Step 1 → Step 2 → Step 3 を順に実行する
3. **Unit 完了時の記録**: 各 Unit の Step 3 が完了したら、`aidlc-docs/aidlc-state.md` の該当 Unit セクションのチェックボックスを `[x]` にする
4. **中断・再開**: 途中で会話が途切れた場合、`aidlc-state.md` の Unit セクションを確認し、未完了の Unit から再開する

#### state.md の Unit セクション管理

最初の Unit に着手する前に、`aidlc-state.md` の Phase 2 セクションに Unit 一覧を追記する:

```markdown
## Phase 2: Blueprint（設計）

- [x] Step 0: GitHub Projects v2 セットアップ

### Unit: voting-basic

- [ ] Step 1: Functional Design
- [ ] Step 2: Architecture
- [ ] Step 3: Bolt Plan

### Unit: voting-ranking

- [ ] Step 1: Functional Design
- [ ] Step 2: Architecture
- [ ] Step 3: Bolt Plan
```

Unit が1つだけの場合も同じフォーマットを使用する。

---

## Step 1: Functional Design（機能設計）

> 技術に依存しない、純粋なビジネスロジックの設計を行う。

### AIの行動

1. **既存の全体設計を把握する（追加開発時）**
   - `blueprint/functional-design.md` が存在する場合、既存のドメインモデル・ビジネスルールを把握する
   - 既存エンティティとの関係・影響範囲を分析する

2. **要件からドメインモデルを抽出し、人間と対話で確認する**
   - requirements.md、user-stories.md、デザインファイル（.pen）を読み込む
   - 今回の Unit で追加・変更するエンティティを特定し一覧化する
   - **エンティティ一覧を人間に提示する**:
     - 「以下のエンティティを抽出しました。過不足はありますか？」
     - 各エンティティの名前・説明・主なリレーションを表形式で提示する
   - 人間の確認を得てから次に進む

3. **エンティティ間のリレーションを定義し、人間と確認する**
   - 承認されたエンティティ一覧をもとに、リレーション（1対多、多対多等）を定義する
   - **確認時の表示形式**（ターミナルでの視認性を優先）:
     - リレーション一覧を **表形式** で提示する（From / To / 関係 / 説明）
     - エンティティ間の構造を **ASCII アート** で図示する
   - 成果物 MD には Mermaid コードブロックとしても記録する
   - **ドメインモデル図を人間に提示する**:
     - 「エンティティ間の関係はこの理解で合っていますか？」
     - 曖昧な関係がある場合は選択肢を提示して人間に選ばせる
   - 人間の確認を得てから次に進む

4. **ビジネスロジックを設計し、人間と確認する**
   - 各ユーザーストーリーに対応するビジネスロジック（処理フロー）を記述する
   - 入力・出力・処理ステップを明確にする
   - **ビジネスロジックの解釈が複数あり得る場合は質問する**:
     - 例: 「投票の日付切り替わりは JST 00:00 でよいですか？」
     - 例: 「コメントなし投票にもいいねできるべきですか？」
   - **ビジネスロジック一覧を人間に提示する**:
     - 「以下のビジネスロジックで問題はありますか？」
   - 人間の確認を得てから次に進む

5. **ビジネスルールを定義し、人間と確認する**
   - バリデーションルール（必須項目、形式、範囲など）を一覧化する
   - 状態遷移ルール（ステータスの遷移条件など）を定義する
   - 認可ルール（誰がどの操作を実行できるか）を明記する
   - ビジネス上の制約条件を記述する
   - **ビジネスルール一覧を人間に提示する**:
     - 「以下のビジネスルールで問題はありますか？追加すべきルールはありますか？」
   - 人間の確認を得てから成果物を生成する

### 対話のルール

- **1件ずつ確認しながら進める**: ドメインモデル → リレーション → ビジネスロジック → ビジネスルールの順に、各段階で人間の確認を得てから次に進む
- AIは常に「こう理解しました」と要約してから確認を求める
- 人間が曖昧な場合、AIが仮の答えを提案し「これでよいですか？」と聞く
- ビジネスロジックの解釈が複数あり得る場合は、一気に進めず質問する
- エッジケースの扱いは人間に確認する（AIの裁量で進めない）
- 人間のペースに合わせる。一度にすべてを出さず、段階的に提示する

### 成果物の生成

1. **Unit設計を生成する**
   - `aidlc-docs/blueprint/units/{unit-name}/functional-design.md` を生成する
   - 今回の Unit で追加・変更するスコープのみを記述する
   - テンプレート: `aidlc-workflows/templates/aidlc-docs/blueprint/units/unit-functional-design.md` を使用

2. **全体設計に統合する**
   - `aidlc-docs/blueprint/functional-design.md` を更新する（初回は新規作成）
   - Unit設計の内容を全体設計にマージし、整合性を確認する
   - テンプレート: `aidlc-workflows/templates/aidlc-docs/blueprint/functional-design.md` を使用

### 自己レビュー（人間に提示する前に実行）

- [ ] 全ユーザーストーリーに対応するビジネスロジックが定義されているか
- [ ] エンティティ間のリレーションが明確に定義されているか
- [ ] バリデーションルール・状態遷移ルール・認可ルールが網羅されているか
- [ ] ユビキタス言語（intent.md）と用語が一致しているか
- [ ] テンプレートの HTML コメント（指示文）が残っていないか
- [ ] ドメインモデル・ビジネスロジック・ビジネスルールの各段階で人間の確認を得ている
- [ ] Unit設計と全体設計の両方が生成されている
- [ ] aidlc-state.md の該当チェックボックスを [x] にした
- 不合格の項目がある場合、成果物を修正してから人間に提示する

---

## Step 2: Architecture（アーキテクチャ設計）

> Functional Design で定義した「何をするか」を踏まえ、「どう実現するか」を設計する。

### AIの行動

1. **既存の全体設計を把握する（追加開発時）**
   - `blueprint/architecture.md` が存在する場合、既存の技術スタック・API・データモデルを把握する
   - 既存アーキテクチャとの整合性を意識する

2. **技術スタックを提案し、人間と確認する（初回のみ）**
   - 要件・UI構成・ビジネスロジックの複雑さに適した技術スタックを2〜3パターン提案する
   - 各パターンのメリット・デメリットを提示する
   - **アプリケーション単位で整理する**: フロントエンド（Web / Native）・バックエンド・共通基盤など、アプリケーションごとにサブセクションを分け、各技術にレイヤー（UI フレームワーク / ルーティング / データフェッチ等）とバージョンを明記する
   - 人間に確認: 「どのパターンが良いですか？または既に決まっている技術はありますか？」
   - **人間の確認を得てから次に進む**
   - 追加開発時は既存の技術スタックに従う

3. **システム構成を設計し、人間と確認する**
   - 選定した技術スタックに基づき全体構成図を作成する
   - **確認時の表示形式**: コンポーネント一覧を **表形式** で、構成の全体像を **ASCII アート** で図示する
   - 成果物 MD には Mermaid コードブロックとしても記録する
   - インフラ構成（デプロイ先、ネットワーク、CDN等）を記述する
   - ディレクトリ構成を設計する
   - **インフラコスト概算を算出する**:
     - Web検索でクラウドプロバイダの公式料金ページから各サービスの最新単価を調べる
     - 各コンポーネント（コンピュート、DB、ストレージ、ネットワーク、CDN、認証等）の月額コストを概算する
     - Inception の想定ユーザー規模（requirements.md の NFR）を前提とする
     - 最小構成と想定規模の2パターンで算出する
     - 合計月額コストを表形式でまとめる
   - **システム構成図・ディレクトリ構成・コスト概算を人間に提示する**:
     - 「このシステム構成・コスト感で進めてよいですか？」
   - 人間の確認を得てから次に進む

4. **API設計を行い、人間と確認する**
   - 今回の Unit で追加・変更するAPIエンドポイントの一覧を作成する（メソッド、パス、説明）
   - リクエスト/レスポンスの概要を記述する
   - 認証・認可の方式を明記する
   - サーバーがない場合: データアクセスパターンを記述する
   - **API一覧を人間に提示する**:
     - 「このAPI設計で問題はありますか？追加・変更すべきエンドポイントはありますか？」
   - 人間の確認を得てから次に進む

5. **データモデル（物理）を設計し、人間と確認する**
   - Functional Design のドメインモデルを物理テーブル/コレクションに変換する
   - 今回の Unit で追加・変更するテーブル/カラムを定義する
   - 各テーブルの属性名・型・制約を定義する
   - ER図を作成する
   - **確認時の表示形式**: テーブル定義を **表形式** で、リレーションを **ASCII アート** で図示する
   - 成果物 MD には Mermaid コードブロックとしても記録する
   - インデックス戦略を定義する（外部キー、検索対象カラム、クエリパターン）
   - API ↔ データモデルのマッピングを作成する
   - **データモデルを人間に提示する**:
     - 「このテーブル設計で問題はありますか？」
   - 人間の確認を得てから次に進む

6. **NFRに関する意思決定を確認する**
   - `inception/requirements.md` のビジネス視点の非機能要件（対応環境・想定ユーザー規模等）を踏まえる
   - パフォーマンス要件（レスポンスタイム目標、スループット等）
   - スケーラビリティ方針（水平/垂直スケーリング、キャッシュ戦略等）
   - セキュリティ方針（暗号化、脆弱性対策等）
   - 可用性・信頼性の方針
   - **NFR方針を人間に提示する**:
     - 「この非機能要件の方針で問題はありますか？」
   - 人間の確認を得てから次に進む

7. **技術的な意思決定を記録する**
   - 重要な技術選定について、選択肢・比較・決定理由を記録する
   - **トレードオフがある場合は人間に選択肢を提示して判断を仰ぐ**

8. **QA Fixture の Seed SQL を設計する**
   - Phase 1 の `inception/qa-plan.md` の Fixture 概念設計を読み込む
   - 手順 5 で確定した物理データモデル（テーブル定義）に基づき、具体的な Seed SQL を設計する:
     - `qa/fixtures/seeds/base.sql` — 全シナリオ共通のマスタデータ INSERT
     - `qa/fixtures/seeds/{unit-name}/{bolt-id}.sql` — Bolt QA 単体実行時に必要な前提データ INSERT（前の Bolt の成果物の代替）
   - Firebase Auth テストユーザー定義を設計する:
     - `qa/fixtures/auth/test-users.json` — UID, email, password の一覧
   - **投入方式**: dump/restore ツール（JSON 形式）ではなく、spanner-cli で SQL を直接実行する
     - DB リセット: `wrench reset --directory database/` でスキーマだけの空 DB にする
     - Seed 投入: `spanner-cli -p $PROJECT -i $INSTANCE -d $DATABASE < base.sql` → `< {bolt}.sql` の順に実行
     - 複数ファイルの順次適用が可能で、INSERT の依存順序（親テーブル → 子テーブル）を SQL ファイル内で制御する
   - Seed SQL が外部キー制約・NOT NULL 制約・インターリーブ等と整合しているか検証する
   - 人間に確認: 「この QA Fixture で問題はありますか？」

### 対話のルール

- **1件ずつ確認しながら進める**: 技術スタック → システム構成・コスト概算 → API設計 → データモデル → NFR → QA Fixture の順に、各段階で人間の確認を得てから次に進む
- AIは常に「こう理解しました」と要約してから確認を求める
- 人間が曖昧な場合、AIが仮の答えを提案し「これでよいですか？」と聞く
- 技術選定でトレードオフがある場合は、一気に進めず選択肢を提示する
- 人間のペースに合わせる。一度にすべてを出さず、段階的に提示する

### 成果物の生成

1. **Unit設計を生成する**
   - `aidlc-docs/blueprint/units/{unit-name}/architecture.md` を生成する
   - 今回の Unit で追加・変更するスコープのみを記述する
   - テンプレート: `aidlc-workflows/templates/aidlc-docs/blueprint/units/unit-architecture.md` を使用

2. **全体設計に統合する**
   - `aidlc-docs/blueprint/architecture.md` を更新する（初回は新規作成）
   - Unit設計の内容を全体設計にマージし、整合性を確認する
   - テンプレート: `aidlc-workflows/templates/aidlc-docs/blueprint/architecture.md` を使用

### 自己レビュー（人間に提示する前に実行）

- [ ] 技術スタックが要件（パフォーマンス、スケーラビリティ等）に適合しているか
- [ ] インフラコスト概算が全主要コンポーネントを網羅しているか（漏れがないか）
- [ ] API設計が Functional Design の全ユースケースをカバーしているか
- [ ] データモデルが Functional Design の全エンティティを物理設計に変換しているか
- [ ] インデックス戦略がクエリパターンと整合しているか
- [ ] NFRの意思決定が requirements.md の非機能要件と矛盾していないか
- [ ] QA Fixture の Seed SQL が物理データモデルの制約（外部キー、NOT NULL、インターリーブ等）と整合しているか
- [ ] QA Fixture が qa-plan.md の概念設計を網羅しているか
- [ ] テンプレートの HTML コメント（指示文）が残っていないか
- [ ] 技術スタック・システム構成・API設計・データモデル・NFR・QA Fixture の各段階で人間の確認を得ている
- [ ] Unit設計と全体設計の両方が生成されている
- [ ] aidlc-state.md の該当チェックボックスを [x] にした
- 不合格の項目がある場合、成果物を修正してから人間に提示する

---

## Step 3: Bolt Plan（Bolt 分割計画）

> Unit の設計成果物を実装可能な Bolt に分割し、受入基準を定義する。Unit 単位で人間の承認を得る。

### Bolt 分割の原則

**Bolt は Unit 内で並列実行可能な「縦割りの機能スライス」である。**

各 Bolt はフルスタック（DB + API + UI）を含む独立した機能単位とし、Bolt 間の依存を最小化して最大限の並列実行を実現する。Bolt 内部の縦の依存（型定義 → コア実装 → 結合）は各 Bolt の実装エージェントが Skill の実行順序に従って制御するため、Bolt レベルでレイヤー分割してはならない。

| 原則                    | 説明                                                                                                                   |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **縦割り分割**          | 1 Bolt = 独立した機能スライス（DB + API + UI）。レイヤー単位（DB だけ、API だけ）で分割しない                          |
| **並列実行可能**        | 原則として全 Bolt が同時実行可能。依存は共通基盤 Bolt への片方向のみ許容                                               |
| **共通基盤の例外**      | 認証・共通テーブル・共有型定義など、複数 Bolt の前提となる基盤は共通基盤 Bolt（B0）として先行実行を許容する            |
| **Bolt 内外の責務分離** | Bolt 間 = 機能の独立性、Bolt 内 = Skill 実行順序によるレイヤーの依存順序。二重の依存チェーンを作らない                 |
| **Bolt ID の命名**      | 基本は `B{N}`（B0, B1, B2 ...）。1つの Bolt をさらに細分化する場合はサブレター `B{N}{letter}`（B2a, B2b 等）を使用する |

#### 良い分割・悪い分割の例

```
❌ 悪い例（レイヤー分割 → 直列依存）:
   B1: DB スキーマ → B2: API 実装 → B3: フロント実装

✅ 良い例（機能スライス → 並列実行）:
   B0: 共通基盤（認証・共通テーブル）
   B1: 投票機能（DB + API + UI）  ← B0 に依存
   B2: コメント機能（DB + API + UI）← B0 に依存
   B1 と B2 は並列実行可能
```

### AIの行動

1. **Unit の設計成果物を読み込む**
   - `blueprint/units/{unit-name}/functional-design.md` と `blueprint/units/{unit-name}/architecture.md` を読み込む
   - 全体設計 `blueprint/architecture.md` を参照し、既存コードとの整合性を確認する
   - `inception/user-stories.md` の受入基準を確認する

2. **共通基盤の有無を判断する**
   - 複数の機能で共有されるスキーマ・API・型定義があるか分析する
   - 共通基盤がある場合は B0（共通基盤 Bolt）として切り出す
   - 共通基盤がない場合は全 Bolt を依存なしの並列とする

3. **Bolt に分割する**
   - Unit のスコープを**並列実行可能な縦割りの機能スライス**で Bolt に分割する
   - 各 Bolt はフルスタック（DB + API + UI）を含む独立した機能単位とする
   - 各 Bolt のスコープ（実装する機能・API・テーブル）を明確にする
   - 各 Bolt が `user-stories.md` のどの受入基準をカバーするかを対応付ける
   - 各 Bolt に対応する QA シナリオ（`inception/qa-plan.md` より）をマッピングする。Phase 1 で生成済みのシナリオファイル（`qa/scenarios/web/{unit-name}/{name}.yml`）のパスを bolt-plan.md の各 Bolt 詳細に記載する
   - 主な成果物（生成されるファイル群の概要）を記述する
   - **自己検証**: Bolt 間に B0 以外への依存が発生していないか確認する。発生している場合は分割を見直す

4. **依存関係を定義する**
   - **Unit 内依存**: 共通基盤 Bolt（B0）がある場合、他の Bolt から B0 への依存を定義する
   - B0 以外の Bolt 間には依存を設定しない（並列実行可能であることを明示する）
   - **B0 以外の Bolt 間に依存が必要な場合は分割の見直しを検討する**（共通部分を B0 に移動するか、Bolt を統合する）
   - **Unit 間依存**: `inception/units.md` の依存関係図を参照し、現在の Unit が前の Unit に依存している場合、現在の Unit の最初の Bolt（B0 がある場合は B0、ない場合は B1）に対して前の Unit の最後の Bolt への依存を設定する。これにより `run-bolt` の Dependency Verification が Unit 間の実行順序を強制する

5. **Bolt Plan を人間に提示し承認を得る**
   - Bolt 一覧と並列実行計画を提示する
   - 共通基盤 Bolt がある場合はその理由を説明する
   - 人間に確認: 「この Bolt 分割で進めてよいですか？」
   - 承認されるまで次に進まない

6. **Bolt を GitHub Issue として登録する**

   承認後、以下の手順で各 Bolt を GitHub Issue として登録する。

   #### 6-1. 事前準備
   - `gh auth status` で GitHub CLI が認証済みであることを確認する
   - ラベル `bolt` が存在しない場合は作成する: `gh label create bolt --description "Bolt unit" --color "0E8A16"`
   - 同じタイトルの Issue が既に存在しないかチェックする: `gh issue list --label bolt --state open`
     - 既に存在する Bolt はスキップし、警告を表示する

   #### 6-2. 各 Bolt の Issue を作成する

   bolt-plan.md の各 Bolt について以下を実行する。

   **`Depends On` フィールドの決定ルール:**
   - **Unit 内依存**（bolt-plan.md に記載）: 同一 Unit 内の B0 等への依存
   - **Unit 間依存**（`inception/units.md` に記載）: 前の Unit の最後の Bolt への依存
   - Unit 間依存がある Bolt（= 依存 Unit の最後の Bolt が完了するまで実行できない Bolt）には、依存先 Issue 番号を `Depends On` に記載する
   - Unit 内依存と Unit 間依存の両方がある場合は、両方を記載する

   > **⚠️ 重要**: `Depends On` に依存先 Issue の `#番号` を**必ず記載すること**。Phase 3 の Bolt パイプライン cleanup ステップは Issue body の `#番号` から依存関係を検出して下流 Bolt を自動 unblock する。記載漏れがあると下流 Bolt が blocked のまま残り、手動介入が必要になる。依存がない Bolt（通常 B0）は `Depends On: —` と記載する。

   ```bash
   gh issue create \
     --title "[I{intent}-B{N}] {unit-name}: {bolt-title}" \
     --label "bolt" \
     --body "$(cat <<'EOF'
   ## Bolt Definition

   - **Bolt ID:** B{N}
   - **Intent:** {intent-id}
   - **Unit:** {unit-name}
   - **Depends On:** {依存 Bolt 一覧（Unit 内 + Unit 間）、なければ "—"}

   ## Scope

   {bolt-plan.md の該当 Bolt 詳細セクションのスコープ}

   ## Acceptance Criteria

   {bolt-plan.md の該当 Bolt 詳細セクションの受入基準}

   ## QA Scenarios

   {bolt-plan.md の該当 Bolt にマッピングされた QA シナリオファイル一覧}
   - `qa/scenarios/web/{unit-name}/{name}.yml`

   ## Key Deliverables

   {bolt-plan.md の該当 Bolt 詳細セクションの主な成果物}
   EOF
   )"
   ```

   > **例**: Unit 2 の B0 は Unit 内では依存なしだが、`units.md` で Unit 1 に依存している場合、
   > `Depends On: #328 (Unit 1 最後の Bolt: [I1-B2] Admin 認証)` と記載する。

   #### 6-3. GitHub Projects v2 に追加しフィールドを設定する

   各 Issue を Projects v2 に追加し、カスタムフィールドを設定する。

   **BoltStatus の決定ルール（Unit 内依存 + Unit 間依存の両方を考慮する）:**

   | 条件                                                         | BoltStatus |
   | ------------------------------------------------------------ | ---------- |
   | Unit 内で依存なし（B0）かつ Unit 間で依存なし（最初の Unit） | `ready`    |
   | Unit 内で依存なし（B0）だが、前の Unit が未完了              | `blocked`  |
   | Unit 内で依存あり（B0 以外）                                 | `blocked`  |

   **Unit 間依存の判定方法:**
   - `inception/units.md` の依存関係図・実行順序を参照する
   - 現在の Unit が依存する Unit がある場合、その Unit の**最後の Bolt**（番号が最大の Bolt）を依存先として Issue body の `Depends On` に記載する
   - 例: Unit 2 の最初の Bolt（B0 または B1）は、Unit 1 の最後の Bolt（例: #328）に依存する

   ```bash
   # Project に追加し、Intent / Bolt フィールドを設定する
   ITEM_ID=$(gh project item-add $PROJECT_NUMBER --owner $PROJECT_ORG --url "https://github.com/$REPO/issues/{issue-number}" --format json | jq -r '.id')

   # Intent フィールド
   gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
     --field-id $FIELD_INTENT --text "I{intent}"
   # Bolt フィールド
   gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
     --field-id $FIELD_BOLT --text "B{N}"
   # BoltStatus: Unit 内依存なし AND Unit 間依存なし → $OPT_READY、それ以外 → $OPT_BLOCKED
   gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
     --field-id $FIELD_STATUS --single-select-option-id $OPT_READY  # or $OPT_BLOCKED
   # Lock: unlocked
   gh project item-edit --project-id $PROJECT_ID --id "$ITEM_ID" \
     --field-id $FIELD_LOCK --single-select-option-id $OPT_UNLOCKED
   ```

   #### 6-4. bolt-plan.md に Issue 番号を追記する

   Bolt 一覧テーブルの `Issue #` カラムに、作成した Issue の番号を書き込む:

   ```
   | Bolt ID | タイトル                       | 依存 Bolt | ステータス | Issue # |
   |---------|-------------------------------|----------|-----------|---------|
   | B0      | voting: 共通基盤               | —        | ready     | #42     |
   | B1      | voting: 投票機能               | B0       | blocked   | #43     |
   | B2      | voting: コメント機能           | B0       | blocked   | #44     |
   ```

   #### 6-5. 作成した Issue 一覧を人間に表示する

   作成した全 Issue の番号・タイトル・ステータス・URL を一覧表示する。

7. **QA Suite ファイルを生成する**

   Bolt Plan が確定し Issue が登録されたら、Phase 1 で生成済みの QA シナリオと Bolt のマッピングに基づき Suite ファイルを生成する。

   #### 7-1. Bolt Suite を生成する

   各 Bolt について `qa/suites/bolt/{unit-name}/{bolt-id}.yml` を生成する:

   ```yaml
   name: "{unit-name} {bolt-id} QA"
   level: bolt
   setup:
     # 1. wrench reset でスキーマだけの空 DB にする
     # 2. spanner-cli で Seed SQL を順次実行する
     db_reset: true # wrench reset
     seeds: # spanner-cli < file の順次実行
       - "qa/fixtures/seeds/base.sql"
       - "qa/fixtures/seeds/{unit-name}/{bolt-id}.sql" # 存在する場合のみ
     auth_seed: "qa/fixtures/auth/test-users.json" # Firebase Auth emulator REST API
   scenarios:
     # bolt-plan.md のマッピングに基づき、この Bolt に対応するシナリオを列挙
     - "qa/scenarios/web/{unit-name}/{name}.yml"
   ```

   #### 7-2. Unit Suite を生成する

   各 Unit について `qa/suites/unit/{unit-name}.yml` を生成する。`inception/qa-plan.md` の Unit QA 概念設計に基づき、正常系シナリオを依存順に列挙する:

   ```yaml
   name: "{unit-name} Unit QA"
   level: unit
   setup:
     db_reset: true
     seeds:
       - "qa/fixtures/seeds/base.sql"
     auth_seed: "qa/fixtures/auth/test-users.json"
   scenarios:
     # Bolt 依存順に正常系シナリオを列挙（先行シナリオの結果が後続の入力データとなる）
     # Unit QA では Bolt 固有 seed は使わない — 先行シナリオの実行結果がデータとなる
     - "qa/scenarios/web/{unit-name}/{name}.yml"
   ```

   #### 7-3. Inception Suite を更新する

   `qa/suites/inception/full.yml` を生成または更新する:

   ```yaml
   name: "Full Inception QA"
   level: inception
   setup:
     db_reset: true
     seeds:
       - "qa/fixtures/seeds/base.sql"
     auth_seed: "qa/fixtures/auth/test-users.json"
   unit_suites:
     # Unit 依存順に全 Unit Suite を列挙
     # Inception QA では base seed のみ — 全シナリオの積み上げでデータが構築される
     - "qa/suites/unit/{unit-name-1}.yml"
     - "qa/suites/unit/{unit-name-2}.yml"
   ```

### 成果物の生成

- `aidlc-docs/blueprint/units/{unit-name}/bolt-plan.md` を生成する
- テンプレート: `aidlc-workflows/templates/aidlc-docs/blueprint/units/bolt-plan.md` を使用
- `qa/suites/bolt/{unit-name}/` 配下に Bolt Suite ファイルを生成する
- `qa/suites/unit/{unit-name}.yml` を生成する
- `qa/suites/inception/full.yml` を生成または更新する

### 自己レビュー（人間に提示する前に実行）

- [ ] 全 Bolt が縦割りの機能スライス（DB + API + UI）になっているか（レイヤー分割していないか）
- [ ] B0 以外の Bolt 間に依存関係がないか（並列実行可能か）
- [ ] 全ユーザーストーリーの受入基準がいずれかの Bolt にマッピングされているか
- [ ] qa-plan.md の全 QA シナリオがいずれかの Bolt にマッピングされているか
- [ ] 各 Suite ファイルの参照パスが実際のシナリオファイルと一致しているか
- [ ] 各 Bolt の Issue に QA Scenarios セクションが記載されているか
- [ ] 各 Bolt のスコープ（実装する機能・API・テーブル）が明確に記述されているか
- [ ] Bolt ID の命名規則（`B{N}` / `B{N}{letter}`）に従っているか
- [ ] bolt-plan.md が人間に承認されている
- [ ] 各 Bolt が GitHub Issue として登録され、Projects v2 にフィールドが設定されている
- [ ] aidlc-state.md の該当チェックボックスを [x] にした
- 不合格の項目がある場合、成果物を修正してから人間に提示する
- **未完了の Unit がある場合**: 次の Unit の Step 1 に進む
- **全 Unit が完了した場合**: Step 4（Blueprint Approval Gate）に進む

---

## Step 4: Blueprint Approval Gate（全体承認）

> **承認ゲート**: 全 Unit の設計と Bolt Plan を横断的にレビューし、Phase 3 に進んでよいか判断する。

### 前提: Project Adapter の作成

Phase 3 の Bolt パイプラインは `aidlc-docs/config/project-adapter.md` を必要とする。Approval Gate の承認前に、このファイルを作成・確認する。

1. **`aidlc-docs/config/project-adapter.md` が存在するか確認する**
   - 存在しない場合、テンプレート（`aidlc-workflows/templates/aidlc-docs/config/project-adapter.md`）から作成する
2. **プロジェクトの技術スタック・設計成果物をもとに Project Adapter を定義する**
   - Workstreams（実装エージェント、Trigger paths、Owned paths、Skills）
   - Gate Commands（CI 相当のチェックコマンド）
   - Review Configuration（オプション）
   - Workspace Setup（依存パッケージインストール）
3. **人間に確認する**: 「この Project Adapter の設定で進めてよいですか？」

### AIの行動

1. **全 Unit の設計成果物を横断チェックする**
   - 全体設計（`blueprint/functional-design.md`, `blueprint/architecture.md`）と各 Unit 設計の整合性を確認する
   - Unit 間のドメインモデル・API・データモデルに矛盾がないか確認する
   - 全 Bolt Plan の依存関係が正しく、並列実行計画に問題がないか確認する
   - 漏れ・矛盾が見つかった場合、該当 Unit の該当ステップに戻って修正する

2. **Blueprint 全体のサマリーを提示する**
   - 全体のドメインモデル概要
   - 技術スタックとアーキテクチャ概要
   - 全 Unit の Bolt Plan と並列実行計画
   - 「Blueprint 全体をレビューしてください」と問う

3. **人間の判断を受ける**
   - **承認**: Phase 2 を完了とし、state を更新する。Phase 3 の実行開始は人間が別途指示する（AIから実装開始を提案しない）
   - **フィードバックあり**: 共通のフィードバックループ手順に従い、該当ステップに戻る（`overview.md` > フィードバックループ手順（共通）参照）

### 完了条件

- 全 Unit の設計成果物間に矛盾がない
- `aidlc-docs/config/project-adapter.md` が作成され、人間が確認している
- 人間が承認した
- `aidlc-docs/aidlc-state.md` の Step 4 チェックボックスを `[x]` にする
- `aidlc-docs/aidlc-state.md` の `Current Phase` を `Phase 3 - Construction` に更新する
- **人間にコミットを促す**: 「Phase 2 Blueprint の全成果物が承認されました。現在の変更をコミットしておくことをお勧めします。コミットしますか？」
- **注意**: Phase 3 の実行開始は人間が別途指示する。AIから「実装に進みますか？」等の提案をしないこと
