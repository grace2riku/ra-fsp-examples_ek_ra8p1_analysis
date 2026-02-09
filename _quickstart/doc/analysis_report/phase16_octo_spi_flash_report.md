# Phase 16: Octo-SPI フラッシュ解析レポート

## 1. エグゼクティブサマリー

本解析では、EK-RA8P1ボード上のOcto-SPIフラッシュメモリ（Macronix MX25LW51245G, 64MByte）について、メモリ領域、ハードウェア設定、プログラムインターフェース、制御方法を包括的に調査した。本プロジェクトでは、OSPI フラッシュは主に **LCD背景画像の格納**と**パフォーマンステストのデータ読み書き**に使用されている。FSP の OSPI_B ドライバを通じて SPI モードと OPI (8D-8D-8D DTR) モードの両方をサポートし、メモリマップドアクセスによる高速読み出しを実現している。

---

## 2. メモリ領域の範囲

### 2.1 ハードウェアとして使えるメモリ領域のサイズ

| 項目 | 仕様 |
|------|------|
| フラッシュデバイス | Macronix MX25LW51245G |
| 容量 | 512Mbit（64MByte） |
| デバイスID | 0x3A86C2 |
| セクタサイズ | 4KB（4,096 bytes） |
| ブロックサイズ | 256KB（262,144 bytes） |
| ページサイズ（書き込み単位） | 64 bytes |

**RA8P1 MCUのOSPIメモリマップドアドレス空間**:

| チップセレクト | 開始アドレス | 定義元 |
|---------------|-------------|--------|
| CS0 (Device 0) | `0x80000000` | `bsp_feature.h` BSP_FEATURE_OSPI_B_DEVICE_0_START_ADDRESS |
| CS1 (Device 1) | `0x90000000` | `bsp_feature.h` BSP_FEATURE_OSPI_B_DEVICE_1_START_ADDRESS |

各チップセレクトにつき、最大256MBのアドレス空間が割り当てられている（0x80000000〜0x8FFFFFFF, 0x90000000〜0x9FFFFFFF）。実際のフラッシュデバイスは64MByteなので、デバイス1つあたり0x04000000分のアドレス空間を使用する。

```
メモリマップ:
┌─────────────────────────────────────────┐
│ 0x80000000 - 0x83FFFFFF  CS0 (64MB)     │  ← OSPI_B Device 0
├─────────────────────────────────────────┤
│ 0x84000000 - 0x8FFFFFFF  (未使用)       │
├─────────────────────────────────────────┤
│ 0x90000000 - 0x93FFFFFF  CS1 (64MB)     │  ← OSPI_B Device 1 ★本プロジェクトで使用
├─────────────────────────────────────────┤
│ 0x94000000 - 0x9FFFFFFF  (未使用)       │
└─────────────────────────────────────────┘
```

> **補足**: 本プロジェクトでは `hal_data.c` の `g_ospi0_extended_cfg` にて `.channel = (ospi_b_device_number_t) 1` と設定されており、**CS1（Device 1、ベースアドレス 0x90000000）を使用**している。

### 2.2 プログラムで設定しているメモリ領域

#### 2.2.1 FSPコンフィグレーション (`hal_data.c`)

| 設定項目 | 値 | ソース |
|----------|-----|--------|
| OSPI_Bユニット | 0 | `g_ospi0_extended_cfg.ospi_b_unit = 0` |
| チャネル（デバイス番号） | 1 (CS1) | `g_ospi0_extended_cfg.channel = 1` |
| ページサイズ | 64 bytes | `g_ospi0_cfg.page_size_bytes = 64` |
| アドレスバイト数 | 4 bytes | `g_ospi0_cfg.address_bytes = SPI_FLASH_ADDRESS_BYTES_4` |
| 初期プロトコル | 1S-1S-1S (Extended SPI) | `g_ospi0_cfg.spi_protocol = SPI_FLASH_PROTOCOL_1S_1S_1S` |
| DOTF開始アドレス | 0x90000000 | `g_ospi_dotf_cfg.p_start_addr` (※DOTF無効時は使用されない) |
| DOTF終了アドレス | 0x90001FFF | `g_ospi_dotf_cfg.p_end_addr` (※DOTF無効時は使用されない) |

> **参照**: `hal_data.c:3579-3618`

#### 2.2.2 アプリケーションで使用しているアドレス

| 用途 | アドレス | 定義 | ファイル |
|------|---------|------|---------|
| パフォーマンステスト用領域 | `0x93F00000` | `OSPI_START_ADDRESS` | `menu_ext.c:60` |
| 背景画像データ (welcome) | `.ospi0_cs1` セクション | セクション属性で配置 | `welcome_screen_1024x600.c` |
| 背景画像データ (user_led) | `.ospi0_cs1` セクション | セクション属性で配置 | `user_led_background_1024x600.c` |

パフォーマンステスト用領域は CS1 の末尾近く（0x93F00000）に配置されており、背景画像データと重ならないように設計されている。

#### 2.2.3 セクション属性によるデータ配置

背景画像は以下のようにセクション属性を指定して OSPI フラッシュに配置されている:

```c
// welcome_screen_1024x600.c
const st_full_image_rgb565_t g_ref_000_welcome_screen
    BSP_ALIGN_VARIABLE(64)
    __attribute__ ((section (".ospi0_cs1"))) = {
    1024, 600, 2,
    "\004\000\004\000..."  // 1024×600×2 = 1,228,800 bytes のピクセルデータ
};

// user_led_background_1024x600.c
const st_full_image_rgb565_t g_ref_001_user_led_background
    BSP_ALIGN_VARIABLE(64)
    __attribute__ ((section (".ospi0_cs1"))) = {
    1024, 600, 2,
    "\000\000\000\000..."  // 同サイズのピクセルデータ
};
```

各画像データの構造体 `st_full_image_rgb565_t` の定義（`r_typedefs.h`）:

