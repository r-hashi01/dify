# CI/CD パイプライン構成

このドキュメントは、Difyプロジェクトの継続的インテグレーション/継続的デリバリー（CI/CD）パイプラインの全体構成を説明します。

## 概要

Difyプロジェクトは、GitHub Actionsを使用した包括的なCI/CDパイプラインを実装しています。このパイプラインは、コード品質の保証、自動テスト、Dockerイメージのビルド・プッシュ、およびデプロイメント自動化をカバーしています。

## ワークフロー一覧

### 1. メインCIパイプライン (`main-ci.yml`)

**トリガー**: PRおよびmainブランチへのpush

**概要**: 変更されたファイルパスに基づいて、必要なテストとチェックを並列実行する統合パイプライン

**ジョブフロー**:
```
check-changes
    ├─> api-tests (api変更時)
    ├─> web-tests (web変更時)
    ├─> style-check (常に実行)
    ├─> vdb-tests (VDB関連変更時)
    └─> db-migration-test (migration変更時)
```

**変更検知パス**:
- **API**: `api/**`, `docker/**`, `.github/workflows/api-tests.yml`
- **Web**: `web/**`
- **VDB**: `api/core/rag/datasource/**`, `docker/**`, `.github/workflows/vdb-tests.yml`, `api/uv.lock`, `api/pyproject.toml`
- **Migration**: `api/migrations/**`, `.github/workflows/db-migration-test.yml`

**並行制御**: `main-ci-${{ github.head_ref || github.run_id }}`

---

### 2. APIテスト (`api-tests.yml`)

**トリガー**: `main-ci.yml`から呼び出し（workflow_call）

**Python バージョンマトリクス**: 3.11, 3.12

**主要ステップ**:
1. **環境セットアップ**
   - UV（高速Pythonパッケージマネージャー）とPythonのセットアップ
   - `uv.lock`のキャッシュと検証
   - 依存関係のインストール（`uv sync --project api --dev`）

2. **コード品質チェック**
   - Pyreflyチェック（実験的、失敗は許容）
   - Dify設定テスト

3. **ミドルウェア起動**
   - PostgreSQL（データベース）
   - Redis（キャッシュ）
   - Sandbox（コード実行環境）
   - SSRF Proxy（セキュリティプロキシ）

4. **テスト実行**
   - Workflowテスト（`dev/pytest/pytest_workflow.sh`）
   - Toolテスト（`dev/pytest/pytest_tools.sh`）
   - TestContainersテスト（`dev/pytest/pytest_testcontainers.sh`）
   - ユニットテスト（`dev/pytest/pytest_unit_tests.sh`）

5. **カバレッジレポート**
   - カバレッジ率の抽出と表示
   - GitHub Step Summaryへのレポート出力

**並行制御**: `api-tests-${{ github.head_ref || github.run_id }}`

---

### 3. Webテスト (`web-tests.yml`)

**トリガー**: `main-ci.yml`から呼び出し（workflow_call）

**主要ステップ**:
1. **変更ファイル検知**
   - `web/**`配下の変更を検出
   - 変更がない場合はスキップ

2. **環境セットアップ**
   - pnpmのインストール
   - Node.js 22のセットアップ
   - 依存関係のインストール（`pnpm install --frozen-lockfile`）

3. **テスト実行**
   - i18nタイプの同期チェック（`pnpm run check:i18n-types`）
   - ユニットテスト（`pnpm test`）

**並行制御**: `web-tests-${{ github.head_ref || github.run_id }}`

---

### 4. スタイルチェック (`style.yml`)

**トリガー**: `main-ci.yml`から呼び出し（workflow_call）

**ジョブ構成**: 4つの並列ジョブ

#### 4.1 Pythonスタイル (`python-style`)

**主要チェック**:
- **Import Linter**: インポート構造の検証
- **Basedpyright**: 型チェック（厳格モード）
- **Mypy**: 追加の型チェック（`tests/`と`migrations/`を除外）
- **Dotenv Linter**: 環境変数ファイルの検証

**変更検知**: `api/**`, `.github/workflows/style.yml`

#### 4.2 Webスタイル (`web-style`)

**主要チェック**:
- **ESLint**: JavaScriptコードのリント（`pnpm run lint`）
- **TypeScript型チェック**: `pnpm run type-check`

**変更検知**: `web/**`

#### 4.3 Docker Compose テンプレート (`docker-compose-template`)

