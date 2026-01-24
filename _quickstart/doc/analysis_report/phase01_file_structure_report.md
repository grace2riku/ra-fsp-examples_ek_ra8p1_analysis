# Phase 1: ファイル構造の把握 - 解析レポート

## 解析日時
2026-01-23（初版）
2026-01-24（更新：FSPコード生成後の構造を反映）

## 概要

EK-RA8P1 評価ボード向けクイックスタートプロジェクトのディレクトリ構造を解析しました。
本プロジェクトはFreeRTOSベースのデモアプリケーションで、LCD表示、カメラ、タッチパネル、LED制御などの機能を統合しています。

### プロジェクト基本情報
- **ターゲットボード**: EK-RA8P1
- **MCU**: RA8P1 (R7KA8P1KFLCAC) - Cortex-M85
- **FSPバージョン**: 6.3.0
- **RTOS**: FreeRTOS (AWS FreeRTOS)
- **ツールチェーン**: GCC ARM Embedded 13.2.1

---

## 詳細解析結果

### 完全なディレクトリツリー

```
quickstart_ek_ra8p1_ep/
├── readme.txt                              # プロジェクト説明・セットアップ手順
└── e2studio/                               # ★ e2 studio プロジェクトディレクトリ
    │
    ├── .project                            # Eclipseプロジェクト設定
    ├── .cproject                           # ★ C/C++プロジェクト設定（188KB）
    ├── configuration.xml                   # ★ FSPコンフィグレーション（306KB）
    ├── ra_cfg.txt                          # FSP設定テキスト出力（214KB）
    │
    ├── .api_xml                            # API XML参照
    ├── .clangd                             # clangd設定
    ├── .secure_azone                       # セキュリティゾーン設定
    ├── .secure_rzone                       # セキュリティリージョン設定
    ├── .secure_xml                         # セキュリティXML設定
    │
    ├── quickstart_ek_ra8p1_ep.hex          # ★ ビルド済みバイナリ（15.8MB）
    ├── quickstart_ek_ra8p1_ep Debug_Flat.launch  # デバッグ起動設定
    ├── quickstart_ek_ra8p1_ep Debug_Flat.jlink   # J-Link設定
    │
    ├── .settings/                          # IDE設定ディレクトリ
    │   ├── org.eclipse.cdt.core.prefs      # CDT設定
    │   ├── org.eclipse.cdt.managedbuilder.core.prefs
    │   ├── e2studio_project.prefs          # e2 studio設定
    │   ├── language.settings.xml           # 言語設定
    │   ├── DebugVirtualConsoleSetting.xml  # デバッグコンソール設定
    │   ├── CoverageSetting.xml             # カバレッジ設定
    │   ├── IORegisterSetting.xml           # IOレジスタ設定
    │   └── com.renesas.cdt.ddsc.*.prefs    # Renesas固有設定
    │
    ├── build/                              # ビルド出力ディレクトリ
    │   └── compile_commands.json           # コンパイルコマンドDB
    │
    ├── Debug/                              # ★ ビルド出力（.gitignoreで除外）
    │   ├── *.o, *.d                        # オブジェクトファイル、依存関係ファイル
    │   └── quickstart_ek_ra8p1_ep.elf      # ELFバイナリ
    │
    ├── script/                             # ★ リンカスクリプト
    │   ├── fsp.ld                          # FSPリンカスクリプト（メイン）
    │   └── RA8x1_Reset_OSPI.JLinkScript    # J-Link OSPIリセットスクリプト
    │
    ├── ra/                                 # ★ FSPドライバソース（自動生成）
    │   ├── arm/CMSIS_6/                    # ARM CMSIS v6ライブラリ
    │   ├── aws/FreeRTOS/                   # FreeRTOSソースコード
    │   ├── board/ra8p1_ek/                 # EK-RA8P1ボードサポート
    │   ├── fsp/                            # FSPドライバ実装
    │   │   ├── inc/api/                    # APIヘッダ
    │   │   ├── inc/instances/              # インスタンスヘッダ
    │   │   └── src/                        # ドライバソース
    │   │       ├── bsp/                    # BSP（ボードサポートパッケージ）
    │   │       ├── r_adc_b/                # ADC-Bドライバ
    │   │       ├── r_drw/                  # D/AVE 2Dドライバ
    │   │       ├── r_glcdc/                # GLCDCドライバ
    │   │       ├── r_gpt/                  # GPTドライバ
    │   │       ├── r_icu/                  # ICUドライバ
    │   │       ├── r_iic_master/           # I2Cマスタドライバ
    │   │       ├── r_ioport/               # IOポートドライバ
    │   │       ├── r_mipi_csi/             # MIPI CSIドライバ
    │   │       ├── r_mipi_phy/             # MIPI PHYドライバ
    │   │       ├── r_ospi_b/               # OSPI-Bドライバ
    │   │       ├── r_sci_b_uart/           # SCI UART-Bドライバ
    │   │       ├── r_vin/                  # ビデオ入力ドライバ
    │   │       └── rm_freertos_port/       # FreeRTOSポート
    │   └── tes/dave2d/                     # D/AVE 2Dグラフィックスエンジン
    │
    ├── ra_cfg/                             # ★ FSP設定ヘッダ（自動生成）
    │   ├── aws/FreeRTOSConfig.h            # FreeRTOS設定
    │   └── fsp_cfg/                        # FSPモジュール設定
    │       ├── bsp/                        # BSP設定ヘッダ
    │       └── r_*_cfg.h                   # 各ドライバ設定ヘッダ
    │
    ├── ra_gen/                             # ★ FSP自動生成コード
    │   ├── main.c                          # メイン関数（スレッド起動）
    │   ├── hal_data.c/h                    # HALデータ定義（199KB）
    │   ├── common_data.c/h                 # 共通データ定義
    │   ├── vector_data.c/h                 # ベクタテーブル定義
    │   ├── pin_data.c                      # ピン設定データ
    │   ├── bsp_clock_cfg.h                 # クロック設定
    │   ├── *_thread.c/h                    # 各スレッドスタブ
    │   └── ...
    │
    └── src/                                # ★ アプリケーションソースコード
        │
        ├── hal_entry.c                     # ★ HALエントリポイント
        ├── common_init.c                   # ★ 共通初期化処理
        ├── common_init.h                   # ★ 共通定義・マクロ
        ├── common_utils.c                  # ユーティリティ関数
        ├── common_utils.h                  # ユーティリティ定義
        ├── r_typedefs.h                    # 型定義
        │
        ├── [FreeRTOSスレッド]
        │   ├── main_menu_thread_entry.c    # ★ メインメニュースレッド
        │   ├── display_thread_entry.c      # ★ ディスプレイスレッド
        │   ├── display_thread_entry.h
        │   ├── camera_thread_entry.c       # ★ カメラスレッド（55KB）
        │   ├── tp_thread_entry.c           # タッチパネルスレッド
        │   ├── tp_thread_entry.h
        │   ├── blinky_thread_entry.c       # LEDブリンキースレッド
        │   └── board_mon_thread_entry.c    # ボード監視スレッド
        │
        ├── [メニュー機能]
        │   ├── menu_main.c                 # メインメニュー処理
        │   ├── menu_main.h
        │   ├── menu_kis.c                  # Kit Information Screen
        │   ├── menu_kis.h
        │   ├── menu_led.c                  # LED制御メニュー（37KB）
        │   ├── menu_led.h
        │   ├── menu_ext.c                  # 外部メモリメニュー
        │   ├── menu_ext.h
        │   ├── menu_ns.c                   # Next Stepsメニュー
        │   ├── menu_ns.h
        │   ├── menu_lcd.c                  # LCD制御
        │   └── menu_lcd.h
        │
        ├── [ボードハードウェア制御]
        │   ├── board_hw_cfg.c              # ハードウェア設定
        │   ├── board_hw_cfg.h
        │   ├── board_cfg_switch.c          # スイッチ設定
        │   ├── board_cfg_switch.h
        │   ├── board_greenpak.c            # GreenPAK IC制御
        │   ├── board_greenpak.h
        │   ├── board_i2c_master.c          # I2Cマスター制御
        │   ├── board_i2c_master.h
        │   ├── board_sdram.c               # SDRAM制御
        │   └── board_sdram.h
        │
        ├── [外部デバイスドライバ]
        │   ├── ospi_commands.c             # OSPIコマンド
        │   ├── ospi_commands.h
        │   ├── ospi_b_commands.c           # OSPI-Bコマンド
        │   ├── ospi_b_commands.h
        │   ├── ospi_b_ep.c                 # OSPI-B制御
        │   ├── ospi_b_ep.h
        │   ├── ov5640.c                    # OV5640カメラドライバ
        │   ├── ov5640_cfg.h                # カメラ設定
        │   ├── touch_FT5316.c              # FT5316タッチパネルドライバ
        │   └── touch_FT5316.h
        │
        ├── [コンソール]
        │   ├── jlink_console.c             # J-Linkコンソール入出力
        │   └── jlink_console.h
        │
        └── images/                         # 画像リソース
            ├── backgrounds/                # 背景画像
            │   ├── welcome_screen_1024x600.c       # ウェルカム画面
            │   ├── welcome_screen_1024x600.bmp
            │   ├── user_led_background_1024x600.c  # LED画面背景
            │   └── user_led_background_1024x600.bmp
            │
            ├── overlays/                   # オーバーレイ画像
            │   ├── menu_overlay.c          # メニューオーバーレイ
            │   └── menu_overlay.png
            │
            ├── user_font_title/            # タイトル用フォント（A-Z）
            │   ├── A.c, A.png ... Z.c, Z.png
            │   ├── SPACE.c, MINUS.c, UNDERSCORE.c
            │   └── user_font_title_if.c/h
            │
            └── user_font_body/             # 本文用フォント
                ├── uc/                     # 大文字（A-Z）
                ├── lc/                     # 小文字（a-z）
                ├── 0-9.c/png               # 数字
                ├── 各種記号.c/png           # 記号類
                └── user_font_body_if.c/h
```