```c
typedef struct {
    uint16_t  width;             // 画像幅 (1024)
    uint16_t  height;            // 画像高さ (600)
    uint16_t  bytes_per_pixel;   // 1ピクセルあたりのバイト数 (2: RGB565)
    uint8_t   pixel_data[...];   // ピクセルデータ配列
} st_full_image_rgb565_t;
```

### 2.3 リンカスクリプトの設定

#### 2.3.1 リンカスクリプトの構成

```
fsp.ld （メインリンカスクリプト）
  ├── INCLUDE memory_regions.ld  （メモリ領域定義 - ビルド時に自動生成）
  └── INCLUDE fsp_gen.ld         （セクション配置定義 - ビルド時に自動生成）
```

**参照**: `script/fsp.ld`

```ld
INCLUDE memory_regions.ld
INCLUDE fsp_gen.ld
```

`memory_regions.ld` と `fsp_gen.ld` は FSP コンフィグレータ（e2 studio）でプロジェクトをビルドする際に自動生成される。これらは `configuration.xml` の設定に基づいて生成され、以下の内容が含まれる:

- **memory_regions.ld**: FLASH, RAM, DATA_FLASH, OSPI_DEVICE_0, OSPI_DEVICE_1, SDRAM 等のメモリ領域定義（MEMORY コマンド）
- **fsp_gen.ld**: 各セクション（`.text`, `.data`, `.bss`, `.ospi0_cs1` 等）の配置ルール（SECTIONS コマンド）

> **注意**: これらの生成ファイルは本リポジトリには含まれていない。ビルド時に e2 studio が `configuration.xml` の設定から自動生成する。

#### 2.3.2 OSPIセクションの配置

`.ospi0_cs1` セクションは、リンカスクリプトにより CS1 のメモリ領域（`0x90000000` 以降）に配置される。`BSP_ALIGN_VARIABLE(64)` により64バイトアラインメントが保証される。

---

## 3. ハードウェア設定

### 3.1 画像・データ・コードを配置するために必要な設定

Octo-SPI フラッシュにデータを配置するには、以下の設定が必要:

| 手順 | 設定内容 | 設定ファイル/ツール |
|------|---------|-------------------|
| 1. FSP設定 | OSPI_Bモジュールをスタックに追加 | `configuration.xml`（e2 studioで設定） |
| 2. ピン設定 | OSPIピン（SIO0-7, CLK, CS, DQS, RESET）を割り当て | `bsp_pin_cfg.h` |
| 3. クロック設定 | OCTACLKを適切な周波数に設定 | `bsp_clock_cfg.h` |
| 4. コマンドセット設定 | SPI/OPIモードのコマンドを定義 | `hal_data.c` (自動生成) |
| 5. セクション属性指定 | データに `__attribute__((section(".ospi0_cs1")))` を付与 | ソースコード |
| 6. リンカスクリプト | `.ospi0_cs1` セクションをOSPIメモリ領域にマッピング | `fsp_gen.ld` (自動生成) |
| 7. デバッガ設定 | JLinkScript でフラッシュ書き込み前にOSPIリセット | `RA8x1_Reset_OSPI.JLinkScript` |

### 3.2 IOポート設定

**参照**: `bsp_pin_cfg.h`

OSPI で使用する全ピン一覧:

| ピン名 | ポート | 機能 | 備考 |
|--------|--------|------|------|
| OSPI_SIO0 | P1_00 (BSP_IO_PORT_01_PIN_00) | データ信号 SIO0 (MISO相当) | SW4切替 |
| OSPI_SIO1 | P8_03 (BSP_IO_PORT_08_PIN_03) | データ信号 SIO1 | SW4切替 |
| OSPI_SIO2 | P1_03 (BSP_IO_PORT_01_PIN_03) | データ信号 SIO2 (SS相当) | SW4切替 |
| OSPI_SIO3 | P1_01 (BSP_IO_PORT_01_PIN_01) | データ信号 SIO3 (MOSI相当) | SW4切替 |
| OSPI_SIO4 | P1_02 (BSP_IO_PORT_01_PIN_02) | データ信号 SIO4 | SW4切替 |
| OSPI_SIO5 | P8_00 (BSP_IO_PORT_08_PIN_00) | データ信号 SIO5 | SW4切替 |
| OSPI_SIO6 | P8_02 (BSP_IO_PORT_08_PIN_02) | データ信号 SIO6 | SW4切替 |
| OSPI_SIO7 | P8_04 (BSP_IO_PORT_08_PIN_04) | データ信号 SIO7 | SW4切替 |
| OSPI_CLK | P8_08 (BSP_IO_PORT_08_PIN_08) | クロック信号 | SW4切替 |
| OSPI_CS | P1_04 (BSP_IO_PORT_01_PIN_04) | チップセレクト (CS0) | SW4切替 |
| OSPI_ECS | P1_05 (BSP_IO_PORT_01_PIN_05) | 拡張チップセレクト (CS1) | SW4切替 |
| OSPI_DQS | P8_01 (BSP_IO_PORT_08_PIN_01) | データストローブ | SW4切替 |
| OSPI_RESET | P1_06 (BSP_IO_PORT_01_PIN_06) | リセット信号 | SW4切替 |

> **注意**: すべてのOSPIピンはボード上のSW4（DIPスイッチ）の設定に依存する。OSPIを使用する場合は、SW4を正しく設定する必要がある（PMOD1機能と排他）。

### 3.3 クロック設定

**参照**: `bsp_clock_cfg.h:65-66`

