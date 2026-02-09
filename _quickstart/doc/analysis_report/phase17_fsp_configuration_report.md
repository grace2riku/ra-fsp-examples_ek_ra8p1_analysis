# Phase 17: FSP設定解析レポート

## 1. 概要

本レポートでは、EK-RA8P1 _quickstartサンプルプログラムにおけるFSP（Flexible Software Package）コンフィギュレータの設定内容を解析し、プロジェクト新規作成時のデフォルトから変更されている設定を一覧化する。

### 解析対象ファイル
| ファイル | 説明 |
|----------|------|
| `e2studio/configuration.xml` | FSPメイン設定ファイル（306KB） |
| `e2studio/ra_cfg.txt` | FSP設定テキストサマリ（214KB） |
| `e2studio/ra_gen/` | FSP自動生成コード |
| `e2studio/ra_cfg/fsp_cfg/` | FSP生成設定ヘッダ |

### プロジェクト基本情報
| 項目 | 設定値 |
|------|--------|
| ボード | EK-RA8P1 |
| MCU | R7KA8P1KFLCAC (Cortex-M85) |
| RTOS | FreeRTOS |
| FSPバージョン | 6.3.0 |
| ツールチェーン | GCC ARM Embedded 13.2.1 |

---

## 2. プロジェクト基本設定（デフォルトからの変更）

### 2.1 RTOS選択

| 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|-------------------|-------------|-------------|--------------|----------------------|
| BSP > General > RTOS | `No RTOS` → **`FreeRTOS`** | マルチスレッド処理（表示、カメラ、メニュー等を並行動作させるため） | `ra_gen/common_data.c` (スレッド生成コード), `ra_cfg/aws/FreeRTOSConfig.h` | 全スレッドエントリ関数: `display_thread_entry.c`, `camera_thread_entry.c`, `main_menu_thread_entry.c`, `blinky_thread_entry.c` 等 |

---

## 3. BSP設定（RA Common）

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | BSP > RA Common > Main stack size | デフォルト `0x400` → **`0x400`** (1024B) | デフォルト値を維持（HALコンテキスト用） | `ra_gen/vector_data.c` (スタック定義) | スタートアップコード |
| 2 | BSP > RA Common > Heap size | デフォルト `0x1000` → **`0x200`** (512B) | FreeRTOSヒープ（0x80000）を使用するため、Cヒープは最小限に抑制 | `ra_gen/common_data.c` | `malloc()`/`free()`使用箇所（ほぼ未使用） |
| 3 | BSP > RA Common > Parameter checking | デフォルト `Default (BSP)` → **`Disabled`** | リリースビルドでのパフォーマンス最適化 | `ra_cfg/fsp_cfg/bsp/bsp_cfg.h` → `BSP_CFG_PARAM_CHECKING_ENABLE (0)` | 全FSPドライバのAPI呼び出し時 |
| 4 | BSP > RA Common > Assert Failures | デフォルト: Return FSP_ERR_ASSERTION | エラー時にFSP_ERR_ASSERTIONを返すよう設定 | `ra_cfg/fsp_cfg/bsp/bsp_cfg.h` | FSP API内部のアサーション処理 |
| 5 | BSP > RA Common > MCU Vcc | **3300 mV** | EK-RA8P1ボードの電源電圧に合わせた設定 | `ra_cfg/fsp_cfg/bsp/bsp_cfg.h` → `BSP_CFG_MCU_VCC_MV (3300)` | ADC変換精度計算等 |

---

## 4. BSP設定（RA8P1 Family）

### 4.1 SDRAM設定

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | BSP > RA8P1 Family > SDRAM > SDRAM Support | `Disabled` → **`Enabled`** | 外部SDRAM (64MB) をフレームバッファ・ビデオバッファ用に使用 | `ra/fsp/src/bsp/mcu/ra8p1/bsp_sdram.c` (SDRAM初期化コード) | フレームバッファ `fb_background`, `fb_foreground`, ビデオバッファ `vin_image_buffer_*` を`.sdram_noinit`セクションに配置 |
| 2 | BSP > RA8P1 Family > SDRAM > tRAS | **6 cycles** | 搭載SDRAMデバイスのタイミング仕様に合致 | `ra/fsp/src/bsp/mcu/ra8p1/bsp_sdram.c` | SDRAM初期化シーケンス |
| 3 | BSP > RA8P1 Family > SDRAM > tRCD | **3 cycles** | 搭載SDRAMデバイスのタイミング仕様に合致 | 同上 | 同上 |
| 4 | BSP > RA8P1 Family > SDRAM > tRP | **3 cycles** | 搭載SDRAMデバイスのタイミング仕様に合致 | 同上 | 同上 |
| 5 | BSP > RA8P1 Family > SDRAM > tWR | **2 cycles** | 搭載SDRAMデバイスのタイミング仕様に合致 | 同上 | 同上 |
| 6 | BSP > RA8P1 Family > SDRAM > tCL | **3 cycles** (CASレイテンシ) | 搭載SDRAMデバイスのタイミング仕様に合致 | 同上 | 同上 |
| 7 | BSP > RA8P1 Family > SDRAM > tRFC | **937 cycles** | オートリフレッシュサイクル時間 | 同上 | 同上 |
| 8 | BSP > RA8P1 Family > SDRAM > Bus Width | **32-bit** | SDRAMバス幅をフル活用 | 同上 | 同上 |
| 9 | BSP > RA8P1 Family > SDRAM > Address Multiplex Shift | **9-bit shift** | SDRAMアドレスマルチプレクス設定 | 同上 | 同上 |
| 10 | BSP > RA8P1 Family > SDRAM > Continuous Access Mode | **Enabled** | バースト転送で高速アクセスを実現 | 同上 | 同上 |