**検証内容**:
- `docker/generate_docker_compose`スクリプトの実行
- 生成されたファイルと既存ファイルの差分チェック

**変更検知**: 
- `docker/generate_docker_compose`
- `docker/.env.example`
- `docker/docker-compose-template.yaml`
- `docker/docker-compose.yaml`

#### 4.4 SuperLinter (`superlinter`)

**検証対象**:
- **Bash**: シェルスクリプトの構文チェック（warning level）
- **Dockerfile**: Hadolintによるベストプラクティスチェック
- **EditorConfig**: コーディングスタイルの一貫性チェック
- **YAML**: YAML構文の検証
- **XML**: XML構文の検証

**変更検知**: `**.sh`, `**.yaml`, `**.yml`, `**Dockerfile`, `dev/**`, `.editorconfig`

**並行制御**: `style-${{ github.head_ref || github.run_id }}`

---

### 5. VDBテスト (`vdb-tests.yml`)

**トリガー**: `main-ci.yml`から呼び出し（workflow_call）

**Python バージョンマトリクス**: 3.11, 3.12

**テスト対象のベクトルデータベース**:
- TiDB + TiFlash
- Weaviate
- Qdrant
- PGVector
- Milvus
- PgVecto-RS
- Chroma
- MyScale
- ElasticSearch
- Couchbase
- OceanBase

**主要ステップ**:
1. **ディスク容量確保**
   - .NET、Haskell、ツールキャッシュの削除

2. **環境セットアップ**
   - UV/Pythonのセットアップとキャッシング
   - 依存関係のインストール

3. **VDBサービス起動**
   - TiDB/TiFlashの起動（専用compose）
   - 他のVDBサービスの起動（メインcompose）

4. **準備状態チェック**
   - TiFlashの準備完了を確認

5. **VDBテスト実行**
   - `dev/pytest/pytest_vdb.sh`

**並行制御**: `vdb-tests-${{ github.head_ref || github.run_id }}`

---

### 6. DBマイグレーションテスト (`db-migration-test.yml`)

**トリガー**: `main-ci.yml`から呼び出し（workflow_call）

**主要ステップ**:
1. **オフラインマイグレーション検証**
   - アップグレード: `flask db upgrade 'base:head' --sql`
   - ダウングレード: `flask db downgrade 'head:base' --sql`
   - SQLファイルの生成確認（実際のDB接続なし）

2. **ミドルウェア起動**
   - PostgreSQL
   - Redis

3. **実際のマイグレーション実行**
   - `flask upgrade-db`

**並行制御**: `db-migration-test-${{ github.ref }}`

---

### 7. Dockerビルド (`docker-build.yml`)

**トリガー**: PRでの`api/Dockerfile`または`web/Dockerfile`の変更

**ビルドマトリクス**:
- API: linux/amd64, linux/arm64
- Web: linux/amd64, linux/arm64

**特徴**:
- プッシュなし（ビルド検証のみ）
- GitHub Actionsキャッシュの活用

**並行制御**: `docker-build-${{ github.head_ref || github.run_id }}`

---

### 8. ビルド＆プッシュ (`build-push.yml`)

**トリガー**: 
- mainブランチへのpush
- `deploy/**`, `build/**`, `hotfix/**`ブランチへのpush
- `release/e-*`ブランチへのpush
- タグのpush

**実行条件**: `github.repository == 'langgenius/dify'`

**2段階プロセス**:

#### ステージ1: ビルドジョブ
- **ランナー**: AMD64は`ubuntu-latest`、ARM64は`arm64_runner`
- **マトリクス**: API/Web × AMD64/ARM64 = 4ジョブ
- **出力**: Digest（イメージハッシュ）をアーティファクトとしてアップロード
- **キャッシュ**: サービス別のGHAキャッシュ

#### ステージ2: マニフェスト作成
- **ダイジェストの統合**: 各アーキテクチャのダイジェストをダウンロード
- **マルチアーチマニフェスト作成**: AMD64/ARM64を統合したマニフェストリストを作成
- **タグ戦略**:
  - `latest`: リリースタグの場合のみ（`-`を含まない）
  - ブランチ名: プッシュされたブランチ名
  - SHA: コミットSHA（long形式）
  - タグ名: Gitタグがプッシュされた場合

**並行制御**: `build-push-${{ github.head_ref || github.run_id }}`

---

### 9. デプロイメント

#### 9.1 開発環境デプロイ (`deploy-dev.yml`)