| 項目 | 設定値 | 備考 |
|------|--------|------|
| OCTACLKソース | PLL2Q | `BSP_CFG_OCTACLK_SOURCE = BSP_CLOCKS_SOURCE_CLOCK_PLL2Q` |
| PLL2ソース | メインオシレータ (XTAL 24MHz) | `BSP_CFG_PLL2_SOURCE` |
| PLL2分周比 | /3 | `BSP_CFG_PLL2_DIV` |
| PLL2逓倍比 | x300 | `BSP_CFG_PLL2_MUL` |
| PLL2周波数 | 2,400 MHz | `BSP_CFG_PLL2_FREQUENCY_HZ` |
| PLL2Q分周比 | /8 | `BSP_CFG_PL2ODIVQ` |
| PLL2Q周波数 | 300 MHz | `BSP_CFG_PLL2Q_FREQUENCY_HZ` |
| OCTACLK分周比 | /5 | `BSP_CFG_OCTACLK_DIV` |
| **OCTACLK周波数** | **60 MHz** | 300MHz / 5 = 60MHz |

```
クロックツリー:
XTAL (24MHz)
  └── PLL2 (/3 × 300 = 2400MHz)
        └── PLL2Q (/8 = 300MHz)
              └── OCTACLK (/5 = 60MHz) → OSPI_B ペリフェラル
```

### 3.4 その他のハードウェア初期設定

#### 3.4.1 OSPI_Bペリフェラルのタイミング設定

**参照**: `hal_data.c:3488-3495`

```c
static ospi_b_timing_setting_t g_ospi0_timing_settings = {
    .command_to_command_interval = OSPI_B_COMMAND_INTERVAL_CLOCKS_2,    // コマンド間隔: 2クロック
    .cs_pullup_lag              = OSPI_B_COMMAND_CS_PULLUP_CLOCKS_NO_EXTENSION,  // CS解除遅延: なし
    .cs_pulldown_lead           = OSPI_B_COMMAND_CS_PULLDOWN_CLOCKS_NO_EXTENSION, // CS設定遅延: なし
    .sdr_drive_timing           = OSPI_B_SDR_DRIVE_TIMING_BEFORE_CK,   // SDRドライブ: CK前
    .sdr_sampling_edge          = OSPI_B_CK_EDGE_FALLING,              // SDRサンプリング: 立下り
    .sdr_sampling_delay         = OSPI_B_SDR_SAMPLING_DELAY_NONE,      // サンプリング遅延: なし
    .ddr_sampling_extension     = OSPI_B_DDR_SAMPLING_EXTENSION_NONE,  // DDR拡張: なし
};
```

#### 3.4.2 OSPI_B設定フラグ

**参照**: `r_ospi_b_cfg.h`

| 設定項目 | 値 | 説明 |
|----------|-----|------|
| DMAC_SUPPORT | 0 (無効) | DMAを使用しない |
| XIP_SUPPORT | 0 (無効) | XIP (Execute In Place) を使用しない |
| AUTOCALIBRATION_SUPPORT | 0 (無効) | 自動キャリブレーションを使用しない |
| PREFETCH_FUNCTION | 1 (有効) | プリフェッチ機能を有効化 |
| COMBINATION_FUNCTION | 64BYTE | 最大64バイトのコマンド結合 |
| DOTF_SUPPORT | 0 (無効) | 暗号化復号 (Decrypted On-The-Fly) を使用しない |
| DATA_LATCH_DELAY | 16 | データラッチ遅延: 16クロックセル |

---

## 4. プログラムとのインターフェース

### 4.1 アプリとOcto-SPIフラッシュアクセスのインターフェース

#### 4.1.1 アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────┐
│  アプリケーション層                                               │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │ menu_ext.c   │  │ main_menu_       │  │ display_thread_  │  │
│  │ (パフォーマンス│  │ thread_entry.c   │  │ entry.c          │  │
│  │  テスト)      │  │ (画像コピー)      │  │ (画面描画)        │  │
│  └──────┬───────┘  └────────┬─────────┘  └────────┬─────────┘  │
│         │                   │                      │            │
│  ┌──────▼───────┐          │                      │            │
│  │ ospi_b_ep.c  │          │                      │            │
│  │ (初期化/     │          │                      │            │
│  │  プロトコル   │          │                      │            │
│  │  切替)        │          │                      │            │
│  └──────┬───────┘          │                      │            │
├─────────┼──────────────────┼──────────────────────┼────────────┤
│  FSP API層                 │                      │            │
│  ┌──────▼──────────────────▼──────────────────────▼──────────┐ │
│  │  g_ospi0 (spi_flash_instance_t)                           │ │
│  │  ├── .p_api->open()          R_OSPI_B_Open                │ │
│  │  ├── .p_api->write()         R_OSPI_B_Write               │ │
│  │  ├── .p_api->erase()         R_OSPI_B_Erase               │ │
│  │  ├── .p_api->statusGet()     R_OSPI_B_StatusGet           │ │
│  │  ├── R_OSPI_B_DirectTransfer() (レジスタ直接操作)           │ │
│  │  └── R_OSPI_B_SpiProtocolSet() (プロトコル切替)             │ │
│  └───────────────────────────────────────────────────────────┘ │
├───────────────────────────────────────────────────────────────┤
│  メモリマップドアクセス (memcpy/ポインタアクセス)                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  0x90000000 - 0x93FFFFFF  CS1 メモリマップド領域            │ │
│  │  ├── .ospi0_cs1 セクション (背景画像データ)                 │ │
│  │  └── 0x93F00000 (パフォーマンステスト用領域)                │ │
│  └───────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

#### 4.1.2 2つのアクセス方式

**方式1: メモリマップドアクセス（読み出し専用）**

OPIモードに切り替え後、OSPI フラッシュのアドレス空間にメモリマップされたポインタを通じて直接読み出す。`memcpy` やポインタデリファレンスで普通のメモリのように読める。

```c
// 例: 背景画像データの読み出し（main_menu_thread_entry.c）
memcpy(&g_sdram_buffer_welcome[0],
       &g_ref_000_welcome_screen.pixel_data[0],  // ← 0x90xxxxxx のOSPIアドレス
       sizeof(g_sdram_buffer_welcome));

// 例: パフォーマンステストの読み出し（menu_ext.c:588-592）
ptr = gp_ospi_cs1;  // = (uint8_t*)0x93F00000
memcpy(s_read_block, ptr, block_size);
```