---

### ファイル分類

#### 1. アプリケーションコード（ユーザー編集対象）

| ファイル | サイズ | 役割 |
|---------|--------|------|
| `hal_entry.c` | 4.5KB | HALエントリポイント、WarmStart処理 |
| `common_init.c/h` | 13KB/9.5KB | 共通初期化、定数定義、構造体定義 |
| `common_utils.c/h` | 5KB/6KB | ユーティリティ関数 |
| `main_menu_thread_entry.c` | 6KB | メインメニュースレッド |
| `display_thread_entry.c/h` | 35KB/2.8KB | ディスプレイ制御スレッド |
| `camera_thread_entry.c` | 55KB | カメラ制御スレッド（最大のソース） |
| `tp_thread_entry.c/h` | 26KB/3.3KB | タッチパネルスレッド |
| `blinky_thread_entry.c` | 3.2KB | LEDブリンキースレッド |
| `board_mon_thread_entry.c` | 16KB | ボード監視スレッド |
| `menu_*.c/h` | 各種 | 各メニュー機能実装 |

#### 2. ボードサポートコード

| ファイル | サイズ | 役割 |
|---------|--------|------|
| `board_hw_cfg.c/h` | 3.2KB/4.5KB | ハードウェア設定 |
| `board_cfg_switch.c/h` | 12KB/3.6KB | スイッチ（SW1/SW2）制御 |
| `board_greenpak.c/h` | 6KB/2.4KB | GreenPAK IC制御 |
| `board_i2c_master.c/h` | 11KB/2.6KB | I2Cマスター通信 |
| `board_sdram.c/h` | 9.3KB/5KB | SDRAM初期化・制御 |