### 4.2 キャッシュ設定

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | BSP > RA8P1 Family > Cache settings > Data cache | **Disabled** | SDRAMとGLCDCのDMA転送との整合性確保（キャッシュコヒーレンシ問題の回避） | `ra_cfg/fsp_cfg/bsp/bsp_cfg.h` | システム全体のメモリアクセス |

---

## 5. クロック設定

### 5.1 クロックツリー概要

```
XTAL (24MHz) ──┬── PLL1 (/3 → x250 = 2000MHz) ──┬── PLL1P (/2) = 1000MHz → システムクロック源
                │                                   ├── PLL1Q (/6) ≈ 333MHz
                │                                   └── PLL1R (/5) = 400MHz
                │
                └── PLL2 (/3 → x300 = 2400MHz) ──┬── PLL2P (/2) = 1200MHz
                                                   ├── PLL2Q (/8) = 300MHz → OCTACLK源
                                                   └── PLL2R (/5) = 480MHz → SCICLK,GPTCLK,LCDCLK等
```

### 5.2 主要クロック周波数一覧

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | Clocks > PLL Src | **XTAL** | 高精度な24MHzクリスタルを基準に使用 | `ra_gen/bsp_clock_cfg.h` | BSPクロック初期化 (`bsp_clock_init()`) |
| 2 | Clocks > PLL Div | **/3** | XTAL 24MHz → 8MHz (PLL入力周波数) | 同上 | 同上 |
| 3 | Clocks > PLL Mul | **x250.00** | 8MHz × 250 = 2000MHz (PLL1 VCO) | 同上 | 同上 |
| 4 | Clocks > PLL1P Div | **/2** | 2000MHz / 2 = **1000MHz** (CPU/システムクロック源) | 同上 | 同上 |
| 5 | Clocks > PLL1Q Div | **/6** | 2000MHz / 6 ≈ **333MHz** | 同上 | 同上 |
| 6 | Clocks > PLL1R Div | **/5** | 2000MHz / 5 = **400MHz** | 同上 | 同上 |
| 7 | Clocks > PLL2 Src | **XTAL** | PLL2も同じクリスタルを基準に使用 | 同上 | 同上 |
| 8 | Clocks > PLL2 Div | **/3** | 24MHz / 3 = 8MHz | 同上 | 同上 |
| 9 | Clocks > PLL2 Mul | **x300.00** | 8MHz × 300 = 2400MHz (PLL2 VCO) | 同上 | 同上 |
| 10 | Clocks > PLL2P Div | **/2** | 2400MHz / 2 = **1200MHz** | 同上 | 同上 |
| 11 | Clocks > PLL2Q Div | **/8** | 2400MHz / 8 = **300MHz** | 同上 | 同上 |
| 12 | Clocks > PLL2R Div | **/5** | 2400MHz / 5 = **480MHz** | 同上 | 同上 |

### 5.3 クロックドメイン分周設定

| # | 設定メニューの場所 | 設定値 | 結果周波数 | 設定変更理由 | FSP出力コード |
|---|-------------------|--------|-----------|-------------|--------------|
| 1 | Clocks > Clock Src | **PLL1P** | - | 最高速クロック源を選択 | `ra_gen/bsp_clock_cfg.h` |
| 2 | Clocks > CPUCLK Div | **/1** | **1000MHz** | CPU0を最高速度で動作 | 同上 |
| 3 | Clocks > CPUCLK1 Div | **/4** | **250MHz** | CPU1（サブコア）は低速で十分 | 同上 |
| 4 | Clocks > NPUCLK Div | **/2** | **500MHz** | NPU（ニューラル処理ユニット）クロック | 同上 |
| 5 | Clocks > ICLK Div | **/4** | **250MHz** | 内部バスクロック | 同上 |
| 6 | Clocks > PCLKA Div | **/8** | **125MHz** | ペリフェラルクロックA（高速ペリフェラル用） | 同上 |
| 7 | Clocks > PCLKB Div | **/16** | **62.5MHz** | ペリフェラルクロックB（中速ペリフェラル用） | 同上 |
| 8 | Clocks > PCLKC Div | **/8** | **125MHz** | ペリフェラルクロックC | 同上 |
| 9 | Clocks > PCLKD Div | **/4** | **250MHz** | ペリフェラルクロックD（GPT等の高速タイマ用） | 同上 |
| 10 | Clocks > PCLKE Div | **/4** | **250MHz** | ペリフェラルクロックE | 同上 |
| 11 | Clocks > BCLK Div | **/8** | **125MHz** | 外部バスクロック（SDRAM用） | 同上 |
| 12 | Clocks > SDCLK | **Enabled** | 125MHz | SDRAM用クロック出力有効化 | 同上 |
| 13 | Clocks > EBCLK Div | **/2** | **62.5MHz** | 外部バスクロック分周 | 同上 |

### 5.4 周辺クロック設定

