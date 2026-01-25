# Phase 8: レイヤー分類解析レポート

## 概要

本レポートでは、EK-RA8P1 Quickstartプロジェクトのソフトウェアを機能レイヤーに分類し、各レイヤーの責務と依存関係を明確にする。

## レイヤー間依存関係図

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        アプリケーション層                                      │
│   main_menu_thread_entry.c, display_thread_entry.c, camera_thread_entry.c   │
│   tp_thread_entry.c, board_mon_thread_entry.c, blinky_thread_entry.c        │
│   menu_main.c, menu_led.c, menu_lcd.c, menu_kis.c, menu_ext.c, menu_ns.c   │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ 呼び出し
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ミドルウェア層                                        │
│   Dave/2D Graphics Library (dave_*.c)                                       │
│   User Font Libraries (user_font_title_if.c, user_font_body_if.c)          │
│   Image Resources (images/*.c)                                              │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ 呼び出し
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                  OS/アプリケーションインターフェース層                         │
│   FreeRTOS API (vTaskDelay, xSemaphore*, xEventGroup*, etc.)               │
│   common_init.c, common_utils.c                                             │
│   Generated Thread Wrappers (ra_gen/*.c)                                    │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ 呼び出し
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                 ボード/デバイスドライバ層                                     │
│   board_i2c_master.c, board_sdram.c, board_greenpak.c                      │
│   board_cfg_switch.c, board_hw_cfg.c                                        │
│   touch_FT5316.c, ov5640.c                                                  │
│   ospi_commands.c, ospi_b_commands.c, ospi_b_ep.c                          │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ 呼び出し
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         HAL層                                                │
│   FSP HAL Drivers: r_glcdc.c, r_iic_master.c, r_gpt.c, r_icu.c             │
│   r_mipi_csi.c, r_mipi_phy.c, r_vin.c, r_ospi_b.c, r_adc_b.c              │
│   r_sci_b_uart.c, r_ioport.c, r_drw.c                                      │
│   BSP: bsp/*.c                                                              │
└─────────────────────────────┬───────────────────────────────────────────────┘
                              │ レジスタアクセス
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ハードウェア                                          │
│   GLCDC, IIC1, GPT, ICU, MIPI-CSI, VIN, OSPI, ADC, SCI8, IOPORT, DRW       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. アプリケーション層

### 1.1 責務
- メインアプリケーションロジック
- 画面制御、ユーザー操作処理
- ビジネスロジック
- FreeRTOSタスクのエントリーポイント

### 1.2 ファイル分類表

| ファイル | 役割 | 主要機能 |
|----------|------|----------|
| `src/main_menu_thread_entry.c` | メインメニュースレッド | コンソール初期化、ウェルカム画面表示、メインメニュー制御ループ |
| `src/display_thread_entry.c` | ディスプレイスレッド | GLCDC/Dave2D初期化、画面レンダリング、背景/オーバーレイ描画 |
| `src/camera_thread_entry.c` | カメラスレッド | MIPI-CSI初期化、OV5640カメラ制御、VIN設定、画像キャプチャ |
| `src/tp_thread_entry.c` | タッチパネルスレッド | タッチ入力処理、メニュー検出、ヒットボックス判定 |
| `src/board_mon_thread_entry.c` | ボード監視スレッド | システムステータス監視、ADC読み取り |
| `src/blinky_thread_entry.c` | LED点滅スレッド | FreeRTOSフレームワーク必須（現在は未使用） |
| `src/menu_main.c` | メインメニュー制御 | メニュー選択、コンソール表示、デモ切り替え |
| `src/menu_led.c` | LEDデモ画面 | LED制御インターフェース、PWMデューティ表示 |
| `src/menu_lcd.c` | LCDデモ画面 | カメラ表示デモ |
| `src/menu_kis.c` | キット情報画面 | EK-RA8P1情報表示 |
| `src/menu_ext.c` | 外部メモリ画面 | OSPIメモリ読み書きデモ |
| `src/menu_ns.c` | 次のステップ画面 | 開発ガイダンス表示 |
| `src/hal_entry.c` | HALエントリー | R_BSP_WarmStart、IOPORT/SDRAM初期化 |

### 1.3 主要モジュール説明

#### main_menu_thread_entry.c
- **責務**: システム起動時のメインスレッド、コンソール初期化、メニュー制御
- **公開API**:
  - `main_menu_thread_entry(void *pvParameters)` - FreeRTOSタスクエントリー
- **他モジュールとの連携**:
  - `jlink_console.c` - シリアルコンソール出力
  - `common_init.c` - ハードウェア初期化
  - `menu_main.c` - メインメニュー表示

#### display_thread_entry.c
- **責務**: 画面描画、GLCDC/Dave2D制御、フレームバッファ管理
- **公開API**:
  - `display_thread_entry(void *pvParameters)` - FreeRTOSタスクエントリー
  - `dsp_set_background(uint32_t choice)` - 背景画面設定
  - `graphics_wait_vsync(void)` - VSYNC待機
  - `graphics_get_draw_buffer(void)` - 描画バッファ取得
  - `display_next_buffer_set(uint8_t* next_buffer)` - カメラバッファ設定
- **他モジュールとの連携**:
  - `menu_*.c` - 各画面の描画更新
  - `tp_thread_entry.c` - オーバーレイ状態

---

## 2. ミドルウェア層

### 2.1 責務
- 上位レイヤーへの機能提供
- 複雑な処理の抽象化
- グラフィックスライブラリ

### 2.2 ファイル分類表

| ディレクトリ/ファイル | 役割 | 説明 |
|----------------------|------|------|
| `ra/tes/dave2d/src/dave_*.c` | Dave/2D グラフィックスライブラリ | 2Dレンダリングエンジン（TES製） |
| `src/images/user_font_title/user_font_title_if.c` | タイトルフォントインターフェース | 大文字フォントレンダリング |
| `src/images/user_font_body/user_font_body_if.c` | ボディフォントインターフェース | 本文フォントレンダリング |
| `src/images/overlays/menu_overlay.c` | メニューオーバーレイ画像 | ポップアップメニュー画像データ |
| `src/images/user_font_title/*.c` | タイトルフォントデータ | A-Z, 記号のビットマップデータ |
| `src/images/user_font_body/lc/*.c` | 小文字フォントデータ | a-z のビットマップデータ |
| `src/images/user_font_body/uc/*.c` | 大文字フォントデータ | A-Z のビットマップデータ |

### 2.3 Dave/2D グラフィックスライブラリ

**主要ファイル**:
| ファイル | 役割 |
|----------|------|
| `dave_context.c` | デバイスコンテキスト管理 |
| `dave_render.c` | レンダリングコア |
| `dave_blit.c` | ブリット（画像転送）処理 |
| `dave_line.c` | 直線描画 |
| `dave_box.c` | 矩形描画 |
| `dave_circle.c` | 円描画 |
| `dave_texture.c` | テクスチャ処理 |
| `dave_driver.c` | ハードウェアドライバインターフェース |
| `dave_rbuffer.c` | レンダーバッファ管理 |
| `dave_dlist.c` | ディスプレイリスト管理 |

**公開API（頻出）**:
- `d2_opendevice()` - デバイスオープン
- `d2_inithw()` - ハードウェア初期化
- `d2_framebuffer()` - フレームバッファ設定
- `d2_setblitsrc()` - ブリットソース設定
- `d2_blitcopy()` - 画像コピー
- `d2_renderbox()` - 矩形レンダリング
- `d2_renderline()` - 直線レンダリング
- `d2_setcolor()` - 描画色設定
- `d2_setalpha()` - アルファ値設定
- `d2_executerenderbuffer()` - レンダーバッファ実行
- `d2_flushframe()` - フレームフラッシュ

---

## 3. OS/アプリケーションインターフェース層

### 3.1 責務
- RTOSのAPI呼び出し
- タスク間通信の管理
- 共通初期化/ユーティリティ

### 3.2 ファイル分類表

| ファイル | 役割 | 説明 |
|----------|------|------|
| `src/common_init.c` | 共通初期化 | ADC/ICU/GPT初期化、GreenPak初期化、SDRAM/OSPI初期化 |
| `src/common_init.h` | 共通初期化ヘッダ | グローバル定義、LCD背景定数 |
| `src/common_utils.c` | 共通ユーティリティ | エラー処理、デバッグサポート |
| `src/common_utils.h` | 共通ユーティリティヘッダ | 型定義、マクロ、構造体 |
| `src/jlink_console.c` | J-Linkコンソール | SEGGER RTTデバッグ出力 |
| `ra_gen/main.c` | 生成されたメイン関数 | FreeRTOS初期化、タスク生成、スケジューラ起動 |
| `ra_gen/common_data.c` | 生成された共通データ | MIPI PHY/CSI/VIN設定構造体 |
| `ra_gen/hal_data.c` | 生成されたHALデータ | ペリフェラル設定構造体 |
| `ra_gen/*_thread.c` | 生成されたスレッドラッパー | FreeRTOSタスク生成関数 |

### 3.3 主要モジュール説明

#### common_init.c
- **責務**: システム起動時の共通初期化
- **公開API**:
  - `common_init(void)` - 全ペリフェラル初期化
  - `led_duty_cycle_update(void)` - LED PWM更新
- **初期化シーケンス**:
  1. GreenPak初期化
  2. ボード設定スイッチ初期化
  3. OSPI初期化（OPIモード設定）
  4. SDRAM初期化
  5. ADC初期化
  6. ICU（割込み）初期化
  7. GPT（タイマー）初期化

#### ra_gen/main.c（自動生成）
- **責務**: FreeRTOSスケジューラ起動
- **生成されるタスク**:
  1. `display_thread` - ディスプレイ処理
  2. `camera_thread` - カメラ処理
  3. `main_menu_thread` - メインメニュー
  4. `board_mon_thread` - ボード監視
  5. `tp_thread` - タッチパネル
  6. `blinky_thread` - LED点滅

### 3.4 FreeRTOS API使用パターン

| API | 使用箇所 | 用途 |
|-----|----------|------|
| `vTaskDelay()` | 全スレッド | タスク遅延 |
| `xSemaphoreGiveFromISR()` | `touch_FT5316.c` | ISRからタッチ通知 |
| `xSemaphoreTake()` | `main.c` | 初期化同期 |
| `xEventGroupWaitBits()` | `board_i2c_master.c` | I2C完了待機 |
| `xEventGroupSetBitsFromISR()` | I2Cコールバック | I2C完了通知 |

---

## 4. ボード/デバイスドライバ層

### 4.1 責務
- 外部デバイスの制御
- デバイス固有のプロトコル実装
- ボード固有のハードウェア設定

### 4.2 ファイル分類表

| ファイル | 役割 | 対象デバイス |
|----------|------|--------------|
| `src/touch_FT5316.c` | タッチコントローラドライバ | FT5316 (I2C) |
| `src/ov5640.c` | カメラセンサドライバ | OV5640 (I2C + MIPI-CSI) |
| `src/board_i2c_master.c` | I2Cバスマスタドライバ | I2Cバス共通 |
| `src/board_sdram.c` | SDRAMドライバ | 外部SDRAM (128MB) |
| `src/board_greenpak.c` | GreenPakドライバ | SLG46826 Mixed-Signal IC |
| `src/board_cfg_switch.c` | ボード設定スイッチ | 機能選択スイッチ |
| `src/board_hw_cfg.c` | ボードハードウェア設定 | ピン/クロック設定 |
| `src/ospi_commands.c` | OSPIコマンド | W25Q512 (SPI Flash) |
| `src/ospi_b_commands.c` | OSPI Bank Bコマンド | Bank B操作 |
| `src/ospi_b_ep.c` | OSPIエンドポイント | OSPI初期化/モード切替 |

### 4.3 主要モジュール説明

#### touch_FT5316.c
- **責務**: FT5316タッチコントローラとのI2C通信
- **公開API**:
  - `touch_panel_reset(void)` - タッチパネルリセット
  - `touch_panel_configure(void)` - タッチパネル設定
  - `read_ft5316_tp_status(void)` - タッチポイント数取得
  - `read_reg8()` / `write_reg8()` - レジスタ読み書き
- **コールバック**:
  - `irq19_tp_callback()` - タッチ割込みハンドラ

#### ov5640.c
- **責務**: OV5640カメラセンサの初期化と制御
- **公開API**:
  - `ov5640_hw_init(void)` - ハードウェア初期化（リセット/パワーダウン）
  - `ov5640_exit_power_down(void)` - パワーダウン解除
  - `ov5640_hw_reset(void)` - ハードウェアリセット
  - `ov5640_write_reg()` / `ov5640_read_reg()` - レジスタ読み書き

#### board_i2c_master.c
- **責務**: 全I2Cデバイスへの共通アクセス
- **公開API**:
  - `board_i2c_master_wait_complete(void)` - I2C転送完了待機
  - `wr_sensor_reg8_8()` / `rd_sensor_reg8_8()` - 8bitレジスタ操作
  - `wr_sensor_reg16_8()` / `rd_sensor_reg16_8()` - 16bitアドレスレジスタ操作
- **同期機構**: FreeRTOS EventGroup使用

---

## 5. HAL（Hardware Abstraction Layer）層

### 5.1 責務
- ハードウェアの抽象化
- 周辺機能への統一インターフェース
- レジスタ直接アクセスのカプセル化

### 5.2 ファイル分類表

| ディレクトリ | モジュール | 対象ペリフェラル |
|--------------|------------|------------------|
| `ra/fsp/src/r_glcdc/` | `r_glcdc.c` | Graphics LCD Controller |
| `ra/fsp/src/r_iic_master/` | `r_iic_master.c` | I2C Master |
| `ra/fsp/src/r_gpt/` | `r_gpt.c` | General PWM Timer |
| `ra/fsp/src/r_icu/` | `r_icu.c` | Interrupt Controller Unit |
| `ra/fsp/src/r_mipi_csi/` | `r_mipi_csi.c` | MIPI CSI-2 Receiver |
| `ra/fsp/src/r_mipi_phy/` | `r_mipi_phy.c` | MIPI D-PHY |
| `ra/fsp/src/r_vin/` | `r_vin.c` | Video Input |
| `ra/fsp/src/r_ospi_b/` | `r_ospi_b.c` | Octal SPI |
| `ra/fsp/src/r_adc_b/` | `r_adc_b.c` | A/D Converter (Type B) |
| `ra/fsp/src/r_sci_b_uart/` | `r_sci_b_uart.c` | Serial Communications (UART) |
| `ra/fsp/src/r_ioport/` | `r_ioport.c` | I/O Port |
| `ra/fsp/src/r_drw/` | `r_drw.c` | 2D Drawing Engine |
| `ra/fsp/src/bsp/` | BSPファイル群 | Board Support Package |
| `ra/fsp/src/rm_freertos_port/` | FreeRTOSポート | RTOS移植層 |

### 5.3 FSP HAL API パターン

全FSPモジュールは以下の統一APIパターンに従う：

```c
R_<MODULE>_Open()    // モジュールオープン
R_<MODULE>_Close()   // モジュールクローズ
R_<MODULE>_Start()   // 動作開始
R_<MODULE>_Stop()    // 動作停止
R_<MODULE>_Read()    // データ読み取り
R_<MODULE>_Write()   // データ書き込み
```

### 5.4 主要HALモジュール

#### R_GLCDC (Graphics LCD Controller)
- **公開API**:
  - `R_GLCDC_Open()` - GLCDCオープン
  - `R_GLCDC_Start()` - 表示開始
  - `R_GLCDC_BufferChange()` - フレームバッファ切替
- **使用箇所**: `display_thread_entry.c`

#### R_IIC_MASTER (I2C Master)
- **公開API**:
  - `R_IIC_MASTER_Open()` - I2Cオープン
  - `R_IIC_MASTER_SlaveAddressSet()` - スレーブアドレス設定
  - `R_IIC_MASTER_Write()` - データ送信
  - `R_IIC_MASTER_Read()` - データ受信
- **使用箇所**: `board_i2c_master.c`, `touch_FT5316.c`, `ov5640.c`

---

## 6. 割込みハンドラ層

### 6.1 責務
- ハードウェア割込みの処理
- 割込みコンテキストでの処理
- コールバック関数の実行

### 6.2 割込みベクタ一覧

`ra_gen/vector_data.c` で定義される割込みベクタ：

| ベクタ番号 | ISR関数 | 説明 |
|------------|---------|------|
| 0-3 | `iic_master_*_isr` | I2C1 (RXI/TXI/TEI/ERI) |
| 4-7 | `sci_b_uart_*_isr` | SCI8 UART (RXI/TXI/TEI/ERI) |
| 8-25 | `adc_b_*_isr` | ADC各種割込み |
| 26-27 | `r_icu_isr` | 外部ピン割込み (IRQ12/13) |
| 28, 30-31 | `gpt_counter_overflow_isr` | GPT オーバーフロー |
| 29 | `r_icu_isr` | 外部ピン割込み (IRQ19 - タッチ) |
| 32-34 | `glcdc_*_isr` | GLCDC (LINE_DETECT/UNDERFLOW) |
| 35 | `drw_int_isr` | DRW (2D描画エンジン) |
| 36-37 | `vin_*_isr` | VIN (IRQ/ERR) |
| 38-42 | `mipi_csi_*_isr` | MIPI CSI (RX/DL/VC/PM/GST) |

### 6.3 アプリケーション定義コールバック

| コールバック関数 | 定義ファイル | 用途 |
|-----------------|--------------|------|
| `glcdc_vsync_isr()` | `display_thread_entry.c` | VSYNC通知、FPS更新 |
| `irq19_tp_callback()` | `touch_FT5316.c` | タッチパネル割込み |
| `gpt_blinker_blue_callback()` | `common_init.c` | LED点滅タイマー |
| `mipi_csi0_callback()` | `camera_thread_entry.c` | MIPI CSIイベント |
| `g_board_i2c_master_callback()` | `board_i2c_master.c` | I2C完了通知 |

### 6.4 ISR処理パターン

```c
// タッチパネル割込みの例
void irq19_tp_callback(external_irq_callback_args_t *p_args)
{
    UNUSED_PARAM(p_args);

    s_touch_panel_int = true;

    // FreeRTOSセマフォでタスクに通知
    BaseType_t xHigherPriorityTaskWoken;
    xResult = xSemaphoreGiveFromISR(g_irq_binary_semaphore, &xHigherPriorityTaskWoken);

    if (pdFAIL != xResult)
    {
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
}
```

---

## 7. 設計パターンの識別

### 7.1 コールバックパターン

FSPフレームワーク全体で使用される主要パターン。HALドライバは割込み完了時にコールバック関数を呼び出す。

**使用例**:
- `glcdc_vsync_isr()` - GLCDC VSYNC割込みコールバック
- `g_board_i2c_master_callback()` - I2C完了コールバック
- `mipi_csi0_callback()` - MIPI CSI割込みコールバック

```c
// FSP構成構造体でコールバック登録
const mipi_csi_cfg_t g_mipi_csi0_cfg = {
    // ...
    .p_callback = mipi_csi0_callback,
    // ...
};
```

### 7.2 オブザーバーパターン（イベント通知）

FreeRTOSのセマフォ/イベントグループを使用したイベント通知。

**使用例**:
- タッチパネル割込み → セマフォ → タッチスレッド
- I2C完了 → イベントグループ → 待機タスク

```c
// ISRからのイベント通知
xSemaphoreGiveFromISR(g_irq_binary_semaphore, &xHigherPriorityTaskWoken);

// タスクでの待機
xEventGroupWaitBits(g_i2c_event_group, I2C_TRANSFER_COMPLETE, ...);
```

### 7.3 ステートマシンパターン

メニュー制御で使用される状態遷移パターン。

**使用例**:
- `g_current_backgroud.active_demo` - 現在のデモ画面状態
- `g_overlay_selected` - オーバーレイ表示状態
- `g_selected_menu` - 選択されたメニュー項目

```c
// display_thread_entry.c での状態に基づく描画切替
switch (g_current_backgroud.active_demo)
{
    case LCD_FULL_BG_GETTING_STARTED:
        // Getting Started画面描画
        break;
    case LCD_FULL_BG_USER_LED:
        menu_led_update_screen(gp_d2_handle);
        break;
    case LCD_FULL_BG_CAMERA_BACKGROUND:
        menu_lcd_update_screen(gp_d2_handle);
        break;
    // ...
}
```

### 7.4 ダブルバッファリングパターン

画面のちらつきを防ぐためのバッファ切替パターン。

```c
// フレームバッファ切替
s_drw_buf = s_drw_buf ? 0 : 1;

void * graphics_get_draw_buffer(void)
{
    return &(fb_foreground[s_drw_buf][0]);
}

void * graphics_get_back_buffer(void)
{
    uint8_t tmp = (s_drw_buf == 0) ? 1 : 0;
    return &(fb_foreground[tmp][0]);
}
```

### 7.5 レイヤー合成パターン

GLCDC Layer1（カメラ）とLayer2（UI）を重ね合わせる。

```c
// Layer 1: カメラ画像
R_GLCDC_BufferChange(&g_plcd_display_ctrl, gp_camera_buffer, DISPLAY_FRAME_LAYER_1);

// Layer 2: UIオーバーレイ
R_GLCDC_BufferChange(g_plcd_display.p_ctrl, graphics_get_back_buffer(), DISPLAY_FRAME_LAYER_2);
```

---

## 8. 総合ファイル分類表

| レイヤー | ファイル | 役割 |
|----------|----------|------|
| **アプリケーション** | `src/main_menu_thread_entry.c` | メインメニュースレッド |
| **アプリケーション** | `src/display_thread_entry.c` | ディスプレイスレッド |
| **アプリケーション** | `src/camera_thread_entry.c` | カメラスレッド |
| **アプリケーション** | `src/tp_thread_entry.c` | タッチパネルスレッド |
| **アプリケーション** | `src/board_mon_thread_entry.c` | ボード監視スレッド |
| **アプリケーション** | `src/blinky_thread_entry.c` | LED点滅スレッド |
| **アプリケーション** | `src/menu_main.c` | メインメニュー制御 |
| **アプリケーション** | `src/menu_led.c` | LEDデモ画面 |
| **アプリケーション** | `src/menu_lcd.c` | LCDデモ画面 |
| **アプリケーション** | `src/menu_kis.c` | キット情報画面 |
| **アプリケーション** | `src/menu_ext.c` | 外部メモリ画面 |
| **アプリケーション** | `src/menu_ns.c` | 次のステップ画面 |
| **アプリケーション** | `src/hal_entry.c` | HALエントリー |
| **ミドルウェア** | `ra/tes/dave2d/src/dave_*.c` | Dave/2D グラフィックス |
| **ミドルウェア** | `src/images/user_font_*/` | フォントライブラリ |
| **ミドルウェア** | `src/images/overlays/` | オーバーレイ画像 |
| **OS/App I/F** | `src/common_init.c` | 共通初期化 |
| **OS/App I/F** | `src/common_utils.c` | 共通ユーティリティ |
| **OS/App I/F** | `src/jlink_console.c` | J-Linkコンソール |
| **OS/App I/F** | `ra_gen/main.c` | メイン関数（生成） |
| **OS/App I/F** | `ra_gen/common_data.c` | 共通データ（生成） |
| **OS/App I/F** | `ra_gen/hal_data.c` | HALデータ（生成） |
| **OS/App I/F** | `ra_gen/*_thread.c` | スレッドラッパー（生成） |
| **デバイスドライバ** | `src/touch_FT5316.c` | タッチコントローラ |
| **デバイスドライバ** | `src/ov5640.c` | カメラセンサ |
| **デバイスドライバ** | `src/board_i2c_master.c` | I2Cバスマスタ |
| **デバイスドライバ** | `src/board_sdram.c` | SDRAM |
| **デバイスドライバ** | `src/board_greenpak.c` | GreenPak |
| **デバイスドライバ** | `src/board_cfg_switch.c` | 設定スイッチ |
| **デバイスドライバ** | `src/board_hw_cfg.c` | ハードウェア設定 |
| **デバイスドライバ** | `src/ospi_*.c` | OSPIフラッシュ |
| **HAL** | `ra/fsp/src/r_glcdc/r_glcdc.c` | GLCDC HAL |
| **HAL** | `ra/fsp/src/r_iic_master/` | I2C Master HAL |
| **HAL** | `ra/fsp/src/r_gpt/` | GPT HAL |
| **HAL** | `ra/fsp/src/r_icu/` | ICU HAL |
| **HAL** | `ra/fsp/src/r_mipi_csi/` | MIPI CSI HAL |
| **HAL** | `ra/fsp/src/r_mipi_phy/` | MIPI PHY HAL |
| **HAL** | `ra/fsp/src/r_vin/` | VIN HAL |
| **HAL** | `ra/fsp/src/r_ospi_b/` | OSPI HAL |
| **HAL** | `ra/fsp/src/r_adc_b/` | ADC HAL |
| **HAL** | `ra/fsp/src/r_sci_b_uart/` | SCI UART HAL |
| **HAL** | `ra/fsp/src/r_ioport/` | IOPORT HAL |
| **HAL** | `ra/fsp/src/r_drw/` | DRW HAL |
| **HAL** | `ra/fsp/src/bsp/` | BSP |
| **割込みハンドラ** | `ra_gen/vector_data.c` | 割込みベクタテーブル |

---

## 9. まとめ

本プロジェクトは、明確なレイヤー構造を持つ組込みグラフィックスシステムである。

### 主な特徴

1. **FSPフレームワーク準拠**: Renesas FSPの標準的なレイヤー構造に従っている
2. **コールバックベース設計**: HAL層からアプリケーション層への非同期通知
3. **FreeRTOSマルチタスク**: 6つの独立タスクによる並列処理
4. **Dave/2Dハードウェアアクセラレーション**: 2D描画のハードウェア支援
5. **ダブルバッファリング**: 画面更新のちらつき防止

### レイヤー間の依存関係

- アプリケーション層は全ての下位レイヤーに依存
- ミドルウェア層はHAL層に依存
- デバイスドライバ層はHAL層の統一APIを使用
- HAL層はBSPとハードウェアレジスタに直接アクセス

### ソフトウェア規模

| レイヤー | ソースファイル数 | 主要行数概算 |
|----------|------------------|--------------|
| アプリケーション | 13 | ~3,500行 |
| ミドルウェア | 50+ | ~15,000行 |
| OS/App I/F | 10 | ~2,000行 |
| デバイスドライバ | 9 | ~2,500行 |
| HAL | 15+ | ~20,000行 |
| 割込みハンドラ | 1 | ~100行 |