**方式2: FSP API経由のコマンドアクセス（書き込み/消去/設定）**

書き込み・消去・フラッシュデバイスレジスタ操作には FSP API を使用する。

```c
// 書き込み
g_ospi0.p_api->write(g_ospi0.p_ctrl, src_data, dest_address, size);

// 消去（4KBセクタ単位）
g_ospi0.p_api->erase(g_ospi0.p_ctrl, address, erase_size);

// ステータス確認（書き込み完了待ち）
g_ospi0.p_api->statusGet(g_ospi0.p_ctrl, &status);

// レジスタ直接操作（Write Enable, Configuration Register等）
R_OSPI_B_DirectTransfer(&g_ospi0_ctrl, &transfer, direction);

// プロトコル切替
R_OSPI_B_SpiProtocolSet(&g_ospi0_ctrl, protocol);
```

### 4.2 Octo-SPIフラッシュ関連ファイルの機能説明

#### 4.2.1 関連ファイル一覧

| # | ファイル | 層 | 機能 |
|---|---------|-----|------|
| 1 | `src/ospi_b_ep.h` | アプリ | OSPI初期化・プロトコル切替のAPI宣言 |
| 2 | `src/ospi_b_ep.c` | アプリ | OSPI初期化・プロトコル切替の実装 |
| 3 | `src/ospi_b_commands.h` | アプリ | OSPIフラッシュコマンド定義（SPI/OPI、初期化用） |
| 4 | `src/ospi_b_commands.c` | アプリ | OSPIフラッシュコマンドテーブル（初期化用） |
| 5 | `src/ospi_commands.h` | アプリ | OSPIフラッシュコマンド定義（テスト用、拡張版） |
| 6 | `src/ospi_commands.c` | アプリ | OSPIフラッシュコマンドテーブル（テスト用、拡張版） |
| 7 | `src/menu_ext.h` | アプリ | 外部メモリメニューのAPI宣言 |
| 8 | `src/menu_ext.c` | アプリ | パフォーマンステスト・メニューUI実装 |
| 9 | `src/common_init.h` | アプリ | DTR_MODE定義、画像データextern宣言 |
| 10 | `src/images/backgrounds/welcome_screen_1024x600.c` | データ | Welcome画面背景画像（.ospi0_cs1セクション） |
| 11 | `src/images/backgrounds/user_led_background_1024x600.c` | データ | LED制御画面背景画像（.ospi0_cs1セクション） |
| 12 | `ra/fsp/inc/instances/r_ospi_b.h` | FSPドライバ | OSPI_Bドライバの型定義・API宣言 |
| 13 | `ra/fsp/src/r_ospi_b/r_ospi_b.c` | FSPドライバ | OSPI_Bドライバの実装（Open/Write/Erase/Read等） |
| 14 | `ra/fsp/inc/api/r_spi_flash_api.h` | FSP API | SPI Flashの汎用APIインターフェース定義 |
| 15 | `ra/fsp/src/bsp/mcu/all/bsp_ospi_b.h` | BSP | BSP OSPI_B初期化のAPI宣言 |
| 16 | `ra/fsp/src/bsp/mcu/all/bsp_ospi_b.c` | BSP | BSP OSPI_B初期化の実装 |
| 17 | `ra/fsp/src/bsp/mcu/ra8p1/bsp_feature.h` | BSP | RA8P1のOSPIメモリアドレス定義 |
| 18 | `ra_gen/hal_data.c` | 自動生成 | OSPI_Bインスタンス設定（g_ospi0_cfg等） |
| 19 | `ra_gen/bsp_clock_cfg.h` | 自動生成 | OCTACLKクロック設定 |
| 20 | `ra_cfg/fsp_cfg/r_ospi_b_cfg.h` | 自動生成 | OSPI_Bモジュール設定フラグ |
| 21 | `ra_cfg/fsp_cfg/bsp/bsp_pin_cfg.h` | 自動生成 | OSPIピン設定 |
| 22 | `script/fsp.ld` | リンカ | メインリンカスクリプト |
| 23 | `script/RA8x1_Reset_OSPI.JLinkScript` | デバッガ | J-Link用OSPIリセットスクリプト |

#### 4.2.2 各ファイルの詳細説明

**`ospi_b_ep.c` / `ospi_b_ep.h`** — OSPI初期化・プロトコル切替

アプリケーションレベルの OSPI 制御を担当。3つの公開関数を提供:

- **`ospi_b_init()`**: OSPIデバイスの初期化。`R_OSPI_B_Open` でドライバを開き、SPIモードに設定後、リセットシーケンスを実行し、Configuration Register 2 (CFG2) にダミーサイクル等を設定する
- **`ospi_b_set_protocol_to_spi()`**: OPIモードからSPIモードへ切り替え。CFG2レジスタに書き込み後、`R_OSPI_B_SpiProtocolSet` でモジュール側も切替
- **`ospi_b_set_protocol_to_opi()`**: SPIモードからOPI (8D-8D-8D DTR) モードへ切り替え。DQSモード、ダミーサイクル、DTRモードをCFG2レジスタに順次設定

**`ospi_b_commands.c` / `ospi_b_commands.h`** — 初期化用コマンドテーブル

`ospi_b_ep.c` が使用するコマンドテーブル `g_ospi_b_direct_transfer[]` を定義。SPI/OPI各モードでの Write Enable、Read Status、Write/Read CFG2 コマンドを定義。

**`ospi_commands.c` / `ospi_commands.h`** — テスト・拡張用コマンドテーブル

`g_ospi_test_direct_transfer[]` を定義。`ospi_b_commands` より多くのコマンド（Sector/Block Erase、Burst Mode、Manufacturer ID Read 等）を含む拡張版。3バイトアドレスモード（3SPI）のコマンドも定義。