| # | 設定メニューの場所 | 設定値 | 結果周波数 | 設定変更理由 | 利用モジュール |
|---|-------------------|--------|-----------|-------------|---------------|
| 1 | Clocks > SCICLK | **Enabled**, Src: PLL2R, Div: /5 | **96MHz** | SCI UART通信用クロック | `g_jlink_console` (SCI8 UART 115200bps) |
| 2 | Clocks > GPTCLK | **Enabled**, Src: PLL2R, Div: /2 | **240MHz** | GPTタイマ高精度動作用 | `g_timer_camera_xclk` (カメラXCLK 24MHz生成), LED PWMタイマ群 |
| 3 | Clocks > LCDCLK | **Enabled**, Src: PLL2R, Div: /2 | **240MHz** | GLCDCピクセルクロック生成用 | `g_plcd_display` (GLCDC, 1/7分周で約34.3MHz) |
| 4 | Clocks > OCTACLK | **Enabled**, Src: PLL2Q, Div: /5 | **60MHz** | Octo-SPIフラッシュアクセス用 | `g_ospi0` (OSPI_B) |
| 5 | Clocks > ADCCLK | **Enabled**, Src: PLL2R, Div: /4 | **120MHz** | ADC高速サンプリング用 | `g_adc` (ADC_B) |
| 6 | Clocks > USB60CLK | **Enabled**, Src: PLL2R, Div: /8 | **60MHz** | USB Full-Speed用 | USB機能（本プロジェクトでは限定的使用） |
| 7 | Clocks > SPICLK | **Disabled** | - | SPI未使用 | - |
| 8 | Clocks > CANFDCLK | **Disabled** | - | CAN FD未使用 | - |
| 9 | Clocks > I3CCLK | **Disabled** | - | I3C未使用 | - |
| 10 | Clocks > USBCLK | **Disabled** | - | USB High-Speed未使用 | - |

---

## 6. FreeRTOSスレッド設定

### 6.1 スレッド一覧

| # | 設定メニューの場所 | スレッド名 | スタック | 優先度 | メモリ | 設定変更理由 | FSP出力コード | 利用コード |
|---|-------------------|-----------|---------|--------|--------|-------------|--------------|-----------|
| 1 | Threads > Display Thread | Display Thread | 2048B | 2 | Static | LCD表示処理用スレッド | `ra_gen/display_thread.c` | `src/display_thread_entry.c` |
| 2 | Threads > Camera Thread | Camera Thread | 10240B | 3 | Static | カメラ映像取得・処理用（大量バッファ操作のため大スタック） | `ra_gen/camera_thread.c` | `src/camera_thread_entry.c` |
| 3 | Threads > main_menu_thread | main_menu_thread | 8096B | 1 | Static | メインメニュー・UIロジック（最低優先度で常時動作） | `ra_gen/main_menu_thread.c` | `src/main_menu_thread_entry.c` |
| 4 | Threads > Brd Mon Thread | board_mon_thread | 1024B | 8 | Static | ボード監視（温度、電圧等） | `ra_gen/board_mon_thread.c` | `src/board_mon_thread_entry.c` |
| 5 | Threads > TP Thread | TP Thread | 4096B | 15 | Static | タッチパネル入力処理（最高優先度で即応性確保） | `ra_gen/tp_thread.c` | `src/tp_thread_entry.c` |
| 6 | Threads > Blinky Thread | Blinky Thread | 512B | 15 | Static | LED点滅（最高優先度、軽量処理） | `ra_gen/blinky_thread.c` | `src/blinky_thread_entry.c` |

### 6.2 RTOS同期オブジェクト

| # | 設定メニューの場所 | オブジェクト名 | 種類 | 設定変更理由 | FSP出力コード | 利用コード |
|---|-------------------|---------------|------|-------------|--------------|-----------|
| 1 | Objects > Event Group | `g_update_console_event` | Event Group (Static) | コンソール更新イベント通知 | `ra_gen/common_data.c` | `src/jlink_console.c` 等 |
| 2 | Objects > Event Group | `g_update_display_event` | Event Group (Static) | 表示更新イベント通知 | `ra_gen/common_data.c` | `src/display_thread_entry.c` 等 |
| 3 | Objects > Binary Semaphore | `g_irq_binary_semaphore` | Binary Semaphore (Static) | 外部割り込み同期用 | `ra_gen/common_data.c` | `src/board_cfg_switch.c` 等 |
| 4 | Objects > Event Group | `g_i2c_event_group` | Event Group (Static) | I2C通信完了待ち同期 | `ra_gen/common_data.c` | `src/board_i2c_master.c` 等 |

---

## 7. モジュール（ドライバ）設定

### 7.1 HALコンテキスト（全スレッド共通）のモジュール

#### 7.1.1 I/O Port (r_ioport)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | HAL > g_ioport > (自動追加) | デフォルトで追加 | ピン制御は必須モジュール | `ra_gen/common_data.c`, `ra_gen/pin_data.c` | `src/common_init.c` (GPIO初期化), 各スレッドでのピン制御 |

#### 7.1.2 FreeRTOS Port (rm_freertos_port)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | HAL > FreeRTOS Port | RTOS選択時に自動追加 | FreeRTOSとRA MCUのブリッジ | `ra_gen/common_data.c` | FreeRTOSカーネル動作全体 |

#### 7.1.3 FreeRTOS Heap 4

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | HAL > FreeRTOS Heap 4 | Heap管理方式としてHeap 4を選択 | フラグメンテーション対策付きメモリ管理 | `ra/aws/FreeRTOS/FreeRTOS/Source/portable/MemMang/heap_4.c` | `pvPortMalloc()`/`vPortFree()` 経由の動的メモリ確保 |