**トリガー**: `build-push.yml`ワークフローが`deploy/dev`ブランチで成功した後

**実行条件**: ビルドワークフローが成功し、`deploy/dev`ブランチの場合

**デプロイ方法**:
- SSH接続によるリモートサーバーへのデプロイ
- スクリプトは`secrets.SSH_SCRIPT`または`vars.SSH_SCRIPT`に定義

#### 9.2 エンタープライズデプロイ (`deploy-enterprise.yml`)
（内容は読み取っていませんが、同様の構造と推測）

#### 9.3 開発環境デプロイトリガー (`deploy-trigger-dev.yml`)
（内容は読み取っていませんが、手動トリガーまたは他のトリガー機構と推測）

---

### 10. SDK テスト (`tool-test-sdks.yaml`)

**トリガー**: `sdks/**`配下の変更を含むPR

**テスト対象**: Node.js SDK

**Node.js バージョンマトリクス**: 16, 18, 20, 22

**テスト手順**:
1. Node.jsのセットアップ
2. 依存関係のインストール（`pnpm install --frozen-lockfile`）
3. テスト実行（`pnpm test`）

**並行制御**: `sdk-tests-${{ github.head_ref || github.run_id }}`

---

### 11. 自動修正 (`autofix.yml`)

**トリガー**: mainブランチへのPR

**実行条件**: `github.repository == 'langgenius/dify'`

**自動修正内容**:

#### Pythonコード:
- **Ruffフォーマット**: コードフォーマット
- **Rufflint修正**: リントエラーの自動修正
- **ast-grep変換**:
  - `.filter()`を`.where()`に変換（SQLAlchemy 2.0スタイル）
  - `db.Column`を`mapped_column`に変換（SQLAlchemy 2.0スタイル）
  - `Optional[T]`を`T | None`に変換（Python 3.10+スタイル）
  - 前方参照の修正（文字列型アノテーションの処理）

#### Markdown:
- **mdformat**: Markdownファイルのフォーマット

#### TypeScript/JavaScript:
- **oxlint**: 高速リンターによる自動修正

**自動コミット**: `autofix-ci/action`により自動的にPRに変更をコミット

---

### 12. Staleイシュー管理 (`stale.yml`)

**トリガー**: 毎日午前3時（UTC）

**動作**:
- **イシュー**: 15日間活動なし → staleラベル付与 → 3日後にクローズ
- **PR**: 同様

**対象ラベル**: `duplicate`, `question`, `invalid`, `wontfix`, `no-issue-activity`, `no-pr-activity`, `enhancement`, `cant-reproduce`, `help-wanted`

---

### 13. i18n翻訳 (`translate-i18n-base-on-english.yml`)

**トリガー**: （読み取っていませんが、英語ベースファイルの変更時と推測）

**目的**: 英語の翻訳ファイルの変更を他言語に自動反映

---

## Dependabot設定

**設定ファイル**: `.github/dependabot.yml`

**監視対象**:
1. **Web依存関係** (`/web`ディレクトリ)
   - パッケージエコシステム: npm
   - スケジュール: 毎週
   - 同時PR上限: 2

2. **API依存関係** (`/api`ディレクトリ)
   - パッケージエコシステム: uv
   - スケジュール: 毎週
   - 同時PR上限: 2

---

## Makefile による開発者ワークフロー

Makefileは、開発者のローカル環境での作業を支援するコマンドを提供します。

### 開発環境セットアップ

```bash
# 完全セットアップ（Docker + Web + API）
make dev-setup

# 個別セットアップ
make prepare-docker  # Dockerミドルウェアのみ
make prepare-web     # Web環境のみ
make prepare-api     # API環境のみ

# クリーンアップ
make dev-clean       # Dockerコンテナとボリュームの削除
```

### コード品質コマンド

```bash
# フォーマット
make format      # Ruffによるコードフォーマット

# チェック
make check       # Ruffによるコードチェック（修正なし）

# リント（フォーマット + 修正 + インポート検証）
make lint        # CI前に必須

# 型チェック
make type-check  # Basedpyrightによる型チェック（CI前に必須）
```

### Dockerイメージ管理

```bash
# ビルド
make build-web   # Webイメージのビルド
make build-api   # APIイメージのビルド
make build-all   # 全イメージのビルド

# プッシュ
make push-web    # Webイメージのプッシュ
make push-api    # APIイメージのプッシュ
make push-all    # 全イメージのプッシュ

# ビルド＆プッシュ
make build-push-web     # Webイメージのビルド＆プッシュ
make build-push-api     # APIイメージのビルド＆プッシュ
make build-push-all     # 全イメージのビルド＆プッシュ
```

