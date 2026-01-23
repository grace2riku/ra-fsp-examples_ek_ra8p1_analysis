# EK-RA8P1 Quickstart プロジェクト解析ドキュメント

## 概要

このディレクトリには、EK-RA8P1 Quickstartプロジェクトの解析に使用する
要求事項と解析レポートが格納されています。

## ディレクトリ構成

```
doc/
├── README.md                    # このファイル
├── CLAUDE_INSTRUCTIONS.md       # Claude Code用の解析実行指示
├── analysis_request/            # 解析要求事項ファイル
│   ├── phase01_file_structure.md
│   ├── phase02_build_system.md
│   ├── phase03_compile_options.md
│   ├── phase04_os_identification.md
│   ├── phase05_startup_sequence.md
│   ├── phase06_memory_map.md
│   ├── phase07_hardware_settings.md
│   ├── phase08_layer_classification.md
│   ├── phase09_interrupt_analysis.md
│   ├── phase10_lcd_display.md
│   ├── phase11_touch_input.md
│   ├── phase12_screen_transition.md
│   ├── phase13_camera_display.md
│   └── phase14_overlay_display.md
└── analysis_report/             # 解析レポート出力先
    └── (解析完了後にレポートが生成されます)
```

## Phase一覧

| Phase | カテゴリ | 内容 | ファイル |
|-------|----------|------|----------|
| 1 | 環境・構成 | ファイル構造の把握 | phase01_file_structure.md |
| 2 | 環境・構成 | ビルドシステム・環境構成 | phase02_build_system.md |
| 3 | 環境・構成 | コンパイル・リンクオプション詳細 | phase03_compile_options.md |
| 4 | 基盤 | OS特定 | phase04_os_identification.md |
| 5 | 基盤 | 起動シーケンス解析 | phase05_startup_sequence.md |
| 6 | 基盤 | メモリマップ解析 | phase06_memory_map.md |
| 7 | 基盤 | ハードウェア設定解析 | phase07_hardware_settings.md |
| 8 | アーキテクチャ | レイヤー分類 | phase08_layer_classification.md |
| 9 | アーキテクチャ | 割込み解析 | phase09_interrupt_analysis.md |
| 10 | 機能 | LCD表示の仕組み | phase10_lcd_display.md |
| 11 | 機能 | タッチ入力解析 | phase11_touch_input.md |
| 12 | 機能 | 画面遷移の仕組み | phase12_screen_transition.md |
| 13 | 機能 | カメラ表示の仕組み | phase13_camera_display.md |
| 14 | 機能 | オーバーレイ表示の仕組み | phase14_overlay_display.md |

## 使用方法

### 1. 事前準備

```bash
# Git LFS のインストールと初期化
git lfs install

# longpath の有効化（Windows）
git config --global core.longpaths true

# リポジトリのクローン
git clone git@github.com:renesas/ra-fsp-examples.git
cd ra-fsp-examples/example_projects/ek_ra8p1/_quickstart/quickstart_ek_ra8p1_ep
```

### 2. 解析の実行

Claude Codeを起動し、解析を実行します：

```bash
# Claude Code を起動
claude

# 解析を実行（Phase番号を指定）
> doc/analysis_request/phase01_file_structure.md を読んで解析を実行してください
```

### 3. 解析レポートの確認

解析結果は `doc/analysis_report/` ディレクトリに出力されます。

## 解析の推奨順序

1. **環境・構成編** (Phase 1-3) を最初に実行
   - プロジェクト構造とビルド環境を把握

2. **基盤編** (Phase 4-7) を次に実行
   - OSと起動処理、メモリ構成を理解

3. **アーキテクチャ編** (Phase 8-9) を実行
   - ソフトウェア構造と割込みを把握

4. **機能編** (Phase 10-14) を最後に実行
   - 各機能の詳細を理解

## 注意事項

- 各Phaseの解析結果は相互に参照することがあります
- 前のPhaseの結果を踏まえて解析すると、より深い理解が得られます
- 解析レポートは随時更新される可能性があります