#### 7.1.4 I2C Master (r_iic_master)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | HAL > g_board_i2c_master > Name | **g_board_i2c_master** | ボード上のI2Cデバイス通信用 | `ra_gen/hal_data.c` (IIC設定構造体) | `src/board_i2c_master.c` |
| 2 | HAL > g_board_i2c_master > Channel | **1** | IIC1チャネルを使用（ピンP511/P512） | 同上 | 同上 |
| 3 | HAL > g_board_i2c_master > Rate | **Standard (100kbps)** | タッチコントローラ（FT5316）・GreenPAK通信 | 同上 | `src/board_i2c_master.c`, `src/board_greenpak.c` |
| 4 | HAL > g_board_i2c_master > Interrupt Priority | **Level 4** | I2C割り込みの応答性確保 | `ra_gen/vector_data.c` | 割り込みハンドラ |

#### 7.1.5 GPTタイマ群

| # | インスタンス名 | チャネル | モード | 周期/周波数 | 設定変更理由 | FSP出力コード | 利用コード |
|---|---------------|---------|--------|-----------|-------------|--------------|-----------|
| 1 | `g_timer_camera_xclk` | CH12 | Periodic | 24MHz (PWM出力) | カメラOV5640のXCLK供給 | `ra_gen/hal_data.c` | `src/camera_thread_entry.c` |
| 2 | `g_memory_performance` | CH0 | Periodic | 17.18秒周期 | OSPI性能測定用タイマ | `ra_gen/hal_data.c` | `src/menu_ext.c` (ospi_performance_test) |
| 3 | `g_gpt_blue` | CH6 | PWM | 2.5MHz (Duty 10%) | 青色LEDのPWM輝度制御 | `ra_gen/hal_data.c` | `src/common_init.c` |
| 4 | `g_blinker_blue` | CH1 | Periodic | 1Hz (1秒周期) | LED点滅制御用 | `ra_gen/hal_data.c` | `src/blinky_thread_entry.c` |
| 5 | `led_pulse_timer` | CH5 | Periodic | ≈1160Hz | LED脈動アニメーション用 | `ra_gen/hal_data.c` | `src/common_init.c` |
| 6 | `led_brightness_timer` | CH3 | Periodic | 5kHz | LED輝度制御PWMデューティ更新用 | `ra_gen/hal_data.c` | `src/common_init.c` |

#### 7.1.6 UART (r_sci_b_uart)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | HAL > g_jlink_console > Name | **g_jlink_console** | J-Link仮想COMポート経由のデバッグコンソール | `ra_gen/hal_data.c` | `src/jlink_console.c` |
| 2 | HAL > g_jlink_console > Channel | **8** (SCI8) | EK-RA8P1ボード上のJ-Link接続ピン | 同上 | 同上 |
| 3 | HAL > g_jlink_console > Baud Rate | **115200** | 標準デバッグ通信速度 | 同上 | 同上 |
| 4 | HAL > g_jlink_console > Data Bits | **8 bits** | 標準設定 | 同上 | 同上 |
| 5 | HAL > g_jlink_console > Parity | **None** | 標準設定 | 同上 | 同上 |
| 6 | HAL > g_jlink_console > Stop Bits | **1** | 標準設定 | 同上 | 同上 |
| 7 | HAL > g_jlink_console > Flow Control | **Hardware RTS** | 通信安定性の確保 | 同上 | 同上 |

#### 7.1.7 OSPI (r_ospi_b)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | HAL > g_ospi0 > Name | **g_ospi0** | 外部Octo-SPIフラッシュ（Macronix MX25LW51245G）アクセス | `ra_gen/hal_data.c` | `src/menu_ext.c` (ospi_performance_test) |
| 2 | HAL > g_ospi0 > Unit | **OSPI_B0** | OSPI_Bユニット使用 | 同上 | 同上 |
| 3 | HAL > g_ospi0 > Chip Select | **CS1** | フラッシュデバイスはCS1に接続 | 同上 | 同上 |
| 4 | HAL > g_ospi0 > Initial SPI Protocol | **1S-1S-1S** (SPI) | 起動時は標準SPIモードで初期化 | 同上 | 同上 |
| 5 | HAL > g_ospi0 > High-Speed Protocol | **8D-8D-8D** (Dual Data Rate OPI) | 高速データ転送モード | 同上 | 同上 |
| 6 | HAL > g_ospi0 > Address Bytes | **4** | 64MBフラッシュに4バイトアドレス必要 | 同上 | 同上 |
| 7 | HAL > g_ospi0 > Sector Erase Size | **4KB** | 最小消去単位 | 同上 | 同上 |
| 8 | HAL > g_ospi0 > Block Erase Size | **256KB** | ブロック消去単位 | 同上 | 同上 |
| 9 | Config > OSPI_B > Prefetch Function | **Enabled** | 読み出し性能向上 | `ra_cfg/fsp_cfg/r_ospi_b_cfg.h` → `OSPI_B_CFG_PREFETCH_FUNCTION (1)` | OSPI読み出しアクセス全般 |
| 10 | Config > OSPI_B > Combination Function | **64 BYTE** | 結合読み出し機能有効化 | `ra_cfg/fsp_cfg/r_ospi_b_cfg.h` → `OSPI_B_CFG_COMBINATION_FUNCTION (OSPI_B_COMBINATION_FUNCTION_64BYTE)` | 同上 |

#### 7.1.8 ADC (r_adc_b)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | HAL > g_adc > Name | **g_adc** | ボード上のアナログセンサ値取得 | `ra_gen/hal_data.c` | `src/board_mon_thread_entry.c` |
| 2 | HAL > g_adc > Virtual Channels | **CH0-CH21** (複数チャネル) | 温度センサ、電源電圧モニタ等 | 同上 | 同上 |
| 3 | HAL > g_adc > Conversion Format | **12-bit** | 十分な分解能 | 同上 | 同上 |