#### 3. 外部デバイスドライバ

| ファイル | サイズ | 役割 |
|---------|--------|------|
| `ospi_commands.c/h` | 7KB/4.2KB | OSPIコマンド定義 |
| `ospi_b_commands.c/h` | 5KB/3.5KB | OSPI-Bコマンド定義 |
| `ospi_b_ep.c/h` | 12KB/2.4KB | OSPI-B制御 |
| `ov5640.c` | 11.5KB | OV5640カメラモジュールドライバ |
| `ov5640_cfg.h` | 2.8KB | カメラ設定パラメータ |
| `touch_FT5316.c/h` | 7.2KB/4.1KB | FT5316タッチパネルドライバ |

#### 4. プロジェクト設定ファイル（自動生成・編集注意）

| ファイル | サイズ | 役割 |
|---------|--------|------|
| `configuration.xml` | 306KB | ★ FSPコンフィグレーション（GUIで編集） |
| `.cproject` | 188KB | C/C++プロジェクト設定 |
| `.project` | 1.5KB | Eclipseプロジェクト設定 |
| `ra_cfg.txt` | 214KB | FSP設定のテキスト出力 |

#### 5. リンカスクリプト

| ファイル | 場所 | 役割 |
|---------|------|------|
| `fsp.ld` | `script/` | FSPリンカスクリプト（memory_regions.ld, fsp_gen.ldをインクルード） |
| `memory_regions.ld` | `Debug/`（ビルド時生成） | メモリ領域定義（自動生成） |
| `fsp_gen.ld` | `Debug/`（ビルド時生成） | FSP生成リンカスクリプト |