**`menu_ext.c`** — パフォーマンステストUI

ユーザーに対してOcto-SPIフラッシュの書き込み/読み出しパフォーマンスを実演するメニュー。`ext_display_menu` がメインメニューから呼ばれ、`ospi_performance_test` で実際のテストを実行する。

### 4.3 Octo-SPIフラッシュに配置しているデータの説明

#### 4.3.1 配置データ一覧

| データ名 | 型 | サイズ | 内容 | セクション |
|----------|-----|--------|------|-----------|
| `g_ref_000_welcome_screen` | `st_full_image_rgb565_t` | 約1.2MB | Welcome画面の背景画像 (1024×600, RGB565) | `.ospi0_cs1` |
| `g_ref_001_user_led_background` | `st_full_image_rgb565_t` | 約1.2MB | LED制御画面の背景画像 (1024×600, RGB565) | `.ospi0_cs1` |

合計で約2.4MBの画像データがOSPIフラッシュに格納されている。

#### 4.3.2 配置データを利用しているソースコード

**1. `main_menu_thread_entry.c` — 起動時の画像コピー**

システム起動時に、OSPIフラッシュから SDRAM へ画像データをコピーする:

```c
// OSPI → SDRAM へコピー
memcpy(&g_sdram_buffer_welcome[0],
       &g_ref_000_welcome_screen.pixel_data[0],
       sizeof(g_sdram_buffer_welcome));
memcpy(&g_sdram_buffer_led[0],
       &g_ref_001_user_led_background.pixel_data[0],
       sizeof(g_sdram_buffer_led));

// バイトスワップ（SPIモードで書き込まれた画像のバイトペア反転を補正）
for (uint32_t x = 0; x < remaining; x += 2) {
    b0 = g_sdram_buffer_welcome[x + 0];
    b1 = g_sdram_buffer_welcome[x + 1];
    g_sdram_buffer_welcome[x + 0] = b1;
    g_sdram_buffer_welcome[x + 1] = b0;
    // LED背景も同様
}
```

> **バイトスワップの理由**: 画像データはSPIモード（1S-1S-1S）でOSPIフラッシュに書き込まれるが、読み出しはOPIモード（8D-8D-8D）で行われる。この際、バイトペアの順序が反転するため、ソフトウェアで補正が必要。

SDRAMバッファは `.sdram` セクションに配置:
```c
uint8_t g_sdram_buffer_welcome[...] BSP_ALIGN_VARIABLE(64) BSP_PLACE_IN_SECTION(".sdram");
uint8_t g_sdram_buffer_led[...]     BSP_ALIGN_VARIABLE(64) BSP_PLACE_IN_SECTION(".sdram");
```

**2. `display_thread_entry.c` — 画面描画**

SDRAMにコピー済みの画像データを Dave/2D グラフィックスエンジンで LCD に描画:

```c
// graphics_reset_blit_buffer()
if (g_current_backgroud.active_demo == LCD_FULL_BG_USER_LED) {
    d2_setblitsrc(gp_d2_handle, (uint8_t *)&g_sdram_buffer_led[0],
                  1024, 1024, 600, d2_mode_rgb565);
} else if (...== LCD_FULL_BG_WELCOME) {
    d2_setblitsrc(gp_d2_handle, (uint8_t *)&g_sdram_buffer_welcome[0],
                  1024, 1024, 600, d2_mode_rgb565);
}
```

**3. `dsp_set_background()` — 背景切替**

メニュー切替時に、表示する背景画像を選択:

```c
void dsp_set_background(uint32_t choice) {
    switch (choice) {
        case LCD_FULL_BG_USER_LED:
            g_current_backgroud.p_bg_src = (uint8_t *)&g_ref_001_user_led_background.pixel_data[0];
            break;
        case LCD_FULL_BG_WELCOME:
            g_current_backgroud.p_bg_src = (uint8_t *)&g_ref_000_welcome_screen.pixel_data[0];
            break;
        // ...
    }
}
```

**データフロー図**:

```
[OSPI Flash (.ospi0_cs1)]
  g_ref_000_welcome_screen     (1.2MB, RGB565)
  g_ref_001_user_led_background (1.2MB, RGB565)
       │
       │  memcpy (起動時1回)
       ▼
[SDRAM (.sdram)]
  g_sdram_buffer_welcome       (バイトスワップ済み)
  g_sdram_buffer_led           (バイトスワップ済み)
       │
       │  d2_setblitsrc / d2_blitcopy (毎フレーム)
       ▼
[LCD フレームバッファ]
  画面表示 (1024×600)
```

### 4.4 ospi_performance_test関数の処理内容

**参照**: `menu_ext.c:167-229`

`ospi_performance_test` 関数は、Octo-SPI フラッシュの書き込みと読み出しのパフォーマンスを計測する。以下に処理フローを詳述する。

#### 4.4.1 処理フロー