#### 7.1.9 External IRQ (r_icu)

| # | インスタンス名 | IRQチャネル | トリガ | 設定変更理由 | FSP出力コード | 利用コード |
|---|---------------|-----------|--------|-------------|--------------|-----------|
| 1 | `g_external_irq13ds` | IRQ13-DS | - | ユーザースイッチSW1入力 | `ra_gen/hal_data.c`, `ra_gen/vector_data.c` | `src/board_cfg_switch.c` |
| 2 | `g_external_irq12ds` | IRQ12-DS | - | ユーザースイッチSW2入力 | 同上 | 同上 |
| 3 | `g_external_irq19` | IRQ19 | - | GreenPAK割り込み入力 | 同上 | `src/board_greenpak.c` |

### 7.2 Display Threadのモジュール

#### 7.2.1 Graphics LCD (r_glcdc)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | Display Thread > g_plcd_display > Name | **g_plcd_display** | LCD表示制御 | `ra_gen/display_thread.c` | `src/display_thread_entry.c` |
| 2 | 同 > Graphics Layer 1 > Enabled | **Yes** | 背景レイヤー（カメラ映像表示用） | 同上 | 同上 |
| 3 | 同 > Graphics Layer 1 > Size | **768 x 450** | カメラ映像の表示サイズ | 同上 | 同上 |
| 4 | 同 > Graphics Layer 1 > Position | **(128, 75)** | 画面中央にオフセット配置 | 同上 | 同上 |
| 5 | 同 > Graphics Layer 1 > Color format | **RGB565 (16-bit)** | メモリ効率とカメラ出力形式の整合 | 同上 | 同上 |
| 6 | 同 > Graphics Layer 1 > Framebuffer name | **fb_background** | 背景フレームバッファ名 | 同上 | `src/display_thread_entry.c` |
| 7 | 同 > Graphics Layer 1 > Number of framebuffers | **1** | シングルバッファ（カメラ映像は直書き） | 同上 | 同上 |
| 8 | 同 > Graphics Layer 1 > Section | **.sdram_noinit** | SDRAMにフレームバッファ配置 | 同上 | リンカスクリプト |
| 9 | 同 > Graphics Layer 2 > Enabled | **Yes** | 前景レイヤー（UI/オーバーレイ表示用） | 同上 | 同上 |
| 10 | 同 > Graphics Layer 2 > Size | **1024 x 600** | LCDフル解像度 | 同上 | 同上 |
| 11 | 同 > Graphics Layer 2 > Position | **(0, 0)** | 画面全体をカバー | 同上 | 同上 |
| 12 | 同 > Graphics Layer 2 > Color format | **ARGB4444 (16-bit)** | アルファ付き（透過表示対応） | 同上 | Dave/2D描画エンジン |
| 13 | 同 > Graphics Layer 2 > Framebuffer name | **fb_foreground** | 前景フレームバッファ名 | 同上 | `src/display_thread_entry.c` |
| 14 | 同 > Graphics Layer 2 > Number of framebuffers | **2** | ダブルバッファ（ティアリング防止） | 同上 | 同上 |
| 15 | 同 > Graphics Layer 2 > Section | **.sdram_noinit** | SDRAMにフレームバッファ配置 | 同上 | リンカスクリプト |
| 16 | 同 > Output > Horizontal total | **1334 cycles** | LCDパネルのタイミング仕様 | 同上 | GLCDC HW |
| 17 | 同 > Output > Horizontal active | **1024 cycles** | LCDパネルの有効表示幅 | 同上 | 同上 |
| 18 | 同 > Output > Horizontal back porch | **300 cycles** | LCDパネルのタイミング仕様 | 同上 | 同上 |
| 19 | 同 > Output > Vertical total | **780 lines** | LCDパネルのタイミング仕様 | 同上 | 同上 |
| 20 | 同 > Output > Vertical active | **600 lines** | LCDパネルの有効表示高さ | 同上 | 同上 |
| 21 | 同 > Output > Vertical back porch | **30 lines** | LCDパネルのタイミング仕様 | 同上 | 同上 |
| 22 | 同 > Output > Color format | **24bits RGB888** | LCDパネルのRGBインターフェース仕様 | 同上 | 同上 |
| 23 | 同 > TCON > Panel clock division | **1/7** | LCDCLKから約34.3MHzピクセルクロック生成 | 同上 | 同上 |
| 24 | 同 > Callback | **glcdc_vsync_isr** | VSYNC割り込みでフレーム同期 | 同上 | `src/display_thread_entry.c` |
| 25 | 同 > Interrupt Priority | **Level 12** | 表示割り込み優先度 | `ra_gen/vector_data.c` | 割り込みハンドラ |

#### 7.2.2 D/AVE 2D Port Interface (r_drw)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | Display Thread > D/AVE 2D > Device Handle | **gp_d2_handle** | 2Dグラフィックスアクセラレータ | `ra_gen/display_thread.c` | `src/display_thread_entry.c` (Dave/2D描画) |
| 2 | Display Thread > D/AVE 2D > Interrupt Priority | **Level 2** | 描画処理の高優先度割り込み | `ra_gen/vector_data.c` | 描画完了割り込み |

### 7.3 Camera Threadのモジュール

