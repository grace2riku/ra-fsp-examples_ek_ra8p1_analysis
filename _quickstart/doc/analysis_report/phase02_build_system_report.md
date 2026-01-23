# Phase 2: ビルドシステム・環境構成 - 解析レポート

## 解析日時
2026-01-23

## 概要

EK-RA8P1 クイックスタートプロジェクトのビルドシステムと開発環境を解析しました。
本プロジェクトは e2 studio IDE と GCC ARM Embedded ツールチェーンを使用し、FSP 6.3.0 をベースにしています。

---

## 1. ビルドシステムの構成

### 基本情報

| 項目 | 値 |
|------|-----|
| **IDE** | e2 studio 2025-12 |
| **ツールチェーン** | GCC ARM Embedded |
| **ツールチェーンバージョン** | 13.2.1.arm-13-7 |
| **FSPバージョン** | 6.3.0 |
| **ターゲットCPU** | Cortex-M85 (`-mcpu=cortex-m85+nopacbti`) |
| **浮動小数点** | Hardware FPU (`-mfloat-abi=hard`) |
| **RTOS** | AWS FreeRTOS |

### ツールチェーン詳細

```
コンパイラ: arm-none-eabi-gcc
C++コンパイラ: arm-none-eabi-g++
アセンブラ: arm-none-eabi-as
リンカ: arm-none-eabi-ld
アーカイバ: arm-none-eabi-ar
Hex変換: arm-none-eabi-objcopy
サイズ表示: arm-none-eabi-size
デバッガ: arm-none-eabi-gdb
```

### ビルドコマンド

- **ビルドシステム**: GNU Make (e2 studio Managed Build)
- **クリーンコマンド**: `rm -rf`
- **並列ビルド**: 有効（最適化された並列数）

---

## 2. プロジェクト設定ファイル

### 設定ファイル一覧

| ファイル | サイズ | 役割 |
|---------|--------|------|
| `.project` | 1.5KB | Eclipseプロジェクト基本設定 |
| `.cproject` | 188KB | C/C++ビルド設定（コンパイラオプション等） |
| `configuration.xml` | 306KB | ★ FSPコンフィグレーション |
| `ra_cfg.txt` | 214KB | FSP設定のテキスト出力 |

### .project の構成

```xml
<projectDescription>
    <name>quickstart_ek_ra8p1_ep</name>
    <buildSpec>
        <!-- FSPコード生成 -->
        <buildCommand>com.renesas.cdt.ddsc.contentgen.ddscBuilder</buildCommand>
        <!-- Makeビルド -->
        <buildCommand>org.eclipse.cdt.managedbuilder.core.genmakebuilder</buildCommand>
        <!-- スキャナ設定 -->
        <buildCommand>org.eclipse.cdt.managedbuilder.core.ScannerConfigBuilder</buildCommand>
        <!-- compile_commands.json生成 -->
        <buildCommand>com.renesas.cdt.managedbuild.jsoncdb.compilationdatabase.compilationDatabaseBuilder</buildCommand>
    </buildSpec>
    <natures>
        <nature>org.eclipse.cdt.core.cnature</nature>
        <nature>com.renesas.cdt.ddsc.contentgen.ddscNature</nature>
        <nature>com.renesas.cdt.ra.contentgen.raNature</nature>
    </natures>
</projectDescription>
```

### .cproject の主要設定

#### コンパイラオプション（Debug構成）

```
最適化レベル: -O0 (なし)
警告オプション:
  -Wall -Wextra -Wunused -Wuninitialized
  -Wmissing-declarations -Wconversion
  -Wpointer-arith -Wshadow -Wlogical-op
  -Waggregate-return -Wfloat-equal

追加オプション:
  -fmessage-length=0
  -fsigned-char
  -ffunction-sections
  -fdata-sections
  -fno-strict-aliasing
  -flax-vector-conversions
  --param=min-pagesize=0
```

#### プリプロセッサ定義

```c
_RENESAS_RA_           // Renesas RAファミリ識別
_RA_ORDINAL=1          // RAデバイス序数
_RA_CORE=CPU0          // 使用コア（CPU0 = Cortex-M85）
_RA_DEBUG_BUILD_       // デバッグビルド識別（Debug構成時）
```

#### インクルードパス

