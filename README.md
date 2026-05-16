# Copier Template Demo

Copier テンプレート管理の検証用リポジトリ。テンプレート更新時に下流プロジェクトへ自動で PR を作成する仕組みを含む。

## 構成

```
copier-tmpl-demo/
├── copier.yml                           # テンプレート設定・質問定義
├── template/                            # テンプレート本体 (_subdirectory)
│   ├── README.md.jinja
│   ├── pyproject.toml.jinja
│   ├── Dockerfile.jinja
│   ├── {{_copier_conf.answers_file}}.jinja   # copier update に必須
│   └── .github/workflows/ci.yml.jinja
└── .github/
    ├── downstream-repos.json            # 下流リポ一覧（一元管理）
    └── workflows/
        ├── notify-downstream.yml        # タグ push → 下流へ dispatch
        └── compliance-check.yml         # 下流の準拠状況チェック
```

## 導入手順

### 1. テンプレートからプロジェクトを生成

```bash
copier copy --defaults \
  --data project_name=my-app \
  --data author=your-name \
  --data use_docker=true \
  --data python_version=3.12 \
  --vcs-ref v4.1.0 \
  gh:1210yuichi0/copier-tmpl-demo . --trust
```

### 2. テンプレート更新を手動で適用

```bash
copier update --defaults --trust
```

### 3. 自動更新の設定（Push 型）

下流プロジェクトリポに以下のワークフローを配置:

```yaml
# .github/workflows/copier-update.yml
name: Auto-update from Copier template
on:
  repository_dispatch:
    types: [template-updated]
  schedule:
    - cron: "0 9 * * 1"
  workflow_dispatch:
```

詳細は [copier-proj-demo](https://github.com/1210yuichi0/copier-proj-demo) を参照。

### 4. Secret の設定

| Secret | 設定先 | スコープ | 用途 |
|--------|--------|---------|------|
| `DOWNSTREAM_PAT` | テンプレートリポ | `repo` | cross-repo dispatch 送信 |
| `CR_PAT` | プロジェクトリポ | `repo`, `workflow` | workflow ファイルを含む PR ブランチの push |

## 自動化フロー

```
テンプレートリポでタグ push (vX.Y.Z)
  → notify-downstream.yml が起動
  → downstream-repos.json の各リポに repository_dispatch 送信
  → プロジェクトリポの copier-update.yml がトリガー
  → copier update 実行 → 差分があれば PR 自動作成
```

## Compliance Check

テンプレートリポ側から下流リポの `.copier-answers.yml` を読み取り、最新バージョンとの乖離を検出:

- 週次スケジュール or 手動実行
- Outdated なリポがあれば Issue を自動作成
- 全リポが最新なら Issue を自動クローズ

## 検証結果

### バージョン履歴

| Tag | 変更内容 |
|-----|---------|
| v1.0.0 | 初期テンプレート |
| v1.1.0 | `answers_file` テンプレート追加 |
| v2.0.0 | CI ワークフロー・README セクション追加 |
| v3.0.0 | GHA 自動化ワークフロー統合 |
| v4.0.0 | `pyproject.toml` に dev 依存関係追加 |
| v4.1.0 | `project.authors` の TOML 構文修正 |

### GHA 実動検証

| テスト | 操作 | 結果 |
|--------|------|------|
| compliance-check | 手動実行 | Outdated 検出 → Issue 自動作成 |
| copier-update | 手動実行 | PR 自動作成（v1.1.0 → v3.0.0） |
| PR マージ | main に反映 | copier-answers.yml 更新 |
| compliance-check 再実行 | 準拠確認 | Issue 自動クローズ |
| フルチェーン | v4.0.0 タグ push | dispatch → copier-update → PR #2 自動作成 |

### ハマりポイント

1. **`{{_copier_conf.answers_file}}.jinja` が必須** - ないと `copier update` が動かない
2. **`_subdirectory` を使う** - ルート直下だと `.git/` がコピーされる
3. **ファイル名に Jinja2 条件分岐は使えない** - `_exclude` で制御
4. **`GITHUB_TOKEN` では workflow ファイルを push できない** - `CR_PAT` が必要
5. **cross-repo dispatch には PAT が必要** - `DOWNSTREAM_PAT` を設定
6. **`project.authors` は TOML 配列テーブル** - `[[project.authors]]` が正しい構文

## 参考

- [Copier 公式ドキュメント](https://copier.readthedocs.io/)
- [Cookiecutter vs Copier 比較](https://zenn.dev/killy/articles/fc8e0803a295d5)
- [Renovate Copier マネージャー](https://docs.renovatebot.com/modules/manager/copier/)