#### 7.3.1 Video Input (r_vin)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | Camera Thread > g_vin0 > Name | **g_vin0** | カメラ映像キャプチャ | `ra_gen/camera_thread.c` | `src/camera_thread_entry.c` |
| 2 | 同 > Interface Format | **YCbCr-422, 8-bit** | OV5640カメラの出力フォーマット | 同上 | 同上 |
| 3 | 同 > Capture Image Size | **768 x 450** | 背景レイヤーに合わせた解像度 | 同上 | 同上 |
| 4 | 同 > Color Space Conversion | **Enabled** | YCbCr→RGB自動変換 | 同上 | 同上 |
| 5 | 同 > Dithering | **Enabled (Cumulative Addition)** | 色品質の向上 | 同上 | 同上 |
| 6 | 同 > Output Buffers | **vin_image_buffer_1/2/3** (.sdram_noinit) | トリプルバッファでティアリング防止 | 同上 | 同上 |
| 7 | 同 > Interpolation | **Bilinear** | 高品質リサイズ | 同上 | 同上 |

#### 7.3.2 MIPI CSI (r_mipi_csi) + MIPI PHY (r_mipi_phy)

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | Camera Thread > MIPI CSI > Name | **g_mipi_csi0** | MIPI CSI-2カメラインターフェース | `ra_gen/camera_thread.c` | `src/camera_thread_entry.c` |
| 2 | Camera Thread > MIPI CSI > Data Lanes | **2 lanes** | OV5640の2レーンMIPI出力 | 同上 | 同上 |
| 3 | Camera Thread > MIPI PHY > PLL MHz | **1000.00** | MIPI PHY PLL周波数 | 同上 | 同上 |
| 4 | Camera Thread > MIPI PHY > LP Divisor | **5** | ローパワーモード分周 | 同上 | 同上 |

---

## 8. FreeRTOS設定

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード | FSPを利用しているところ |
|---|-------------------|-------------|-------------|--------------|----------------------|
| 1 | FreeRTOS > Total Heap Size | デフォルト `0x10000` → **`0x80000`** (512KB) | 大量の動的メモリ確保（タスクスタック、Dave/2Dバッファ等） | `ra_cfg/aws/FreeRTOSConfig.h` → `configTOTAL_HEAP_SIZE` | `pvPortMalloc()`による全動的確保 |
| 2 | FreeRTOS > Tick Rate Hz | **1000** | 1ms分解能のタイムスライス | 同上 → `configTICK_RATE_HZ` | `vTaskDelay()`, `xTaskGetTickCount()` |
| 3 | FreeRTOS > Max Priorities | デフォルト `5` → **`16`** | 6スレッド + システムタスクの優先度を細かく制御 | 同上 → `configMAX_PRIORITIES` | スレッド優先度設定 (1-15) |
| 4 | FreeRTOS > Max Task Name Len | デフォルト `16` → **`32`** | 長いスレッド名に対応（例: "main_menu_thread"） | 同上 → `configMAX_TASK_NAME_LEN` | デバッグ時のタスク名表示 |
| 5 | FreeRTOS > Use Idle Hook | **Enabled** | アイドル時のカスタム処理 | 同上 → `configUSE_IDLE_HOOK` | `vApplicationIdleHook()` |
| 6 | FreeRTOS > Support Static Allocation | **Enabled** | 全スレッドをスタティックアロケーション | 同上 → `configSUPPORT_STATIC_ALLOCATION` | `xTaskCreateStatic()` |
| 7 | FreeRTOS > Support Dynamic Allocation | **Enabled** | 動的メモリ確保も併用 | 同上 → `configSUPPORT_DYNAMIC_ALLOCATION` | Heap 4経由の動的確保 |
| 8 | FreeRTOS > Use Timers | **Enabled** | ソフトウェアタイマ機能の有効化 | 同上 → `configUSE_TIMERS` | タイマベースの遅延処理 |
| 9 | FreeRTOS > Timer Task Priority | **3** | タイマタスクの優先度 | 同上 → `configTIMER_TASK_PRIORITY` | FreeRTOSタイマデーモン |
| 10 | FreeRTOS > Use Counting Semaphores | **Enabled** | カウンティングセマフォの使用 | 同上 → `configUSE_COUNTING_SEMAPHORES` | スレッド間同期 |
| 11 | FreeRTOS > Use Time Slicing | **Disabled** | 同一優先度のラウンドロビン無効（明示的な優先度制御を優先） | 同上 → `configUSE_TIME_SLICING` | スレッドスケジューリング |
| 12 | FreeRTOS > Use Mutexes | **Disabled** | ミューテックス未使用（Event Group/Semaphoreで代替） | 同上 → `configUSE_MUTEXES` | - |
| 13 | FreeRTOS > Check for Stack Overflow | **Disabled** | パフォーマンス優先（リリースビルド想定） | 同上 → `configCHECK_FOR_STACK_OVERFLOW` | - |
| 14 | FreeRTOS > Thread Local Storage Pointers | デフォルト `0` → **`5`** | スレッドローカルストレージの使用 | 同上 → `configNUM_THREAD_LOCAL_STORAGE_POINTERS` | TLS経由のスレッド固有データ |
| 15 | FreeRTOS > Library Max Syscall Interrupt Priority | **Priority 1** | FreeRTOS管理下割り込みの最高優先度 | 同上 → `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY` | ISR内のFreeRTOS API呼び出し |

---

## 9. モジュールごとのドライバ設定（Config）