#### 6. FSP自動生成コード（ra_gen/）

| ファイル | サイズ | 役割 |
|---------|--------|------|
| `main.c` | 8.7KB | メイン関数、スレッド起動 |
| `hal_data.c/h` | 199KB/4.3KB | HALインスタンス・設定データ |
| `common_data.c/h` | 35KB/7.5KB | 共通データ定義 |
| `vector_data.c/h` | 8.9KB/10KB | 割り込みベクタテーブル |
| `pin_data.c` | 26KB | ピン設定データ |
| `bsp_clock_cfg.h` | 5.4KB | クロック設定 |
| `*_thread.c/h` | 各3-4KB | 各スレッドのスタブコード |

#### 7. 画像リソース

| ディレクトリ | ファイル数 | 内容 |
|-------------|-----------|------|
| `backgrounds/` | 4 | 背景画像（ウェルカム、LED画面） |
| `overlays/` | 2 | メニューオーバーレイ |
| `user_font_title/` | 60+ | タイトル用カスタムフォント |
| `user_font_body/` | 120+ | 本文用カスタムフォント |

---

### メモリ配置（Debug/memory_regions.ld より）

※ ビルド時に`Debug/`ディレクトリに自動生成される

| 領域 | 開始アドレス | サイズ | 用途 |
|------|-------------|--------|------|
| ITCM | 0x00000000 | 128KB | 命令TCM（高速実行） |
| DTCM | 0x20000000 | 128KB | データTCM（高速アクセス） |
| RAM | 0x22000000 | 1,872KB | メインRAM |
| FLASH | 0x02000000 | 1MB | 内蔵フラッシュ |
| SDRAM | 0x68000000 | 128MB | 外部SDRAM |
| OSPI1_CS0 | 0x70000000 | 128MB | OSPI1チップセレクト0 |
| OSPI1_CS1 | 0x78000000 | 128MB | OSPI1チップセレクト1 |
| OSPI0_CS0 | 0x80000000 | 256MB | OSPI0チップセレクト0 |
| OSPI0_CS1 | 0x90000000 | 256MB | OSPI0チップセレクト1 |

---

## 発見事項

### 1. プロジェクト構造の特徴

- **FSPコード生成済み**: `ra/`, `ra_gen/`, `ra_cfg/` ディレクトリが含まれている
  - `ra/`: FSPドライバソース（CMSIS、FreeRTOS、FSPドライバ、D/AVE 2D）
  - `ra_gen/`: FSP自動生成コード（スレッドスタブ、HALデータ、ベクタテーブル等）
  - `ra_cfg/`: FSP設定ヘッダ（各モジュールの設定）
  - e2 studioで`configuration.xml`を開いて再生成も可能

- **ビルド済みバイナリ提供**: `quickstart_ek_ra8p1_ep.hex`（15.8MB）が同梱
  - 即座に動作確認可能

### 2. アプリケーションアーキテクチャ

