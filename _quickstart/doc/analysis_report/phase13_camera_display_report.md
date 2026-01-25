# Phase 13: カメラ表示の仕組み 解析レポート

## 1. エグゼクティブサマリー

本解析では、EK-RA8P1 quickstartプロジェクトにおけるカメラ撮影画像のLCD表示の仕組みを詳細に調査した。OV5640 CMOSカメラセンサからMIPI-CSIインターフェース経由で画像を取得し、VIN（Video Input Module）で処理した後、GLCDCでLCDに表示する一連のデータフローを解明した。

## 2. カメラハードウェア

### 2.1 カメラセンサの仕様

| 項目 | 値 |
|------|-----|
| センサ型番 | OV5640 (OmniVision) |
| センサタイプ | 5MP CMOS イメージセンサ |
| 最大解像度 | 2592 x 1944 (5MP) |
| 使用解像度 | 1024 x 600 (LCD同サイズ) |
| 出力形式 | YUV422 8-bit |
| インターフェース | MIPI-CSI (2レーン) |
| I2Cアドレス | 0x3C (0x78 >> 1) |

**出典**: `src/ov5640_cfg.h:27`
```c
#define OV5640_I2C_SLAVE_ADDR               (0x78>>1)
```

### 2.2 カメラ制御ピン

| 機能 | ピン | 論理 |
|------|------|------|
| 電源制御 (PWDN) | P07_04 | Active HIGH |
| リセット (RST) | P07_09 | Active LOW |

**出典**: `src/ov5640_cfg.h:28-29`
```c
#define OV5640_CAM_PWR_ON                   (BSP_IO_PORT_07_PIN_04)
#define OV5640_CAM_RESET                    (BSP_IO_PORT_07_PIN_09)
```

### 2.3 MIPI-CSI設定

| 項目 | 設定値 |
|------|--------|
| レーン数 | 2 |
| HS Clock | 185 MHz |
| 外部クロック | 24 MHz (GPT出力) |
| 仮想チャネル | 0 |
| データタイプ | YUV422 8-bit |
| PHY PLL周波数 | 1000 MHz |

**出典**: `src/camera_thread_entry.c:67-68, 146`
```c
#define MIPI_NUM_LANES (2)
#define MIPI_XCLK_HZ (24000000) // 24 MHz
#define OV5640_HSCLK_MHZ    (185)
```

**出典**: `ra_gen/common_data.c:43-44`
```c
.pll_settings = /* Calculated MIPI PHY PLL frequency: 1000000000 Hz (error 0.00%) = (24000000 Hz / 3) * 125.00 / 1 */
{ .div = 3 - 1, .pll_div = 0, .mul_int = 125 - 1, .mul_frac = 0 /* Value: 0 */},
```

### 2.4 MIPI-CSIデータ受信設定

| 項目 | 設定値 |
|------|--------|
| データタイプ有効化 | YUV422 8-bit |
| フレームエラー通知 | 有効 |
| ECC 24ビットチェック | 有効 |
| 連続クロック | 有効 |

**出典**: `ra_gen/common_data.c:54-65`
```c
.ctrl_data.control_0_bits.lane_count = 2,
.ctrl_data.control_0_bits.err_frame_notify = 1,
.ctrl_data.control_0_bits.ecc_check_24_bits = 1,
.option_data.data_type_enable = (mipi_csi_rx_data_enable_t) (MIPI_CSI_RX_DATA_ENABLE_YUV422_8_BIT | 0x0),
```

## 3. VIN（Video Input Module）設定

### 3.1 入力制御設定

| 項目 | 設定値 |
|------|--------|
| 入力モード | YCbCr422 8-bit |
| 色空間変換バイパス | 無効（変換実行） |
| インターレースモード | 奇数/偶数フィールドキャプチャ |
| ディザリング | 加算モード |
| スケーリング | 無効 |

**出典**: `ra_gen/common_data.c:180-188`
```c
.input_ctrl.cfg_bits.color_space_convert_bypass = 0,
.input_ctrl.cfg_bits.interlace_mode = VIN_INTERLACE_MODE_ODD_EVEN_FIELD_CAPTURE,
.input_ctrl.cfg_bits.input_mode = VIN_INPUT_FORMAT_YCBCR422_8_BIT,
.input_ctrl.cfg_bits.dithering_mode = VIN_DITHERING_MODE_WITH_ADDITION,
.input_ctrl.cfg_bits.scaling_enable = false,
```