| # | 設定メニューの場所 | 設定変更内容 | 設定変更理由 | FSP出力コード |
|---|-------------------|-------------|-------------|--------------|
| 1 | Stacks > Config > GPT > Output Support | **Enabled** | GPTのPWM出力機能を有効化（LED制御、カメラXCLK） | `ra_cfg/fsp_cfg/r_gpt_cfg.h` → `GPT_CFG_OUTPUT_SUPPORT_ENABLE (1)` |
| 2 | Stacks > Config > GLCDC > Color Correction | **Off** | 色補正機能未使用（パフォーマンス優先） | `ra_cfg/fsp_cfg/r_glcdc_cfg.h` → `GLCDC_CFG_COLOR_CORRECTION_ENABLE (false)` |
| 3 | Stacks > Config > SCI_B_UART > FIFO Support | **Disabled** | FIFOバッファ未使用 | `ra_cfg/fsp_cfg/r_sci_b_uart_cfg.h` |
| 4 | Stacks > Config > SCI_B_UART > DTC Support | **Disabled** | DTC転送未使用 | 同上 |
| 5 | Stacks > Config > IIC Master > DTC Enable | **Disabled** | DTC転送未使用 | `ra_cfg/fsp_cfg/r_iic_master_cfg.h` → `IIC_MASTER_CFG_DTC_ENABLE (0)` |
| 6 | Stacks > Config > IIC Master > 10-bit Address | **Disabled** | 7ビットアドレスモードのみ使用 | `ra_cfg/fsp_cfg/r_iic_master_cfg.h` → `IIC_MASTER_CFG_ADDR_MODE_10_BIT_ENABLE (0)` |
| 7 | Stacks > Config > OSPI_B > DMAC Support | **Disabled** | DMA転送未使用 | `ra_cfg/fsp_cfg/r_ospi_b_cfg.h` → `OSPI_B_CFG_DMAC_SUPPORT_ENABLE (0)` |
| 8 | Stacks > Config > OSPI_B > XIP Support | **Disabled** | XIP（Execute In Place）未使用 | `ra_cfg/fsp_cfg/r_ospi_b_cfg.h` → `OSPI_B_CFG_XIP_SUPPORT_ENABLE (0)` |
| 9 | Stacks > Config > OSPI_B > Auto Calibration | **Disabled** | 自動キャリブレーション未使用 | `ra_cfg/fsp_cfg/r_ospi_b_cfg.h` → `OSPI_B_CFG_AUTOCALIBRATION_SUPPORT_ENABLE (0)` |
| 10 | Stacks > Config > D/AVE 2D > D-List Indirect | **Enabled** | 間接ディスプレイリストモード | `ra_cfg/fsp_cfg/r_drw_cfg.h` → `DRW_CFG_USE_DLIST_INDIRECT (1)` |

---

## 10. 主要ピン設定（デフォルトからの変更）

### 10.1 ペリフェラルピン割り当て一覧

| # | ピン | パッケージ | 機能 | 設定変更理由 | FSP出力コード |
|---|------|----------|------|-------------|--------------|
| 1 | P602 | P2 | SCI0_RXD (UART RX) | J-Linkコンソール受信 | `ra_gen/pin_data.c` |
| 2 | P603 | P1 | SCI0_TXD (UART TX) | J-Linkコンソール送信 | 同上 |
| 3 | P511 | U15 | IIC1_SDA | I2Cデータライン | 同上 |
| 4 | P512 | P13 | IIC1_SCL | I2Cクロックライン | 同上 |
| 5 | P100 | U6 | OSPI0_SIO0 | OSPIデータ0 | 同上 |
| 6 | P101 | R5 | OSPI0_SIO3 | OSPIデータ3 | 同上 |
| 7 | P104 | M6 | OSPI0_CS1 | OSPIチップセレクト | 同上 |
| 8 | P808 | U5 | OSPI0_SCLK | OSPIクロック | 同上 |
| 9 | P501 | R8 | GPT12 GTIOCA (CAM_XCLK) | カメラXCLK (24MHz) | 同上 |
| 10 | P709 | P16 | GPIO Output (CAMERA_RESET) | カメラリセット制御 | 同上 |
| 11 | P606 | N3 | GPIO Output (LCD_RESET) | LCDリセット制御 | 同上 |
| 12 | P805 | P12 | LCD_TCON1 (HSYNC) | LCD水平同期信号 | 同上 |
| 13 | P806 | T14 | LCD_TCON0 (VSYNC) | LCD垂直同期信号 | 同上 |
| 14 | P600 | P3 | GPIO Output (USER_LED_BLUE) | ユーザーLED（青）| 同上 |
| 15 | P000 | U13 | GPIO Input (USER_SW_CFG_INT) | ユーザースイッチ設定 | 同上 |
| 16 | P008 | U11 | IRQ12-DS Input (USER_SW2) | ユーザースイッチ2（IRQ割り込み） | 同上 |
| 17 | P009 | P11 | IRQ13-DS Input (USER_SW1) | ユーザースイッチ1（IRQ割り込み） | 同上 |
| 18 | P112-P115 | - | SDRAM DQ3-DQ6 | SDRAMデータバス | 同上 |
| 19 | PA00-PA04 | - | SDRAM Address | SDRAMアドレスバス | 同上 |
| 20 | MIPI_CLN/CLP | U2/T2 | MIPI CSI Clock Lane | カメラMIPIクロック | 同上 |
| 21 | MIPI_DL0N/DL0P | U1/T1 | MIPI CSI Data Lane 0 | カメラMIPIデータ0 | 同上 |
| 22 | MIPI_DL1N/DL1P | U3/T3 | MIPI CSI Data Lane 1 | カメラMIPIデータ1 | 同上 |

---

## 11. スレッドとモジュールの関係図