```
${ProjName}/src                          # アプリケーションソース
${ProjName}/ra/fsp/inc                   # FSP共通ヘッダ
${ProjName}/ra/fsp/inc/api               # FSP APIヘッダ
${ProjName}/ra/fsp/inc/instances         # FSPインスタンスヘッダ
${ProjName}/ra/fsp/src/rm_freertos_port  # FreeRTOSポート
${ProjName}/ra/aws/FreeRTOS/FreeRTOS/Source/include  # FreeRTOSヘッダ
${ProjName}/ra/arm/CMSIS_6/CMSIS/Core/Include        # CMSIS-Coreヘッダ
${ProjName}/ra_gen                       # FSP生成コード
${ProjName}/ra_cfg/fsp_cfg/bsp           # BSP設定
${ProjName}/ra_cfg/fsp_cfg               # FSP設定
${ProjName}/ra_cfg/aws                   # AWS設定
${ProjName}/ra/tes/dave2d/inc            # Dave2Dグラフィックス
```

#### リンカオプション

```
リンカスクリプト: fsp.ld
ライブラリパス: ${ProjName}/script
オプション:
  --gc-sections          # 未使用セクション削除
  --specs=nano.specs     # newlib-nano使用
```

### configuration.xml の構成

FSPコンフィグレーションの主要設定：

```xml
<raConfiguration version="11">
  <generalSettings>
    <option key="#Board#" value="board.ra8p1ek" />
    <option key="CPU" value="RA8P1" />
    <option key="Core" value="CPU0" />
    <option key="#TargetName#" value="R7KA8P1KFLCAC" />
    <option key="#TargetARCHITECTURE#" value="cortex-m85" />
    <option key="#RTOS#" value="rtos.awsfreertos" />
    <option key="#FSPVersion#" value="6.3.0" />
    <option key="#SELECTED_TOOLCHAIN#" value="gcc-arm-embedded" />
    <option key="#ToolchainVersion#" value="13.2.1.arm-13-7" />
  </generalSettings>
</raConfiguration>
```

---

## 3. ビルド手順

### e2 studio でのビルド方法（GUI操作）

#### 初回セットアップ

1. **プロジェクトのインポート**
   - File → Import → Existing Projects into Workspace
   - `quickstart_ek_ra8p1_ep/e2studio` を選択

2. **FSPコード生成**
   - `configuration.xml` をダブルクリック
   - FSP Configuration 画面で設定を確認
   - 「Generate Project Content」ボタンをクリック
   - `ra/`, `ra_gen/`, `ra_cfg/` ディレクトリが生成される

#### ビルド実行

| 操作 | 方法 |
|------|------|
| **フルビルド** | Project → Build Project (Ctrl+B) |
| **クリーンビルド** | Project → Clean → Build |
| **インクリメンタルビルド** | 自動（ファイル保存時） |
| **ビルド構成切替** | Project → Build Configurations → Set Active |

### コマンドラインでのビルド（Windows環境）

```batch
# e2 studioのビルドツールパスを設定
set PATH=%PATH%;C:\Renesas\e2_studio\eclipse\plugins\...\tools\bin

# プロジェクトディレクトリに移動
cd quickstart_ek_ra8p1_ep\e2studio

# ビルド実行
make -j all

# クリーン
make clean
```

### FSPコード生成の手順

1. e2 studioで `configuration.xml` を開く
2. 必要に応じてBSP、クロック、ピン、スタックを設定
3. 「Generate Project Content」を実行
4. 生成されるディレクトリ：
   - `ra/` - FSPドライバソース
   - `ra_gen/` - 自動生成コード（hal_data.c等）
   - `ra_cfg/` - 設定ヘッダファイル

### ビルド時の依存関係

```
configuration.xml
    ↓ [FSP Code Generator]
ra_gen/hal_data.c, ra_gen/vector_data.c, ...
ra_cfg/fsp_cfg/*.h
    ↓ [GNU Make]
src/*.c + ra/**/*.c
    ↓ [arm-none-eabi-gcc]
Debug/*.o (オブジェクトファイル)
    ↓ [arm-none-eabi-ld]
Debug/quickstart_ek_ra8p1_ep.elf
    ↓ [arm-none-eabi-objcopy]
Debug/quickstart_ek_ra8p1_ep.hex
```

---

## 4. 出力ファイル

### ビルド出力ファイル一覧

| ファイル種別 | パス | 用途 |
|-------------|------|------|
| **ELFファイル** | `Debug/quickstart_ek_ra8p1_ep.elf` | デバッグ用実行ファイル |
| **HEXファイル** | `Debug/quickstart_ek_ra8p1_ep.hex` | フラッシュ書き込み用 |
| **MAPファイル** | `Debug/quickstart_ek_ra8p1_ep.map` | メモリ配置確認用 |
| **オブジェクトファイル** | `Debug/**/*.o` | 中間ファイル |
| **依存関係ファイル** | `Debug/**/*.d` | 依存関係追跡用 |
| **リストファイル** | `Debug/quickstart_ek_ra8p1_ep.lst` | 逆アセンブルリスト |