### 3.2 プリクリップ設定

| 項目 | 設定値 |
|------|--------|
| ライン開始 | 0 |
| ライン終了 | 449 |
| ピクセル開始 | 0 |
| ピクセル終了 | 767 |

**出典**: `ra_gen/common_data.c:191-194`

### 3.3 画像ストライド設定

| 項目 | 設定値 |
|------|--------|
| イメージストライド | 768 ピクセル |
| 1行あたりのバイト数 | 1536 バイト |
| フレームあたりのバイト数 | 691,200 バイト (1536 × 450) |

**出典**: `ra_gen/common_data.h:50-58`
```c
#define VIN_CFG_IMAGE_STRIDE (768)
#define VIN_CFG_BYTES_PER_LINE (1536)
#define VIN_BYTES_PER_FRAME (VIN_CFG_BYTES_PER_LINE * 450)
```

### 3.4 トリプルバッファ構成

| バッファ名 | サイズ | 配置 |
|-----------|--------|------|
| vin_image_buffer_1 | 691,200 bytes | SDRAM (noinit) |
| vin_image_buffer_2 | 691,200 bytes | SDRAM (noinit) |
| vin_image_buffer_3 | 691,200 bytes | SDRAM (noinit) |

**出典**: `ra_gen/common_data.c:171-173`
```c
uint8_t vin_image_buffer_1[VIN_BYTES_PER_FRAME] BSP_ALIGN_VARIABLE(128) BSP_PLACE_IN_SECTION(BSP_UNINIT_SECTION_PREFIX ".sdram_noinit");
uint8_t vin_image_buffer_2[VIN_BYTES_PER_FRAME] BSP_ALIGN_VARIABLE(128) BSP_PLACE_IN_SECTION(BSP_UNINIT_SECTION_PREFIX ".sdram_noinit");
uint8_t vin_image_buffer_3[VIN_BYTES_PER_FRAME] BSP_ALIGN_VARIABLE(128) BSP_PLACE_IN_SECTION(BSP_UNINIT_SECTION_PREFIX ".sdram_noinit");
```

## 4. 画像データフロー

### 4.1 データ取得フロー図

```
┌─────────────────────┐
│   OV5640 Camera     │
│   (5MP CMOS)        │
└──────────┬──────────┘
           │ YUV422 8-bit
           │ MIPI-CSI 2-lane
           │ 185MHz HS Clock
           ▼
┌─────────────────────┐
│  MIPI-CSI Interface │
│  (MIPI PHY + CSI)   │
│  - 2 Data Lanes     │
│  - PLL: 1GHz        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  VIN Module         │
│  - YCbCr422→RGB変換 │
│  - トリプルバッファ │
│  - フレーム完了割込 │
└──────────┬──────────┘
           │ DMA Write
           ▼
┌─────────────────────┐
│  SDRAM Buffers      │
│  - Buffer 1 (691KB) │
│  - Buffer 2 (691KB) │
│  - Buffer 3 (691KB) │
└──────────┬──────────┘
           │ vin0_callback()
           │ display_next_buffer_set()
           ▼
┌─────────────────────┐
│  GLCDC Layer 1      │
│  (Camera Image)     │
│  - RGB565 直接表示  │
└──────────┬──────────┘
           │ R_GLCDC_BufferChange()
           ▼
┌─────────────────────┐
│  LCD Panel          │
│  (1024 x 600)       │
└─────────────────────┘
```

### 4.2 画像フォーマット変換

| ステージ | フォーマット | サイズ |
|----------|-------------|--------|
| センサ出力 | YUV422 8-bit | 1024 x 600 |
| MIPI-CSI受信 | YUV422 8-bit | 1024 x 600 |
| VIN出力 | RGB565 | 768 x 450 (クリップ) |
| LCD表示 | RGB565 | 1024 x 600 |

### 4.3 カラー変換係数

VINではYCbCr→RGB変換に以下の係数を使用：

**出典**: `ra_gen/common_data.c:219-226`
```c
.yc_rgb_conversion_setting_1_bits.y_mul = 4767,
.yc_rgb_conversion_setting_2_bits.csub2 = 2048,
.yc_rgb_conversion_setting_2_bits.ysub2 = 256,
.yc_rgb_conversion_setting_3_bits.cgrmul2 = 3330,
.yc_rgb_conversion_setting_3_bits.rcrmul2 = 6537,
.yc_rgb_conversion_setting_4_bits.gcbmul2 = 1605,
.yc_rgb_conversion_setting_4_bits.bcbmul2 = 8261,
```

