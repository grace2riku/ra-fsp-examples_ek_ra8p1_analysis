# Claude Code 解析実行指示

## 概要

このファイルは、Claude Codeで解析を実行する際の指示事項をまとめたものです。

## 解析の実行方法

### 基本的な使い方

```
# 特定のPhaseを解析
doc/analysis_request/phase01_file_structure.md を読んで解析を実行してください

# 複数Phaseを連続解析
Phase 1 から Phase 3 まで順番に解析を実行してください

# 全Phase解析
全てのPhaseを順番に解析し、総合レポートを作成してください
```

### Phase指定の簡略形式

以下の形式でも解析を開始できます：

```
Phase 1 を解析して
phase01 を実行
ファイル構造を解析して
```

## 解析実行時のルール

### 1. 解析要求ファイルの読み込み

- `doc/analysis_request/` ディレクトリから該当するPhaseファイルを読み込む
- ファイル内の「解析要求事項」セクションに従って解析を実施

### 2. 解析対象ディレクトリ

```
quickstart_ek_ra8p1_ep/
├── e2studio/          # プロジェクト設定
├── src/               # アプリケーションコード
├── ra/                # FSPドライバ
├── ra_gen/            # FSP自動生成コード
├── ra_cfg/            # FSP設定
└── script/            # リンカスクリプト等
```

### 3. 解析レポートの出力

- 出力先: `doc/analysis_report/`
- ファイル名: `phase{番号}_{内容}_report.md`
- 形式: Markdown

例：
```
doc/analysis_report/phase01_file_structure_report.md
doc/analysis_report/phase02_build_system_report.md
```

### 4. レポートの構成

各レポートは以下の構成で作成：

```markdown
# Phase X: {タイトル} - 解析レポート

## 解析日時
YYYY-MM-DD HH:MM

## 概要
{解析結果の要約}

## 詳細解析結果
{解析要求事項に対応した詳細な結果}

## 発見事項
{解析中に見つかった重要な情報}

## 関連ファイル
{解析で参照したファイル一覧}

## 次のPhaseへの申し送り
{後続の解析に必要な情報}
```

## 解析のヒント

### ファイル検索

```bash
# 特定のパターンを含むファイルを検索
find . -name "*.c" | xargs grep -l "pattern"

# ヘッダファイルを検索
find . -name "*.h" -path "*/src/*"
```

### 設定ファイルの解析

- `configuration.xml` - FSPの設定（クロック、ピン、周辺機能）
- `.cproject` - コンパイラオプション
- `*.ld` - リンカスクリプト

### コード構造の理解

1. `main.c` からエントリポイントを特定
2. `hal_data.c` から使用している周辺機能を確認
3. `vector_data.c` から割込み構成を確認

## トラブルシューティング

### 解析が進まない場合

1. 対象ファイルの存在確認
2. ファイルパスの確認
3. 解析要求の範囲を絞る

### 情報が見つからない場合

1. 関連するキーワードでgrep検索
2. FSPドキュメントを参照
3. 「不明」として記録し、推測を併記

## 総合レポートの作成

全Phaseの解析完了後、以下のプロンプトで総合レポートを作成：

```
全Phaseの解析結果を統合した総合レポートを作成してください。
出力ファイル: doc/analysis_report/final_report.md
```