```
HAL Context (共通)
├── g_ioport (I/O Port)
├── FreeRTOS Port + Heap 4
├── g_board_i2c_master (IIC1) ─── タッチパネル(FT5316), GreenPAK
├── g_jlink_console (SCI8 UART) ─── デバッグコンソール
├── g_ospi0 (OSPI_B, CS1) ─── 外部フラッシュ(MX25LW51245G)
├── g_memory_performance (GPT CH0) ─── OSPI性能測定
├── g_adc (ADC_B) ─── 温度・電圧モニタ
├── g_external_irq12ds (IRQ12-DS) ─── SW2
├── g_external_irq13ds (IRQ13-DS) ─── SW1
├── g_external_irq19 (IRQ19) ─── GreenPAK
├── g_gpt_blue (GPT CH6) ─── 青LED PWM制御
├── g_blinker_blue (GPT CH1) ─── LED点滅
├── led_pulse_timer (GPT CH5) ─── LED脈動
└── led_brightness_timer (GPT CH3) ─── LED輝度

Display Thread (優先度: 2, Stack: 2048B)
├── g_plcd_display (GLCDC) ─── 1024x600 LCD
│   ├── Layer1: 768x450 RGB565 (fb_background) ─── カメラ映像
│   └── Layer2: 1024x600 ARGB4444 (fb_foreground) ─── UIオーバーレイ
└── D/AVE 2D (r_drw) ─── 2Dアクセラレータ

Camera Thread (優先度: 3, Stack: 10240B)
└── g_vin0 (VIN) ─── 768x450 YCbCr-422→RGB変換
    └── g_mipi_csi0 (MIPI CSI-2) ─── 2レーン
        └── g_mipi_phy0 (MIPI PHY) ─── 1000MHz PLL

main_menu_thread (優先度: 1, Stack: 8096B)
└── (モジュールなし - HALモジュールを共有利用)

board_mon_thread (優先度: 8, Stack: 1024B)
└── (モジュールなし - g_adcをHALコンテキスト経由で使用)

TP Thread (優先度: 15, Stack: 4096B)
└── (モジュールなし - g_board_i2c_masterをHAL経由で使用)

Blinky Thread (優先度: 15, Stack: 512B)
└── (モジュールなし - GPTタイマをHAL経由で使用)
```

---

## 12. FSP設定変更のサマリ

### 新規プロジェクト作成時に必要な主要設定変更

プロジェクトを新規作成する場合（`EK-RA8P1`ボード、`FreeRTOS`選択後）、以下の設定変更が必要となる：

| カテゴリ | 変更項目数 | 重要度 | 説明 |
|---------|-----------|--------|------|
| RTOS選択 | 1 | 必須 | FreeRTOSの選択 |
| BSP RA Common | 3 | 高 | ヒープ縮小、パラメータチェック無効化、VCC設定 |
| BSP SDRAM | 10 | 必須 | SDRAM有効化とタイミング設定（フレームバッファに必須） |
| クロック (PLL) | 12 | 必須 | PLL1/PLL2の設定（全ペリフェラルの動作に必須） |
| クロック (分周) | 13 | 必須 | 各クロックドメインの分周比設定 |
| 周辺クロック | 10 | 高 | SCICLK, GPTCLK, LCDCLK, OCTACLK等の有効化 |
| スレッド | 6 | 高 | 6つのFreeRTOSスレッドの追加 |
| RTOS同期オブジェクト | 4 | 高 | Event Group×3, Binary Semaphore×1 |
| FreeRTOS設定 | 15 | 高 | ヒープ512KB、優先度16段階、スタティックアロケーション等 |
| HALモジュール | 15 | 必須 | I2C, UART, OSPI, GPT×6, ADC, IRQ×3 |
| Display モジュール | 2 | 必須 | GLCDC（2レイヤー詳細設定）, D/AVE 2D |
| Camera モジュール | 3 | 必須 | VIN, MIPI CSI, MIPI PHY |
| ドライバConfig | 10 | 中 | 各ドライバの機能有効/無効設定 |
| ピン設定 | 22+ | 必須 | 全ペリフェラルのピン割り当て |
| **合計** | **約126項目** | - | - |

### 設定の依存関係

```
1. ボード・RTOS選択
   ↓
2. BSP設定 (SDRAM有効化、基本設定)
   ↓
3. クロック設定 (PLL→分周→周辺クロック)
   ↓
4. ピン設定 (ペリフェラルのピン割り当て)
   ↓
5. スレッド追加 + RTOS同期オブジェクト
   ↓
6. モジュール追加 (各スレッドにドライバを紐付け)
   ↓
7. モジュールプロパティ設定 (各ドライバの詳細パラメータ)
   ↓
8. コード生成 (Generate Project Content)
```

---

## 13. 注意事項

1. **FSPバージョン依存**: 本設定はFSP 6.3.0に基づく。バージョンが異なる場合、設定項目やデフォルト値が変わる可能性がある
2. **ボード固有設定**: SDRAM タイミング、ピン割り当て等はEK-RA8P1ボード固有であり、カスタムボードでは変更が必要
3. **LCDパネル設定**: GLCDCのタイミング設定はEK-RA8P1付属のLCDパネル固有値であり、異なるパネルを使用する場合は変更が必要
4. **MIPI PHY設定**: タイミングパラメータ（t_hs_prep, t_clk_zero等）はOV5640カメラモジュール固有の値
5. **configuration.xml**: FSPコンフィギュレータのGUIで編集するファイルであり、直接テキスト編集は非推奨