## 5. 割り込み処理

### 5.1 カメラ関連割り込みベクタ

| IRQ番号 | 割り込み名 | ISR | 優先度 |
|---------|-----------|-----|--------|
| 36 | VIN_IRQ | vin_status_isr | 12 |
| 37 | VIN_ERR | vin_error_isr | 12 |
| 38 | MIPICSI_RX | mipi_csi_rx_isr | 12 |
| 39 | MIPICSI_DL | mipi_csi_dl_isr | 12 |
| 40 | MIPICSI_VC | mipi_csi_vc_isr | 12 |
| 41 | MIPICSI_PM | mipi_csi_pm_isr | 12 |
| 42 | MIPICSI_GST | mipi_csi_gst_isr | 12 |

**出典**: `ra_gen/vector_data.h:125-138`

### 5.2 VINコールバック処理

**出典**: `src/camera_thread_entry.c:907-941`
```c
void vin0_callback (capture_callback_args_t * p_args)
{
    vin_event_t event = (vin_event_t) p_args->event;
    vin_interrupt_status_t interrupt_status = (vin_interrupt_status_t) p_args->interrupt_status;

    switch (event)
    {
        case VIN_EVENT_NOTIFY:
        {
            if (interrupt_status.bits.frame_complete)
            {
                /* Tell display thread camera image has updated */
                display_next_buffer_set(p_args->p_buffer);
            }
            break;
        }
        case VIN_EVENT_ERROR:
        {
            // Process Error based on interrupt and module status;
            break;
        }
    }
}
```

### 5.3 MIPI-CSIコールバック処理

**出典**: `src/camera_thread_entry.c:949-1000`
```c
void mipi_csi0_callback (mipi_csi_callback_args_t * p_args)
{
    switch (p_args->event)
    {
        case MIPI_CSI_EVENT_DATA_LANE:
        case MIPI_CSI_EVENT_FRAME_DATA:
        case MIPI_CSI_EVENT_POWER:
        case MIPI_CSI_EVENT_SHORT_PACKET_FIFO:
        case MIPI_CSI_EVENT_VIRTUAL_CHANNEL:
            // 各イベントのステータス処理
            break;
    }
}
```

## 6. カメラ初期化フロー

### 6.1 初期化シーケンス

```
camera_thread_entry()
    │
    ├─→ システム起動待ち (system_up())
    │
    ├─→ JLink設定待ち (jlink_configured())
    │
    ├─→ R_VIN_Open() - VINドライバ初期化
    │
    ├─→ システムクロック取得
    │      g_pclka_hz = R_FSP_SystemClockHzGet(FSP_PRIV_CLOCK_PCLKA)
    │
    ├─→ R_GPT_Open() - カメラクロック用タイマ初期化
    │
    ├─→ R_GPT_Start() - 24MHz外部クロック出力開始
    │
    ├─→ ov5640_hw_init() - 電源OFF、リセット
    │
    ├─→ ov5640_exit_power_down() - 電源ON、リセット解除
    │
    ├─→ ov5640_hw_reset() - ハードウェアリセット
    │
    ├─→ ov5640_sw_reset() - ソフトウェアリセット
    │
    ├─→ センサID確認 (0x300A, 0x300B)
    │      PIDL = 0x40, 0x41, 0x4C のいずれか
    │
    ├─→ OV5640レジスタ初期化 (200+レジスタ設定)
    │      - AWB, 色マトリクス, ガンマ
    │      - MIPI設定, タイミング設定
    │      - クロック設定
    │
    ├─→ ov5640_configure_clocks() - PLLクロック設定
    │
    ├─→ ov5640_set_mipi_virtual_channel(0) - VC0設定
    │
    ├─→ ov5640_stream_off() - ストリーム停止
    │
    ├─→ R_VIN_CaptureStart() - キャプチャ開始
    │
    ├─→ ov5640_stream_on() - ストリーム開始
    │
    └─→ s_camera_detected = true
```

**出典**: `src/camera_thread_entry.c:309-377`

### 6.2 電源シーケンス