```
ospi_performance_test(data_size, &write_result, &read_result)
│
├── 1. ospi_b_init()              ← OSPI デバイス初期化
│       ├── R_OSPI_B_Open()       ← ドライバオープン（未オープンの場合）
│       ├── R_OSPI_B_SpiProtocolSet(SPI) ← SPIモードに設定
│       ├── XSPI0->LIOCTL_b.RSTCS0 = 0/1 ← フラッシュリセット
│       ├── ospi_b_write_enable() ← Write Enable
│       └── CFG2レジスタ設定      ← ダミーサイクル等
│
├── 2. ospi_b_set_protocol_to_spi() ← SPIモードに切替
│
├── 3. フラッシュ消去（SPIモード）
│       └── for (nb = 0; nb < 16; nb++)  ← 16 × 4KB = 64KB を消去
│           ├── g_ospi0.p_api->erase(gp_ospi_cs1 + 4096*nb, 4096)
│           └── wait_for_write()   ← 書き込み完了待ち
│
├── 4. ospi_b_set_protocol_to_opi() ← OPIモード(8D-8D-8D DTR)に切替
│
├── 5. 書き込みテスト（OPIモード） ← write_test_opi(data_size / 64)
│       ├── GPTタイマー開始
│       ├── for (block = 0; block < num_blocks; block++)
│       │   ├── ランダムデータ生成（srand(0)で再現性確保）
│       │   ├── GPT Start
│       │   ├── g_ospi0.p_api->write(g_test_data, gp_ospi_cs1 + offset, 64)
│       │   ├── wait_for_write()
│       │   └── GPT Stop
│       └── g_counter = タイマーカウント値 → *write_result に格納
│
└── 6. 読み出しテスト（OPIモード） ← read_test_opi(data_size / 64)
        ├── GPTタイマー開始
        ├── memcpy(s_read_block, gp_ospi_cs1, block_size)  ← メモリマップド読み出し
        ├── GPT Stop
        ├── g_counter = タイマーカウント値 → *read_result に格納
        └── データ検証
            ├── srand(0)で同じ乱数列を生成
            └── 読み出しデータと比較、不一致があればエラー報告
```

#### 4.4.2 詳細解説

**消去処理** (`menu_ext.c:179-187`):

```c
for (uint32_t nb = 0; nb < FOURKB_LIMIT; nb++) {  // FOURKB_LIMIT = 16
    err = g_ospi0.p_api->erase(g_ospi0.p_ctrl,
                                (uint8_t*)(gp_ospi_cs1 + (erase_size * nb)),
                                erase_size);  // erase_size = 4096
    wait_for_write();
}
```
- SPIモードで16セクタ（64KB分）を消去
- 各消去後に `wait_for_write()` でステータスレジスタを確認し完了を待つ
- 消去は 0x93F00000 から開始

**書き込みテスト** (`write_test_opi`, `menu_ext.c:489-551`):

```c
// ランダムデータを64バイトずつ生成して書き込み
srand(0);  // 再現性のある乱数シード
for (s_test_current_block = 0; s_test_current_block < num_blocks; s_test_current_block++) {
    // 64バイトのランダムデータを g_test_data に生成
    for (s_test_current_sub = 0; s_test_current_sub < WRITE_BLOCK_SIZE; ...) {
        g_test_data[s_test_current_sub++] = (uint8_t)rand();
    }
    // GPTタイマーで書き込み時間を計測
    R_GPT_Start(g_memory_performance.p_ctrl);
    g_ospi0.p_api->write(g_ospi0.p_ctrl, g_test_data,
                         (uint8_t*)(gp_ospi_cs1 + offset), WRITE_BLOCK_SIZE);
    wait_for_write();
    R_GPT_Stop(g_memory_performance.p_ctrl);
}
```
- OPI (8D-8D-8D DTR) モードで 64バイト単位で書き込み
- GPTタイマー（240MHz）で実行時間をカウント
- `FSP_CRITICAL_SECTION_ENTER` で割り込み禁止区間を設定し、正確な計測を実現

**読み出しテスト** (`read_test_opi`, `menu_ext.c:562-629`):

```c
ptr = gp_ospi_cs1;  // 0x93F00000
R_GPT_Start(g_memory_performance.p_ctrl);
memcpy(s_read_block, ptr, block_size);  // メモリマップド読み出し
R_GPT_StatusGet(g_memory_performance.p_ctrl, &timer_status);
g_counter = timer_status.counter;
```
- メモリマップドアクセスで一括読み出し（`memcpy`）
- 読み出し後、`srand(0)` で同じ乱数列を生成し、書き込みデータと比較してベリファイ
- 不一致があった場合、エラー数と最初の不一致位置を報告

**パフォーマンス結果の計算** (`ext_display_menu`, `menu_ext.c:400-401`):

```c
ospi_write_result = ospi_performance_write_result / GPT_CLOCK_MHZ;  // GPT_CLOCK_MHZ = 240
ospi_read_result  = ospi_performance_read_result / GPT_CLOCK_MHZ;
```
タイマーカウント値を240（MHz）で割ることで、マイクロ秒単位の実行時間に変換。

#### 4.4.3 呼び出し元: ext_display_menu

`ext_display_menu`（`menu_ext.c:239-445`）はメインメニューの「3. External Memory Read Write Demo」として登録されている（`menu_main.c`）。

処理フロー:
1. LCD背景を外部メモリデモ画面に設定
2. ユーザーにブロックサイズを入力させる（2KB〜64KB、2KB単位）
3. 入力値を検証（数値チェック、範囲チェック、2KB境界チェック）
4. `ospi_performance_test` を呼び出してテスト実行
5. 結果（書き込み/読み出し時間）をコンソールとLCDに表示

---

## 5. JLinkScriptファイルの解説

### 5.1 RA8x1_Reset_OSPI.JLinkScript

**参照**: `script/RA8x1_Reset_OSPI.JLinkScript`

```c
int HandleBeforeFlashProg(void)
{
    JLINK_MEM_WriteU8(0x40400d14, 0x00);  // PWR_S: BOWIビットをクリア（PFSWEビット書き込み許可）
    JLINK_MEM_WriteU8(0x40400d14, 0x40);  // PWR_S: PFSWEビットをセット（PFSレジスタ書き込み許可）

    JLINK_MEM_WriteU32(0x40400858, 0x00000005); // P1_06 PFS: 出力 High
    JLINK_SYS_Sleep(1);

    JLINK_MEM_WriteU32(0x40400858, 0x00000004); // P1_06 PFS: 出力 Low
    JLINK_SYS_Sleep(1);

    JLINK_MEM_WriteU32(0x40400858, 0x00000005); // P1_06 PFS: 出力 High
    JLINK_SYS_Sleep(1);

    JLINK_MEM_WriteU8(0x40400d14, 0x00);  // PWR_S: PFSWEビットをクリア
    JLINK_MEM_WriteU8(0x40400d14, 0x80);  // PWR_S: BOWIビットをセット（PFSレジスタ保護）

    return 0;
}
```