- **FreeRTOSマルチスレッド構成**: 6つのスレッドで機能を分離
  - main_menu_thread: メニュー制御
  - display_thread: LCD表示制御
  - camera_thread: カメラ映像処理（最大規模）
  - tp_thread: タッチパネル入力処理
  - blinky_thread: LED制御
  - board_mon_thread: ボード状態監視

- **モジュール化設計**: 各機能がファイル単位で分離

### 3. 使用している周辺機能

- **ディスプレイ**: GLCD（LCD解像度1024x600）
- **カメラ**: MIPI CSI + OV5640
- **タッチパネル**: FT5316（I2C接続）
- **メモリ**: OSPI（外部フラッシュ）、SDRAM
- **通信**: I2C、シリアルコンソール（J-Link経由）
- **その他**: GreenPAK IC、ADC（温度センサ）、GPT（PWM）

### 4. 画像データの特徴

- 画像データはC言語配列として変換済み
- RGB565形式で格納
- カスタムフォントはアルファベット・数字・記号を個別ファイルで管理

---

## 関連ファイル

- `_quickstart/quickstart_ek_ra8p1_ep/readme.txt`
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/configuration.xml`
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/src/*.c`
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/ra_gen/*.c`
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/ra/fsp/src/**/*.c`

---

## ファイル編集時の注意事項

### 編集可能なファイル

| 分類 | ファイル | 注意点 |
|------|---------|--------|
| アプリケーション | `src/*.c`, `src/*.h` | 自由に編集可能 |
| 画像データ | `src/images/**/*.c` | 元画像から再生成する場合はツール使用 |

### 編集すべきでないファイル（自動生成）

| 分類 | ファイル | 理由 |
|------|---------|------|
| FSP設定 | `configuration.xml` | e2 studio GUIで編集 |
| リンカスクリプト | `Debug/memory_regions.ld`, `Debug/fsp_gen.ld` | ビルド時に自動生成 |
| IDE設定 | `.cproject`, `.project` | e2 studioが管理 |
| FSP生成コード | `ra_gen/*`, `ra_cfg/*`, `ra/*` | FSPコンフィグレータで再生成される |
| ビルド出力 | `Debug/*` | ビルド時に生成（.gitignoreで除外推奨） |

### FSPコード生成について

本リポジトリにはFSP生成済みコードが含まれています。再生成する場合：

1. e2 studioで `configuration.xml` を開く
2. FSPコンフィグレータで設定を確認/変更
3. 「Generate Project Content」を実行
4. 以下のディレクトリが更新される：
   - `ra_gen/` - FSP自動生成コード
   - `ra_cfg/` - FSP設定ヘッダ
   - `ra/` - FSPドライバソース

5. ビルド実行時に以下が生成される：
   - `Debug/memory_regions.ld` - メモリ領域定義
   - `Debug/fsp_gen.ld` - FSP生成リンカスクリプト
   - `Debug/*.o`, `Debug/*.elf` - コンパイル出力

---

## 次のPhaseへの申し送り

### Phase 2（ビルドシステム解析）への情報

- `.cproject` に GCC ARM Embedded 13.2.1 の設定が含まれる
- リンカスクリプトは `fsp.ld` → `memory_regions.ld` + `fsp_gen.ld` の構成
- ビルド済み `.hex` ファイルが存在（15.8MB）

### Phase 3（コード構造解析）への情報

- エントリポイント: `hal_entry.c` → `hal_entry()` 関数
- FreeRTOSスレッドが6つ存在
- `common_init.c` で共通初期化処理を実施
- 主要なマクロ・定数は `common_init.h` に定義

### 補足情報

- 解析対象ソースファイル: 105個（.c）、206個（.h）※画像・Debug除く
  - うちアプリケーションコード: 26個（.c）、24個（.h）
  - うちFSP/CMSIS/FreeRTOS: 79個（.c）、182個（.h）
- 最大のソースファイル: `ra_gen/hal_data.c`（199KB）
- 最大のアプリケーションソース: `camera_thread_entry.c`（55KB）
- FSP設定ファイル: `configuration.xml`（306KB）