**出典**: `src/ov5640.c:130-146`
```c
void ov5640_exit_power_down(void)
{
    /* Reset pin is Active LOW */
    OV5640_RST_PIN_SET(BSP_IO_LEVEL_LOW);
    delay_ms(20);

    /* Power Down pin is Active HIGH */
    OV5640_PWDN_PIN_SET(BSP_IO_LEVEL_LOW);  // Power ON
    delay_ms(50);

    /* Reset pin is Active LOW */
    OV5640_RST_PIN_SET(BSP_IO_LEVEL_HIGH);  // Reset解除
    delay_ms(20);
}
```

## 7. LCD表示統合

### 7.1 GLCDCレイヤー構成

| レイヤー | 用途 | バッファ | 更新方法 |
|---------|------|----------|----------|
| Layer 1 | カメラ画像 | VINバッファ直接参照 | vin0_callback() |
| Layer 2 | フォアグラウンド | Dave/2D描画バッファ | VSyncタイミング |

### 7.2 バッファ切り替え処理

**出典**: `src/display_thread_entry.c:157-163`
```c
void display_next_buffer_set(uint8_t* next_buffer)
{
    gp_camera_buffer = next_buffer;
}
```

**出典**: `src/display_thread_entry.c:264-269`
```c
if(LCD_FULL_BG_CAMERA_BACKGROUND == g_current_backgroud.active_demo)
{
    do{
        R_GLCDC_BufferChange(&g_plcd_display_ctrl, (uint8_t * const) gp_camera_buffer, DISPLAY_FRAME_LAYER_1);
    }while (err == FSP_ERR_INVALID_UPDATE_TIMING);
}
```

### 7.3 カメラ画面のGUI統合

カメラ画面 (LCD_FULL_BG_CAMERA_BACKGROUND = 4) では：
- Layer 1: カメラ映像（VINバッファ直接表示）
- Layer 2: タッチポイント表示、タイトルテキスト

**出典**: `src/display_thread_entry.c:324-341`
```c
case 4: // lcd demo (Camera)
    d2_clear(gp_d2_handle, 0x00000000);

#if R_CAMERA_LCD_TITLE_ENABLED_CFG
    /* Enable title on LCD for Camera demo */
    lv_point_t cursor_pos = {TITLE_LEFT_EDGE,18};
    user_font_title_draw_line(&cursor_pos, g_camera_welcome_message_str);
#endif
    menu_lcd_update_screen(gp_d2_handle);
    break;
```

## 8. 主要API/関数一覧

### 8.1 初期化・制御関数

| 関数名 | ファイル | 役割 |
|--------|----------|------|
| `camera_thread_entry()` | camera_thread_entry.c:309 | カメラスレッドメイン |
| `ov5640_init()` | camera_thread_entry.c:456 | センサ初期化 |
| `ov5640_hw_init()` | ov5640.c:110 | ハードウェア初期化 |
| `ov5640_exit_power_down()` | ov5640.c:130 | 電源ON処理 |
| `ov5640_hw_reset()` | ov5640.c:154 | ハードウェアリセット |
| `ov5640_sw_reset()` | ov5640.c:180 | ソフトウェアリセット |
| `ov5640_stream_on()` | camera_thread_entry.c:426 | ストリーミング開始 |
| `ov5640_stream_off()` | camera_thread_entry.c:440 | ストリーミング停止 |
| `ov5640_configure_clocks()` | camera_thread_entry.c:1056 | PLL/クロック設定 |

### 8.2 レジスタアクセス関数

| 関数名 | ファイル | 役割 |
|--------|----------|------|
| `ov5640_write_reg()` | ov5640.c:66 | レジスタ書き込み (16bitアドレス, 8bitデータ) |
| `ov5640_read_reg()` | ov5640.c:92 | レジスタ読み込み |

### 8.3 コールバック関数

| 関数名 | ファイル | 役割 |
|--------|----------|------|
| `vin0_callback()` | camera_thread_entry.c:907 | VINフレーム完了通知 |
| `mipi_csi0_callback()` | camera_thread_entry.c:949 | MIPI-CSIイベント処理 |

### 8.4 表示連携関数

| 関数名 | ファイル | 役割 |
|--------|----------|------|
| `display_next_buffer_set()` | display_thread_entry.c:157 | 表示バッファポインタ更新 |
| `check_ov5640_camera_connected()` | camera_thread_entry.c:387 | カメラ接続状態確認 |

## 9. クロック構成

### 9.1 OV5640内部クロック構成