#### 5.1.1 このスクリプトの目的

J-Link デバッガがフラッシュプログラミングを開始する**直前**に呼ばれるコールバック関数 `HandleBeforeFlashProg` を定義している。この関数は **OSPI フラッシュデバイスのハードウェアリセット** を実行する。

#### 5.1.2 処理の詳細

| 手順 | レジスタ/アドレス | 操作 | 目的 |
|------|-----------------|------|------|
| 1 | 0x40400D14 (PWPR) | BOWI=0 | Pin Function Select Write Enable ビットへの書き込みを許可 |
| 2 | 0x40400D14 (PWPR) | PFSWE=1 | PFS レジスタへの書き込みを許可 |
| 3 | 0x40400858 (P106 PFS) | 出力 High (0x05) | OSPI_RESETピン (P1_06) を High に設定 |
| 4 | (1ms待機) | - | リセット信号安定化 |
| 5 | 0x40400858 (P106 PFS) | 出力 Low (0x04) | OSPI_RESETピンを Low に設定 → **リセットアサート** |
| 6 | (1ms待機) | - | リセットパルス幅確保 |
| 7 | 0x40400858 (P106 PFS) | 出力 High (0x05) | OSPI_RESETピンを High に設定 → **リセット解除** |
| 8 | (1ms待機) | - | デバイスリカバリ時間 |
| 9 | 0x40400D14 (PWPR) | PFSWE=0 | PFS レジスタ書き込みを禁止 |
| 10 | 0x40400D14 (PWPR) | BOWI=1 | PFS Write Enable の書き込みを禁止（保護復帰） |

#### 5.1.3 なぜリセットが必要か

OSPI フラッシュデバイスが OPI モード等の特殊モードに入っている場合、デバッガからの通常のSPIコマンドが受け付けられない。フラッシュプログラミング前にハードウェアリセットを行うことで、フラッシュデバイスをデフォルトの SPI モードに戻し、デバッガがフラッシュの消去・書き込みを行えるようにする。

---

## 6. Octo-SPI フラッシュの制御方法

### 6.1 読み書きを可能にするための全体手順

```
┌─────────────────────────────────────────────────────────┐
│ Step 1: ハードウェア準備                                  │
│   ・ボードのSW4を正しく設定（OSPIピン有効化）               │
│   ・OCTACLKが供給されていること                            │
└─────────────────┬───────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────┐
│ Step 2: OSPI_Bドライバオープン                             │
│   R_OSPI_B_Open(&g_ospi0_ctrl, &g_ospi0_cfg)            │
│   ・ペリフェラルレジスタ初期化                              │
│   ・コマンドセット設定                                    │
│   ・SPIモードで開始                                       │
└─────────────────┬───────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────┐
│ Step 3: フラッシュデバイスリセット                          │
│   R_XSPI0->LIOCTL_b.RSTCS0 = 0 → 1                     │
│   ・CS0のリセット信号をトグル                               │
│   ・デバイスがデフォルトSPIモードに復帰                     │
└─────────────────┬───────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────┐
│ Step 4: フラッシュデバイス設定（SPIモードで）               │
│   ・Write Enable → CFG2レジスタ書き込み                   │
│   ・ダミーサイクル設定（CFG2 addr=0x300, data=0x07）       │
└─────────────────┬───────────────────────────────────────┘
                  ▼
┌───────────────────────────────────────────────────────────────┐
│ Step 5: プロトコルモード選択                                    │
│                                                               │
│   [読み出しのみ/高速] → OPIモード (8D-8D-8D DTR)              │
│     ospi_b_set_protocol_to_opi()                              │
│     ・CFG2 addr=0x200: DQSモード設定                          │
│     ・CFG2 addr=0x300: ダミーサイクル設定                      │
│     ・CFG2 addr=0x00:  DTRモード有効化 (data=0x02)            │
│     ・R_OSPI_B_SpiProtocolSet(SPI_FLASH_PROTOCOL_8D_8D_8D)   │
│     → メモリマップド読み出し可能                               │
│     → 書き込みもOPIコマンドで可能                              │
│                                                               │
│   [消去が必要] → SPIモード (1S-1S-1S)                         │
│     ospi_b_set_protocol_to_spi()                              │
│     ・CFG2 addr=0x00: SPIモード設定 (data=0x00)               │
│     ・R_OSPI_B_SpiProtocolSet(SPI_FLASH_PROTOCOL_EXTENDED_SPI)│
│     → 消去コマンド実行可能                                    │
└──────────────────┬────────────────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────────┐
│ Step 6: データ操作                                        │
│                                                         │
│ ■ 読み出し（メモリマップド）                               │
│   ptr = (uint8_t*)0x90xxxxxx;                           │
│   memcpy(buffer, ptr, size);                            │
│                                                         │
│ ■ 書き込み（FSP API）                                    │
│   ① Write Enable必須                                    │
│   ② g_ospi0.p_api->write(ctrl, src, dest_addr, size);  │
│   ③ wait_for_write() で完了待ち                         │
│   ※ 書き込み前にそのセクタの消去が必要                    │
│                                                         │
│ ■ 消去（FSP API）                                       │
│   ① Write Enable必須                                    │
│   ② g_ospi0.p_api->erase(ctrl, addr, size);            │
│   ③ wait_for_write() で完了待ち                         │
│   ※ セクタ (4KB) またはブロック (256KB) 単位              │
└─────────────────────────────────────────────────────────┘
```

### 6.2 プロトコルモードの詳細

本プロジェクトで使用されるプロトコルモード:

| モード | FSP定数 | 説明 | 用途 |
|--------|---------|------|------|
| 1S-1S-1S (Extended SPI) | `SPI_FLASH_PROTOCOL_EXTENDED_SPI` | 標準SPI（1ビット幅） | 初期化、消去、設定変更 |
| 8D-8D-8D (DTR OPI) | `SPI_FLASH_PROTOCOL_8D_8D_8D` | 8ビット幅 DDR（両エッジ） | 高速読み出し、書き込み |