**デフォルトレジストリ**: `langgenius` (環境変数で変更可能)

---

## CI/CD ベストプラクティス

### 1. 並行実行制御

各ワークフローは`concurrency`設定により、同一ブランチ/PRでの重複実行を防止：
- `cancel-in-progress: true`により、新しい実行が古い実行をキャンセル
- リソースの効率的な使用とフィードバックの高速化

### 2. 変更検知による最適化

`main-ci.yml`は`dorny/paths-filter`を使用し、変更されたファイルに基づいて必要なテストのみを実行：
- 不要なテストをスキップしてCI時間を短縮
- リソースの効率的な使用

### 3. マトリクス戦略

複数のPythonバージョン（3.11, 3.12）およびNode.jsバージョン（16, 18, 20, 22）でテスト：
- 互換性の保証
- 並列実行による時間短縮

### 4. キャッシング戦略

- **UV**: `enable-cache: true`とロックファイルベースのキャッシュ
- **pnpm**: Node.jsセットアップアクションの統合キャッシュ
- **Docker**: GitHub Actionsキャッシュ（`type=gha`）

### 5. マルチアーキテクチャサポート

AMD64とARM64の両方をサポート：
- ARM64は専用ランナー（`arm64_runner`）を使用
- マニフェストリストによる透過的なアーキテクチャ選択

### 6. テスト隔離

各テストスイートは独立したサービスセットを使用：
- TestContainers: コンテナベースのテスト隔離
- Docker Compose: ミドルウェアの一貫した環境

### 7. セキュリティ

- **persist-credentials: false**: チェックアウト時の認証情報の永続化を無効化
- **SSRF Proxy**: サーバーサイドリクエストフォージェリ対策
- **シークレット管理**: GitHub Secretsによる機密情報の保護

---

## CI実行フロー図

```
PR作成/更新
    |
    v
main-ci.yml
    |
    +-- check-changes
    |       |
    |       +-- paths-filter
    |               |
    |               v
    +-- 並列実行 --+-- api-tests (api変更時)
    |               |       |
    |               |       +-- pytest (3.11, 3.12)
    |               |       +-- カバレッジレポート
    |               |
    |               +-- web-tests (web変更時)
    |               |       |
    |               |       +-- pnpm test
    |               |       +-- i18n型チェック
    |               |
    |               +-- style-check (常時)
    |               |       |
    |               |       +-- python-style
    |               |       +-- web-style
    |               |       +-- docker-compose-template
    |               |       +-- superlinter
    |               |
    |               +-- vdb-tests (VDB変更時)
    |               |       |
    |               |       +-- VDBサービス起動
    |               |       +-- pytest (3.11, 3.12)
    |               |
    |               +-- db-migration-test (migration変更時)
    |                       |
    |                       +-- オフライン検証
    |                       +-- オンライン実行
    |
    v
全ジョブ成功
    |
    v
マージ可能
```

---

## デプロイメントフロー

```
main/deploy/*/release/*ブランチへのプッシュまたはタグ
    |
    v
build-push.yml
    |
    +-- ビルドステージ（並列4ジョブ）
    |       |
    |       +-- API AMD64
    |       +-- API ARM64
    |       +-- Web AMD64
    |       +-- Web ARM64
    |       |
    |       v
    +-- マニフェスト作成（並列2ジョブ）
            |
            +-- API マニフェスト
            +-- Web マニフェスト
            |
            v
        Docker Hub へプッシュ
            |
            v
    (deploy/devブランチの場合)
            |
            v
        deploy-dev.yml
            |
            v
        SSH経由でデプロイ
```

---

## ローカル開発からCI/CDへの統合

### ローカル開発ワークフロー

1. **環境セットアップ**
   ```bash
   make dev-setup
   ```

2. **開発作業**
   - コード変更
   - ローカルテスト実行

3. **提出前チェック（必須）**
   ```bash
   # バックエンド
   make lint        # フォーマット + lint
   make type-check  # 型チェック
   uv run --project api --dev dev/pytest/pytest_unit_tests.sh
   
   # フロントエンド
   cd web
   pnpm lint
   pnpm lint:fix
   pnpm test
   ```