```
External Clock (24MHz from GPT)
    │
    ├─→ PLL Pre-divider (÷8)
    │       = 3MHz
    │
    ├─→ PLL Multiplier (×246)
    │       = 738MHz
    │
    ├─→ PLL Root Divider (÷2)
    │       = 369MHz (Base PLL)
    │
    ├─→ System Clock Divider (÷1)
    │   └─→ SCLK Root Divider (÷4)
    │           = 92.25MHz (System Clock)
    │
    └─→ MIPI Clock Divider (÷2)
        └─→ PCLK Root Divider (÷2)
                = 92.25MHz (MIPI PCLK)
```

**出典**: `src/camera_thread_entry.c:253-267`
```c
static uint8_t s_sys_clock_div  = 1;    /* System clock divider */
static uint8_t s_mipi_clock_div = 2;    /* Scale divider for MIPI */
static uint8_t s_pll_multiplier = 246;  /* PLL multiplier */
static uint8_t s_pll_root_div   = 1u;   /* PLL root divider (1=÷2) */
static uint8_t pll_pre_div      = 8u;   /* PLL pre-divider */
static uint8_t pclk_root_div    = 2;    /* PCLK root divider */
static uint8_t sclk2x_root_div  = 1;    /* sclk2x root divider */
static uint8_t sclk_root_div    = 4;    /* SCLK root divider */
```

### 9.2 MIPI PHYタイミングパラメータ

| パラメータ | 値 | 単位 |
|-----------|-----|------|
| T_INIT | 74999 | cycles |
| T_CLK_PREP | 9 | cycles |
| T_CLK_SETTLE | 62 | cycles |
| T_CLK_MISS | 37 | cycles |
| T_HS_PREP | 6 | cycles |
| T_HS_SETT | 24 | cycles |
| T_CLK_TRAIL | 7 | cycles |
| T_CLK_POST | 20 | cycles |
| T_CLK_PRE | 1 | cycles |
| T_CLK_ZERO | 28 | cycles |

**出典**: `ra_gen/common_data.c:9-22`

## 10. フレームタイミング

### 10.1 タイミング図

```
時間軸 →

VIN Frame Capture:
    ┌─────────────────────┐     ┌─────────────────────┐
    │   Frame N Capture   │     │  Frame N+1 Capture  │
    └──────────┬──────────┘     └──────────┬──────────┘
               │                           │
    Frame Complete IRQ              Frame Complete IRQ
               │                           │
               ▼                           ▼
    vin0_callback()              vin0_callback()
    display_next_buffer_set()    display_next_buffer_set()
               │                           │
               ▼                           ▼

GLCDC Display:
    ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
    │ Display Frame │ │ Display Frame │ │ Display Frame │
    │      N-1      │ │      N        │ │     N+1       │
    └───────────────┘ └───────────────┘ └───────────────┘
            │               │               │
         VSync           VSync           VSync
            │               │               │
            ▼               ▼               ▼
    R_GLCDC_BufferChange()
    (Update Layer 1 pointer)
```

### 10.2 フレームレート

| 項目 | 値 |
|------|-----|
| カメラ出力 | ~30 fps (設定依存) |
| LCD表示 | 60 fps (GLCDC設定) |
| 同期方式 | VSync同期バッファ切り替え |

## 11. エラー処理

### 11.1 カメラ検出失敗時の処理

**出典**: `src/camera_thread_entry.c:484-498`
```c
reg_val = ov5640_read_reg(REG_PIDL);
if ((reg_val == 0x40) || (reg_val == 0x41) || (reg_val == 0x4C))
{
    /* Valid Camera */
}
else
{
    s_camera_detected = false;
    camera_state_determined = true;

    /* Halt the camera setup and block the thread */
    while(1)
    {
        vTaskDelay(100);
    }
}
```

### 11.2 VINエラー処理

VIN_EVENT_ERROR時には割り込みステータスとモジュールステータスに基づいてエラー処理を行う構造となっている（現在は未実装）。