**DTR_MODE の定義** (`common_init.h:80`):
```c
#define DTR_MODE  (1)  // DTR (Double Transfer Rate) モードを使用
```

### 6.3 コマンドセットの対応表

#### 6.3.1 初期プロトコル (1S-1S-1S) のコマンド

**参照**: `hal_data.c:3517-3538`

| 操作 | コマンドコード | 説明 |
|------|--------------|------|
| Read | 0x13 | 4バイトアドレス Read |
| Program | 0x12 | 4バイトアドレス Page Program |
| Write Enable | 0x06 | Write Enable |
| Status Read | 0x05 | Status Register Read |
| Sector Erase (4KB) | 0x21 | 4バイトアドレス Sector Erase |
| Block Erase (256KB) | 0xDC | 4バイトアドレス Block Erase |
| Chip Erase | 0x60 | Chip Erase |

#### 6.3.2 高速プロトコル (8D-8D-8D) のコマンド

**参照**: `hal_data.c:3539-3559`

| 操作 | コマンドコード | ダミーサイクル | 説明 |
|------|--------------|-------------|------|
| Read | 0xEE11 | 6 | xSPI Profile 1.0 Read |
| Program | 0x12ED | 0 | 8D Page Program |
| Write Enable | 0x06F9 | 0 | Write Enable (OPI) |
| Status Read | 0x05FA | 4 | Status Register Read (OPI) |
| Sector Erase (4KB) | 0x21DE | 0 | Sector Erase (OPI) |
| Block Erase (256KB) | 0xDC23 | 0 | Block Erase (OPI) |
| Chip Erase | 0xC738 | 0 | Chip Erase (OPI) |

> OPIモードでは、コマンドコードが2バイト（コマンド + 反転コマンド）で構成される。

### 6.4 書き込みの制約事項

1. **書き込み前の消去が必須**: フラッシュメモリの特性上、書き込む前にセクタ/ブロックを消去する必要がある
2. **ページサイズ制限**: 1回の書き込みは最大64バイト（ページサイズ）
3. **Write Enable必須**: 書き込み/消去のたびにWrite Enableコマンドを発行する必要がある（FSP APIが内部で処理）
4. **完了待ち必須**: 書き込み/消去後は `statusGet` でWIP (Write In Progress) ビットが0になるまで待機する必要がある
5. **アドレスアライメント**: ページ境界をまたぐ書き込みは正しく動作しない場合がある

---

## 7. 関連ファイル一覧（参照パス）

すべてのパスは `quickstart_ek_ra8p1_ep/e2studio/` からの相対パス:

| カテゴリ | ファイルパス |
|---------|------------|
| アプリ | `src/ospi_b_ep.c`, `src/ospi_b_ep.h` |
| アプリ | `src/ospi_b_commands.c`, `src/ospi_b_commands.h` |
| アプリ | `src/ospi_commands.c`, `src/ospi_commands.h` |
| アプリ | `src/menu_ext.c`, `src/menu_ext.h` |
| アプリ | `src/common_init.h` |
| データ | `src/images/backgrounds/welcome_screen_1024x600.c` |
| データ | `src/images/backgrounds/user_led_background_1024x600.c` |
| FSPドライバ | `ra/fsp/inc/instances/r_ospi_b.h` |
| FSPドライバ | `ra/fsp/src/r_ospi_b/r_ospi_b.c` |
| FSP API | `ra/fsp/inc/api/r_spi_flash_api.h` |
| BSP | `ra/fsp/src/bsp/mcu/all/bsp_ospi_b.c`, `bsp_ospi_b.h` |
| BSP | `ra/fsp/src/bsp/mcu/ra8p1/bsp_feature.h` |
| 自動生成 | `ra_gen/hal_data.c`, `ra_gen/hal_data.h` |
| 自動生成 | `ra_gen/bsp_clock_cfg.h` |
| 自動生成 | `ra_cfg/fsp_cfg/r_ospi_b_cfg.h` |
| 自動生成 | `ra_cfg/fsp_cfg/bsp/bsp_pin_cfg.h` |
| リンカ | `script/fsp.ld` |
| デバッガ | `script/RA8x1_Reset_OSPI.JLinkScript` |

---

## 8. 発見事項と注意点

### 8.1 設計上の特徴

1. **2つのコマンドテーブルの共存**: `ospi_b_commands.c` (初期化用) と `ospi_commands.c` (テスト/拡張用) の2つのコマンドテーブルが存在する。前者は `ospi_b_ep.c` で使用され、後者は `menu_ext.c` のテスト機能から参照可能な拡張版
2. **バイトスワップの必要性**: SPIモードで書き込まれたデータをOPIモードで読み出す際にバイトペアの反転が発生するため、ソフトウェアでの補正が必要
3. **パフォーマンステスト領域の配置**: テスト領域 (0x93F00000) は CS1 の末尾近くに配置され、画像データ領域と重ならないよう設計されている

### 8.2 コードの注意点

1. **エラーハンドリングの一部がコメントアウト**: `ospi_b_ep.c` の多くの `APP_ERR_RETURN` マクロがコメントアウトされており、エラー発生時にも処理が継続する箇所がある
2. **DTR_MODE設定の冗長性**: `ospi_b_set_protocol_to_opi()` の `menu_ext.c:218-224` で `#if DTR_MODE` による条件分岐の後に `transfer.data = 0x2U; /* DTR */` が無条件に上書きされており、条件分岐が事実上無効化されている
3. **DOTF (Decrypted On-The-Fly) は無効**: 設定としてはDOTF（暗号化データの復号読み出し）の構成が定義されているが、`OSPI_B_CFG_DOTF_SUPPORT_ENABLE = 0` で無効化されている