### ビルド済みバイナリ（同梱）

```
e2studio/quickstart_ek_ra8p1_ep.hex  (15.8MB)
```

このファイルは事前にビルドされたバイナリで、ビルド環境なしでも書き込みが可能です。

---

## 5. デバッグ設定

### デバッガ情報

| 項目 | 設定値 |
|------|--------|
| **デバッガ種類** | SEGGER J-Link ARM |
| **接続方式** | SWD |
| **インターフェース速度** | 4000 kHz |
| **ターゲットデバイス** | R7KA8P1KF_CPU0 |
| **GDBサーバーポート** | 61234 |

### デバッグ構成ファイル

| ファイル | 役割 |
|---------|------|
| `quickstart_ek_ra8p1_ep Debug_Flat.launch` | e2 studioデバッグ起動設定 |
| `quickstart_ek_ra8p1_ep Debug_Flat.jlink` | J-Link固有設定 |
| `script/RA8x1_Reset_OSPI.JLinkScript` | OSPIリセット用J-Linkスクリプト |

### J-Link設定（.jlink）

```ini
[BREAKPOINTS]
EnableFlashBP = 1      # フラッシュブレークポイント有効

[FLASH]
SkipProgOnCRCMatch = 1 # CRC一致時はプログラムスキップ
VerifyDownload = 1     # ダウンロード後検証
AllowCaching = 1       # キャッシュ有効

[GENERAL]
WorkRAMSize = 0x10000  # 64KB
WorkRAMAddr = 0x22060000
```

### デバッグ開始手順

1. **ハードウェア接続**
   - J-LinkをPCに接続
   - EK-RA8P1ボードのDEBUG1 (J10)にJ-Linkを接続

2. **デバッグ起動**
   - Run → Debug Configurations
   - `quickstart_ek_ra8p1_ep Debug_Flat` を選択
   - Debug ボタンをクリック

3. **初期停止位置**
   - `main()` 関数で停止（設定による）

### デバッグ機能

| 機能 | 対応状況 |
|------|----------|
| ブレークポイント | ○（フラッシュBP対応） |
| ステップ実行 | ○ |
| 変数ウォッチ | ○ |
| メモリ表示 | ○ |
| レジスタ表示 | ○（SVDファイル使用） |
| RTOS認識 | ○（FreeRTOS対応） |
| SWO出力 | △（設定により可能） |

---

## 6. デュアルコア構成

### RA8P1のコア構成

| コア | 種類 | 用途 |
|------|------|------|
| CPU0 | Cortex-M85 | メインアプリケーション |
| CPU1 | Cortex-M33 | (本プロジェクトでは未使用) |

### 本プロジェクトの構成

- **シングルコア動作**: CPU0（Cortex-M85）のみ使用
- **ビルド構成名**: `Debug_Flat`（Flat = TrustZone無効）
- **TrustZone**: 無効 (`-mcmse` オプションなし)

### デュアルコアデバッグ時の設定

（本プロジェクトでは使用しないが参考情報）

```xml
<!-- .launchファイルの設定例 -->
<stringAttribute key="com.renesas.cdt.core.targetDevice"
                 value="R7KA8P1KF_CPU0"/>
<!-- CPU1の場合は R7KA8P1KF_CPU1 -->
```

---

## 発見事項

### 1. ビルドシステムの特徴

- **Managed Build**: e2 studioが自動的にMakefileを生成・管理
- **並列ビルド対応**: 最適化された並列ビルドが有効
- **compile_commands.json**: clangd等のLSP対応

### 2. 最適化設定

| 構成 | 最適化レベル | 備考 |
|------|-------------|------|
| Debug | -O0 (なし) | デバッグ情報あり |
| Release | (未確認) | 最適化有効化推奨 |

### 3. 特殊なコンパイラオプション

```
-flax-vector-conversions  # ベクタ型の暗黙変換を許可
--param=min-pagesize=0    # ページサイズ最小化
-Wno-stringop-overflow    # 文字列操作警告抑制
-Wno-format-truncation    # フォーマット切り詰め警告抑制
```

これらはFSPライブラリやDave2Dとの互換性のために必要です。

### 4. フォルダ固有のコンパイル設定

`.cproject` では以下のフォルダに対して警告を抑制：

- `ra/tes/` (Dave2D)
- `ra/aws/` (FreeRTOS)
- `ra/arm/` (CMSIS)

---

## ビルドフロー図