## 12. システムブロック図

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EK-RA8P1 Camera System                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐                                                        │
│  │  Camera Module  │                                                        │
│  │    (OV5640)     │                                                        │
│  │                 │                                                        │
│  │  ┌───────────┐  │     I2C (0x3C)      ┌─────────────────────────┐       │
│  │  │  5MP CMOS │◄─┼────────────────────►│   IIC1 Master           │       │
│  │  │  Sensor   │  │                      │   (Config/Control)      │       │
│  │  └─────┬─────┘  │                      └─────────────────────────┘       │
│  │        │        │                                                        │
│  │  ┌─────▼─────┐  │     MIPI-CSI 2-lane  ┌─────────────────────────┐       │
│  │  │   MIPI    │──┼──────────────────────►│   MIPI PHY + CSI        │       │
│  │  │Transmitter│  │     185MHz HS Clock  │   (1GHz PLL)            │       │
│  │  └───────────┘  │                      └───────────┬─────────────┘       │
│  │                 │                                  │                     │
│  │  ◄──────────────┼─ 24MHz Clock (GPT) ─────────────►│                     │
│  │  PWR: P07_04    │                                  │                     │
│  │  RST: P07_09    │                                  │                     │
│  └─────────────────┘                                  │                     │
│                                                       │                     │
│                           ┌───────────────────────────▼──────────────────┐  │
│                           │           VIN (Video Input Module)            │  │
│                           │  ┌─────────────────────────────────────────┐  │  │
│                           │  │  Input: YCbCr422 8-bit                  │  │  │
│                           │  │  Pre-clip: 768 x 450                    │  │  │
│                           │  │  Output: RGB565                         │  │  │
│                           │  │  Triple Buffer: 3 x 691KB (SDRAM)       │  │  │
│                           │  └─────────────────────┬───────────────────┘  │  │
│                           │                        │ Frame Complete IRQ   │  │
│                           └────────────────────────┼──────────────────────┘  │
│                                                    │                         │
│                                                    ▼                         │
│                           ┌─────────────────────────────────────────────┐    │
│                           │                 SDRAM                        │    │
│                           │  ┌─────────────────────────────────────────┐ │   │
│                           │  │  vin_image_buffer_1  (691KB)            │ │   │
│                           │  │  vin_image_buffer_2  (691KB)            │ │   │
│                           │  │  vin_image_buffer_3  (691KB)            │ │   │
│                           │  │  fb_foreground[2]    (Dave/2D用)        │ │   │
│                           │  └─────────────────────────────────────────┘ │   │
│                           └───────────────────────┬─────────────────────┘    │
│                                                   │                          │
│                                                   ▼                          │
│                           ┌─────────────────────────────────────────────┐    │
│                           │              GLCDC (Graphics LCD)            │    │
│                           │  ┌──────────────────────────────────────┐   │    │
│                           │  │ Layer 1: Camera Image (VIN Buffer)   │   │    │
│                           │  │ Layer 2: Foreground (Dave/2D)        │   │    │
│                           │  └──────────────────────────────────────┘   │    │
│                           └───────────────────────┬─────────────────────┘    │
│                                                   │                          │
│                                                   ▼                          │
│                           ┌─────────────────────────────────────────────┐    │
│                           │           LCD Panel (1024 x 600)             │    │
│                           │             RGB565 @ 60Hz                    │    │
│                           └─────────────────────────────────────────────┘    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 13. 解析所見

### 13.1 アーキテクチャの特徴

1. **効率的なゼロコピー設計**: VINバッファをGLCDCが直接参照することで、メモリコピーを排除
2. **トリプルバッファリング**: ティアリングを防止しつつ、フレーム処理の柔軟性を確保
3. **ハードウェアカラー変換**: VINがYUV→RGB変換を実行し、CPU負荷を軽減
4. **レイヤー合成**: カメラ映像とGUI要素を別レイヤーで管理し、独立した更新が可能

### 13.2 設計上の考慮点

1. **カメラ検出**: 起動時にセンサIDを確認し、未接続時はスレッドをブロック
2. **同期処理**: VSync割り込みとフレーム完了割り込みを連携してバッファを切り替え
3. **クロック構成**: OV5640の複雑なPLL設定を事前計算し、定数として保持

### 13.3 制限事項

1. **解像度**: 現在は1024x600固定（LCD解像度に依存）
2. **フレームレート**: センサ設定とMIPIタイミングに依存
3. **メモリ使用量**: トリプルバッファで約2MB以上のSDRAM使用

## 14. 参考ファイル一覧

| ファイル | 役割 |
|----------|------|
| `src/camera_thread_entry.c` | カメラスレッド、OV5640初期化 |
| `src/ov5640.c` | OV5640低レベルドライバ |
| `src/ov5640_cfg.h` | OV5640ハードウェア設定 |
| `src/display_thread_entry.c` | 表示スレッド、バッファ管理 |
| `src/menu_lcd.c` | カメラデモUI |
| `ra_gen/common_data.c` | MIPI/VIN/GLCDC設定 |
| `ra_gen/vector_data.h` | 割り込みベクタ定義 |