4. **コミット＆プッシュ**
   ```bash
   git add .
   git commit -m "feat: ..."
   git push origin feature-branch
   ```

5. **PR作成**
   - GitHub UIでPR作成
   - CIが自動実行

6. **CI結果確認**
   - 全チェック通過を確認
   - 必要に応じて修正

7. **Autofix適用**
   - `autofix.yml`が自動的に軽微な問題を修正
   - 自動コミットを確認

8. **レビュー＆マージ**
   - コードレビュー
   - 承認後マージ

### CI失敗時のデバッグ

1. **ログ確認**: GitHub ActionsのUIでログを確認
2. **ローカル再現**: 同じコマンドをローカルで実行
3. **修正**: 問題を修正
4. **再プッシュ**: 変更をプッシュしてCIを再実行

---

## 環境変数とシークレット

### 必須シークレット（デプロイ用）

- `DOCKERHUB_USER`: Docker Hubユーザー名
- `DOCKERHUB_TOKEN`: Docker Hubアクセストークン
- `SSH_HOST`: デプロイ先サーバーホスト
- `SSH_USER`: SSHユーザー名
- `SSH_PRIVATE_KEY`: SSH秘密鍵
- `SSH_SCRIPT` または `vars.SSH_SCRIPT`: デプロイスクリプト

### カスタマイズ可能な変数

- `DIFY_WEB_IMAGE_NAME`: Webイメージ名（デフォルト: `langgenius/dify-web`）
- `DIFY_API_IMAGE_NAME`: APIイメージ名（デフォルト: `langgenius/dify-api`）

---

## パフォーマンスとコスト最適化

### 実行時間の最適化

1. **並行実行**: 独立したジョブは並列実行
2. **変更検知**: 不要なテストをスキップ
3. **キャッシング**: 依存関係とビルドアーティファクトをキャッシュ
4. **早期終了**: `cancel-in-progress`により古い実行をキャンセル

### コスト最適化

1. **マトリクス戦略**: 必要最小限のバージョン数でテスト
2. **条件付き実行**: 変更されたコンポーネントのみテスト
3. **リソース管理**: 
   - VDBテストでディスク容量を確保
   - 不要なツールキャッシュを削除

---

## メンテナンスとモニタリング

### 定期的なメンテナンス

1. **Dependabot**: 毎週依存関係を更新
2. **Stale管理**: 非アクティブなイシュー/PRを自動クローズ

### モニタリング

- **カバレッジレポート**: 各APIテスト実行でカバレッジを表示
- **GitHub Status Checks**: PRのマージ条件として必須チェックを設定
- **通知**: GitHub通知またはSlack統合（オプション）

---

## トラブルシューティング

### よくある問題

#### 1. UVロックファイルの不整合
```bash
# ローカルで修正
uv lock --project api

# コミット
git add api/uv.lock
git commit -m "fix: update uv.lock"
```

#### 2. 型チェックエラー
```bash
# ローカルで確認
make type-check

# または
dev/basedpyright-check
```

#### 3. Docker Composeテンプレート不一致
```bash
# 再生成
cd docker
./generate_docker_compose

# 差分確認
git diff docker-compose.yaml
```

#### 4. VDBテストのタイムアウト
- ディスク容量を確認
- Dockerサービスのログを確認
- 必要に応じてタイムアウト値を調整

---

## 今後の改善案

### 短期（実装済みまたは進行中）
- [x] マルチアーキテクチャビルド（AMD64/ARM64）
- [x] 包括的なテストカバレッジ
- [x] 自動修正ワークフロー

### 中期（検討中）
- [ ] E2Eテストの追加
- [ ] パフォーマンステスト
- [ ] セキュリティスキャン（Trivy, Snyk）
- [ ] カバレッジ閾値の強制

### 長期（将来構想）
- [ ] カナリアデプロイメント
- [ ] ブルー/グリーンデプロイメント
- [ ] 自動ロールバック
- [ ] マルチクラウドデプロイメント

---

## 参考リンク

- [GitHub Actions ドキュメント](https://docs.github.com/en/actions)
- [UV ドキュメント](https://github.com/astral-sh/uv)
- [Docker Buildx](https://docs.docker.com/buildx/working-with-buildx/)
- [Ruff](https://docs.astral.sh/ruff/)
- [Basedpyright](https://github.com/DetachHead/basedpyright)

---

**最終更新**: 2025年11月5日
**バージョン**: 1.0.0