```
┌─────────────────────────────────────────────────────────────┐
│                    FSP Configuration                         │
│  configuration.xml                                           │
│    ├── BSP設定                                               │
│    ├── クロック設定                                          │
│    ├── ピン設定                                              │
│    └── スタック設定（FreeRTOS, ドライバ等）                   │
└──────────────────────────┬──────────────────────────────────┘
                           │ Generate Project Content
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    生成コード                                │
│  ra_gen/                      ra_cfg/                        │
│    ├── hal_data.c               ├── fsp_cfg/                 │
│    ├── vector_data.c            │     ├── bsp/               │
│    ├── main.c                   │     └── r_*_cfg.h          │
│    └── common_data.c            └── aws/                     │
│                                       └── FreeRTOSConfig.h   │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    コンパイル                                │
│                                                              │
│  src/*.c ─────┐                                              │
│  ra_gen/*.c ──┼──→ arm-none-eabi-gcc ──→ Debug/*.o          │
│  ra/**/*.c ───┘                                              │
│                                                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    リンク                                    │
│                                                              │
│  Debug/*.o ──┐                                               │
│  fsp.ld ─────┼──→ arm-none-eabi-ld ──→ quickstart_ek_ra8p1_ep.elf
│  ra/tes/*.a ─┘                                               │
│                                                              │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    後処理                                    │
│                                                              │
│  .elf ──→ objcopy ──→ .hex (Intel HEX)                      │
│       └─→ objdump ──→ .lst (逆アセンブル)                   │
│       └─→ size    ──→ サイズ情報表示                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 設定ファイル関係図

```
┌────────────────────────────────────────────────────────────────┐
│                        e2 studio                               │
└────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  .project   │    │    .cproject    │    │ configuration.  │
│             │    │                 │    │      xml        │
│ プロジェクト │    │ ビルド設定      │    │                 │
│ 基本設定    │    │ コンパイラ      │    │ FSP設定         │
│ ビルダー    │    │ リンカ          │    │ クロック        │
│ Nature      │    │ インクルード    │    │ ピン            │
└─────────────┘    └─────────────────┘    │ 周辺機能        │
                            │              └─────────────────┘
                            │                       │
                            ▼                       ▼
                   ┌─────────────────┐    ┌─────────────────┐
                   │   Makefile      │    │  ra_gen/        │
                   │   (自動生成)     │    │  ra_cfg/        │
                   └─────────────────┘    │  ra/            │
                            │              └─────────────────┘
                            │                       │
                            └───────────┬───────────┘
                                        ▼
                               ┌─────────────────┐
                               │ ビルド成果物     │
                               │ .elf, .hex, .map│
                               └─────────────────┘
```

---

## クイックリファレンス

### よく使うビルド操作

| 操作 | ショートカット / 方法 |
|------|----------------------|
| ビルド | Ctrl+B |
| クリーン | Project → Clean |
| FSPコード生成 | configuration.xml → Generate |
| デバッグ開始 | F11 |
| デバッグ停止 | Ctrl+F2 |

### トラブルシューティング

| 問題 | 解決方法 |
|------|----------|
| ビルドエラー: ヘッダが見つからない | FSPコード生成を実行 |
| リンクエラー: 未定義シンボル | FSPスタック設定を確認 |
| デバッグ接続失敗 | J-Link接続、ドライバを確認 |
| フラッシュ書き込み失敗 | ボードの電源、SW4設定を確認 |

### ビルド済みバイナリの書き込み

J-Link Commander または Renesas Flash Programmer を使用：

```
# J-Link Commander
J-Link> loadfile quickstart_ek_ra8p1_ep.hex
J-Link> r
J-Link> g
```

---

## 関連ファイル

- `e2studio/.project` - Eclipseプロジェクト設定
- `e2studio/.cproject` - C/C++ビルド設定
- `e2studio/configuration.xml` - FSPコンフィグレーション
- `e2studio/quickstart_ek_ra8p1_ep Debug_Flat.launch` - デバッグ設定
- `e2studio/script/fsp.ld` - リンカスクリプト

---

## 次のPhaseへの申し送り

### Phase 3（スタートアップ処理解析）への情報

- エントリポイント: `Reset_Handler` → `SystemInit` → `main`
- FreeRTOS使用のため、`main()` 後はスケジューラが起動
- `hal_entry()` は FreeRTOS 使用時は呼ばれない可能性あり

### Phase 4（FSP設定解析）への情報

- FSP 6.3.0 使用
- 主要スタック: GLCD, MIPI CSI, VIN, I2C, GPT, ADC, OSPI
- RTOS: AWS FreeRTOS
