# Phase 9: 割込み解析レポート

## 概要

本レポートでは、EK-RA8P1 Quickstartプロジェクトで使用している割込みを詳細に解析し、割込み処理の全体像を把握する。

## 1. 割込みベクタテーブル

### 1.1 ベクタテーブル構造

本プロジェクトは**43個の周辺割込み**を使用している（`vector_data.h`より）。

RA8シリーズ（Cortex-M85）の割込みベクタ構造：

| ベクタ番号 | 種別 | ハンドラ | 説明 |
|------------|------|----------|------|
| 0 | Initial SP | - | スタックポインタ初期値 |
| 1 | Reset | Reset_Handler | リセットハンドラ |
| 2 | NMI | NMI_Handler | Non-Maskable Interrupt |
| 3 | HardFault | HardFault_Handler | ハードフォルト例外 |
| 4 | MemManage | MemManage_Handler | メモリ管理例外 |
| 5 | BusFault | BusFault_Handler | バスフォルト例外 |
| 6 | UsageFault | UsageFault_Handler | 使用フォルト例外 |
| 7-10 | Reserved | - | 予約 |
| 11 | SVCall | SVC_Handler | システムコール |
| 12 | Debug | DebugMon_Handler | デバッグモニタ |
| 13 | Reserved | - | 予約 |
| 14 | PendSV | PendSV_Handler | コンテキストスイッチ |
| 15 | SysTick | SysTick_Handler | システムティック |
| 16+ | IRQ0-42 | 周辺ISR | 周辺機能割込み |

### 1.2 周辺割込みベクタ一覧

`ra_gen/vector_data.c` で定義される43個の割込み：

| IRQ番号 | 割込みソース | ハンドラ関数 | 優先度 | 用途 |
|---------|-------------|--------------|--------|------|
| 0 | IIC1_RXI | `iic_master_rxi_isr` | 4 | I2C1 受信完了 |
| 1 | IIC1_TXI | `iic_master_txi_isr` | 12 | I2C1 送信バッファ空 |
| 2 | IIC1_TEI | `iic_master_tei_isr` | 12 | I2C1 送信完了 |
| 3 | IIC1_ERI | `iic_master_eri_isr` | 12 | I2C1 エラー |
| 4 | SCI8_RXI | `sci_b_uart_rxi_isr` | 4 | UART8 受信完了 |
| 5 | SCI8_TXI | `sci_b_uart_txi_isr` | - | UART8 送信バッファ空 |
| 6 | SCI8_TEI | `sci_b_uart_tei_isr` | - | UART8 送信完了 |
| 7 | SCI8_ERI | `sci_b_uart_eri_isr` | - | UART8 エラー |
| 8 | ADC_LIMCLPI | `adc_b_limclpi_isr` | 12 | ADC リミッタクリップ |
| 9 | ADC_ERR0 | `adc_b_err0_isr` | - | ADC Unit0 エラー |
| 10 | ADC_ERR1 | `adc_b_err1_isr` | - | ADC Unit1 エラー |
| 11 | ADC_RESOVF0 | `adc_b_resovf0_isr` | - | ADC Unit0 オーバーフロー |
| 12 | ADC_RESOVF1 | `adc_b_resovf1_isr` | - | ADC Unit1 オーバーフロー |
| 13 | ADC_CALEND0 | `adc_b_calend0_isr` | - | ADC Unit0 キャリブレーション完了 |
| 14 | ADC_CALEND1 | `adc_b_calend1_isr` | - | ADC Unit1 キャリブレーション完了 |
| 15 | ADC_ADI0 | `adc_b_adi0_isr` | - | ADC Gr.0 スキャン完了 |
| 16 | ADC_ADI1 | `adc_b_adi1_isr` | - | ADC Gr.1 スキャン完了 |
| 17 | ADC_ADI2 | `adc_b_adi2_isr` | - | ADC Gr.2 スキャン完了 |
| 18 | ADC_ADI3 | `adc_b_adi3_isr` | - | ADC Gr.3 スキャン完了 |
| 19 | ADC_ADI4 | `adc_b_adi4_isr` | - | ADC Gr.4 スキャン完了 |
| 20 | ADC_FIFOOVF | `adc_b_fifoovf_isr` | 12 | ADC FIFO オーバーフロー |
| 21-25 | ADC_FIFOREQ0-4 | `adc_b_fiforeqN_isr` | - | ADC FIFO読出要求 |
| 26 | ICU_IRQ13 | `r_icu_isr` | 12 | 外部ピン割込み13 (SW1) |
| 27 | ICU_IRQ12 | `r_icu_isr` | 12 | 外部ピン割込み12 (SW2) |
| 28 | GPT1_OVF | `gpt_counter_overflow_isr` | 8 | GPT1 オーバーフロー |
| 29 | ICU_IRQ19 | `r_icu_isr` | 2 | 外部ピン割込み19 (タッチパネル) |
| 30 | GPT5_OVF | `gpt_counter_overflow_isr` | 8 | GPT5 オーバーフロー (LED Pulse) |
| 31 | GPT3_OVF | `gpt_counter_overflow_isr` | 10 | GPT3 オーバーフロー (LED Brightness) |
| 32 | GLCDC_LINE | `glcdc_line_detect_isr` | 12 | GLCDC ライン検出 (VSYNC) |
| 33 | GLCDC_UNF1 | `glcdc_underflow_1_isr` | 12 | GLCDC Layer1 アンダーフロー |
| 34 | GLCDC_UNF2 | `glcdc_underflow_2_isr` | 12 | GLCDC Layer2 アンダーフロー |
| 35 | DRW_INT | `drw_int_isr` | - | DRW 2D描画完了 |
| 36 | VIN_IRQ | `vin_status_isr` | 12 | VIN フレーム完了 |
| 37 | VIN_ERR | `vin_error_isr` | 12 | VIN 同期エラー |
| 38 | MIPICSI_RX | `mipi_csi_rx_isr` | 12 | MIPI CSI 受信 |
| 39 | MIPICSI_DL | `mipi_csi_dl_isr` | 12 | MIPI CSI データレーン |
| 40 | MIPICSI_VC | `mipi_csi_vc_isr` | 12 | MIPI CSI 仮想チャネル |
| 41 | MIPICSI_PM | `mipi_csi_pm_isr` | 12 | MIPI CSI パワーマネジメント |
| 42 | MIPICSI_GST | `mipi_csi_gst_isr` | 12 | MIPI CSI ショートパケット |

---

## 2. 割込み優先度設計

### 2.1 優先度レベル一覧

RA8シリーズ（Cortex-M85）は8ビット優先度フィールドをサポートするが、FSPでは0-15の範囲を使用。**数値が小さいほど高優先度**。

| 優先度 | 用途 | 該当割込み |
|--------|------|-----------|
| 2 | 最高優先度 | IRQ19 (タッチパネル) |
| 4 | 高優先度 | IIC1_RXI, SCI8_RXI |
| 8 | 中優先度 | GPT1/GPT5 オーバーフロー |
| 10 | 中低優先度 | GPT3 オーバーフロー |
| 12 | 低優先度 | GLCDC, VIN, MIPI-CSI, ADC, IIC1_TXI/TEI/ERI, ICU_IRQ12/13 |

### 2.2 優先度階層図

```
高優先度 (Low Number)
    │
    ├── 優先度 2: IRQ19 (タッチパネル割込み)
    │              └─ リアルタイム性重視のユーザー入力
    │
    ├── 優先度 4: I2C/UART受信
    │              └─ 通信データ取りこぼし防止
    │
    ├── 優先度 8: GPT1/GPT5 (LED制御)
    │              └─ LED点滅/PWM制御
    │
    ├── 優先度 10: GPT3 (LED輝度)
    │              └─ PWM輝度制御
    │
    └── 優先度 12: GLCDC/VIN/MIPI-CSI/ADC/スイッチ
                   └─ フレーム処理、非クリティカル処理
低優先度 (High Number)
```

### 2.3 設計意図

1. **タッチパネル (優先度2)**: ユーザー入力の即座の応答が必要
2. **通信受信 (優先度4)**: データバッファオーバーランを防止
3. **タイマー (優先度8-10)**: 周期的処理だが遅延許容
4. **グラフィックス/カメラ (優先度12)**: フレームレートは60fps程度で十分

---

## 3. 主要割込みの処理内容

### 3.1 LCDフレーム完了割込み（GLCDC VSYNC）

```
GLCDC_LINE_DETECT割込み発生 (毎フレーム、約60Hz)
    │
    ▼
glcdc_line_detect_isr() [FSP HAL]
    │
    ├─→ ハードウェアフラグクリア
    │
    └─→ ユーザーコールバック呼び出し
            │
            ▼
        glcdc_vsync_isr() [display_thread_entry.c:744]
            │
            ├─→ イベント判定 (UNDERFLOW検出)
            │
            ├─→ g_update_fps_text = 1 (FPS更新フラグ)
            │
            └─→ g_vsync_flag = 1 (VSYNC完了フラグ)
```

**コード抜粋** (`display_thread_entry.c:744-766`):
```c
void glcdc_vsync_isr(display_callback_args_t *p_args)
{
    FSP_PARAMETER_NOT_USED(p_args);
    g_update_fps_text = 1;

    switch (p_args->event)
    {
        case DISPLAY_EVENT_GR1_UNDERFLOW:
            __NOP();
            break;
        case DISPLAY_EVENT_GR2_UNDERFLOW:
            __NOP();
            break;
        default:
    }
    g_vsync_flag = 1;
    return;
}
```

### 3.2 タッチパネル割込み（IRQ19）

```
タッチ検出 (FT5316がINT信号をアサート)
    │
    ▼
ICU_IRQ19割込み発生
    │
    ▼
r_icu_isr() [FSP HAL]
    │
    ├─→ ハードウェアフラグクリア
    │
    └─→ ユーザーコールバック呼び出し
            │
            ▼
        irq19_tp_callback() [touch_FT5316.c:134]
            │
            ├─→ s_touch_panel_int = true (タッチフラグ設定)
            │
            └─→ xSemaphoreGiveFromISR() でタスク通知
                    │
                    ▼
                tp_thread_entry() がウェイクアップ
                    │
                    └─→ I2C経由でタッチ座標読み取り
```

**コード抜粋** (`touch_FT5316.c:134-152`):
```c
void irq19_tp_callback(external_irq_callback_args_t *p_args)
{
    UNUSED_PARAM(p_args);
    s_touch_panel_int = true;

    BaseType_t xHigherPriorityTaskWoken;
    BaseType_t xResult;

    xHigherPriorityTaskWoken = pdFALSE;
    xResult = xSemaphoreGiveFromISR(g_irq_binary_semaphore, &xHigherPriorityTaskWoken);

    if (pdFAIL != xResult)
    {
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
}
```

### 3.3 カメラフレーム完了割込み（VIN）

```
VINフレームキャプチャ完了
    │
    ▼
VIN_IRQ割込み発生
    │
    ▼
vin_status_isr() [FSP HAL]
    │
    ├─→ ステータスレジスタ読み取り
    │
    └─→ ユーザーコールバック呼び出し
            │
            ▼
        vin0_callback() [camera_thread_entry.c:907]
            │
            ├─→ VIN_EVENT_NOTIFY イベント判定
            │
            ├─→ frame_complete確認
            │
            └─→ display_next_buffer_set() でディスプレイに通知
                    │
                    ▼
                ディスプレイスレッドがカメラバッファを表示
```

**コード抜粋** (`camera_thread_entry.c:907-938`):
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
                display_next_buffer_set(p_args->p_buffer);
            }
            break;
        }
        case VIN_EVENT_ERROR:
        {
            break;
        }
        default:
            break;
    }
}
```

### 3.4 I2C完了割込み

```
I2C転送完了/エラー発生
    │
    ▼
IIC1_RXI/TXI/TEI/ERI割込み発生
    │
    ▼
iic_master_*_isr() [FSP HAL]
    │
    ├─→ 転送状態更新
    │
    └─→ ユーザーコールバック呼び出し
            │
            ▼
        board_i2c_master_callback() [board_i2c_master.c:219]
            │
            ├─→ TX_COMPLETE/RX_COMPLETE判定
            │       └─→ xEventGroupSetBitsFromISR(I2C_TRANSFER_COMPLETE)
            │
            ├─→ ABORTED判定
            │       └─→ xEventGroupSetBitsFromISR(I2C_TRANSFER_ABORT)
            │
            └─→ portYIELD_FROM_ISR() でタスク切替
                    │
                    ▼
                待機中タスク (board_i2c_master_wait_complete) がウェイクアップ
```

**コード抜粋** (`board_i2c_master.c:219-252`):
```c
void board_i2c_master_callback(i2c_master_callback_args_t * p_args)
{
    BaseType_t xHigherPriorityTaskWoken;
    BaseType_t xResult = pdFAIL;

    xHigherPriorityTaskWoken = pdFALSE;

    if ((I2C_MASTER_EVENT_TX_COMPLETE == p_args->event) ||
        (I2C_MASTER_EVENT_RX_COMPLETE == p_args->event))
    {
        xResult = xEventGroupSetBitsFromISR(g_i2c_event_group,
                                            I2C_TRANSFER_COMPLETE,
                                            &xHigherPriorityTaskWoken);
    }
    else if (I2C_MASTER_EVENT_ABORTED == p_args->event)
    {
        xResult = xEventGroupSetBitsFromISR(g_i2c_event_group,
                                            I2C_TRANSFER_ABORT,
                                            &xHigherPriorityTaskWoken);
    }

    if (pdFAIL != xResult)
    {
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }

    R_BSP_IrqStatusClear(R_FSP_CurrentIrqGet());
}
```

### 3.5 GPTタイマー割込み（LED制御）

```
GPT1カウンタオーバーフロー (LED点滅周期)
    │
    ▼
GPT1_COUNTER_OVERFLOW割込み発生
    │
    ▼
gpt_counter_overflow_isr() [FSP HAL]
    │
    └─→ ユーザーコールバック呼び出し
            │
            ▼
        gpt_blinker_blue_callback() [common_init.c:190]
            │
            ├─→ blink_on フラグトグル
            │
            └─→ R_GPT_Start() / R_GPT_Stop() でPWM制御
```

**コード抜粋** (`common_init.c:190-205`):
```c
void gpt_blinker_blue_callback(timer_callback_args_t *p_args)
{
    UNUSED_PARAM(p_args);
    static bool_t blink_on = true;

    if (blink_on)
    {
        R_GPT_Start(s_pwm_pins[0].p_timer->p_ctrl);
    }
    else
    {
        R_GPT_Stop(s_pwm_pins[0].p_timer->p_ctrl);
    }

    blink_on = !blink_on;
}
```

---

## 4. タイマー割込み

### 4.1 システムティック（FreeRTOS）

| タイマー | 用途 | 周期 | ハンドラ |
|----------|------|------|----------|
| SysTick | FreeRTOSティック | 1ms | `SysTick_Handler` |

FreeRTOSのティックはCortex-M85のSysTickタイマーを使用。

### 4.2 GPTタイマー

| タイマー | インスタンス名 | 用途 | 周期 | ハンドラ |
|----------|----------------|------|------|----------|
| GPT1 | `g_blinker_blue` | LED点滅制御 | 可変 | `gpt_blinker_blue_callback` |
| GPT3 | `led_brightness_timer` | LED輝度PWM | - | - |
| GPT5 | `led_pulse_timer` | LEDパルス制御 | - | - |

---

## 5. RTOS用割込み

FreeRTOS（Cortex-M85）が使用するシステム割込み：

| 割込み | ベクタ番号 | 用途 |
|--------|-----------|------|
| SysTick | 15 | システムティック (1ms周期) |
| PendSV | 14 | コンテキストスイッチ (遅延) |
| SVCall | 11 | システムコール (vPortSVCHandler) |

### 5.1 FreeRTOS割込み優先度設定

FSP生成の`FreeRTOSConfig.h`での設定：

```c
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY      (15)
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY (1)
```

- **最低優先度**: 15 (SysTick, PendSV用)
- **SYSCALL最大優先度**: 1 (この値より高い優先度の割込みからFreeRTOS APIを呼べない)

---

## 6. 割込みからタスクへの通知メカニズム

### 6.1 セマフォ方式（タッチパネル）

```
ISRコンテキスト                      タスクコンテキスト
      │                                    │
irq19_tp_callback()                  tp_thread_entry()
      │                                    │
      ├─→ xSemaphoreGiveFromISR() ───────▶ xSemaphoreTake() でブロック解除
      │                                    │
      └─→ portYIELD_FROM_ISR()            └─→ タッチ座標読み取り
```

### 6.2 イベントグループ方式（I2C）

```
ISRコンテキスト                              タスクコンテキスト
      │                                            │
board_i2c_master_callback()              board_i2c_master_wait_complete()
      │                                            │
      ├─→ xEventGroupSetBitsFromISR() ───────────▶ xEventGroupWaitBits() でブロック解除
      │    (I2C_TRANSFER_COMPLETE or ABORT)        │
      │                                            └─→ 転送結果判定
      └─→ portYIELD_FROM_ISR()
```

### 6.3 フラグ方式（GLCDC VSYNC）

```
ISRコンテキスト                      タスクコンテキスト
      │                                    │
glcdc_vsync_isr()                   display_thread_entry()
      │                                    │
      └─→ g_vsync_flag = 1 ───────────────▶ while(!g_vsync_flag) でポーリング
                                           │
                                           └─→ 次フレーム描画
```

---

## 7. コールバック呼び出し経路

### 7.1 FSP HALコールバック機構

```
┌─────────────────────────────────────────────────────────────────────┐
│                       割込み発生                                      │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   FSP HAL ISR (例: iic_master_rxi_isr)              │
│  - ハードウェアレジスタ処理                                           │
│  - 内部状態更新                                                      │
│  - p_cfg->p_callback() 呼び出し                                      │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              ユーザーコールバック (例: board_i2c_master_callback)     │
│  - イベント判定                                                      │
│  - FreeRTOS API呼び出し (FromISR版)                                 │
│  - 必要に応じてコンテキストスイッチ要求                               │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   待機タスクのウェイクアップ                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 コールバック登録方法

コールバック関数はFSP構成構造体で登録される：

**I2Cコールバック登録** (`ra_gen/hal_data.c`):
```c
const i2c_master_cfg_t g_board_i2c_master_cfg =
{
    // ...
    .p_callback = board_i2c_master_callback,
    // ...
};
```

**GLCDCコールバック登録** (`ra_gen/common_data.c`):
```c
const display_cfg_t g_plcd_display_cfg =
{
    // ...
    .p_callback = glcdc_vsync_isr,
    // ...
};
```

**外部割込みコールバック登録** (`ra_gen/common_data.c`):
```c
const external_irq_cfg_t g_external_irq19_cfg =
{
    .channel = 19,
    .trigger = EXTERNAL_IRQ_TRIG_RISING,
    .p_callback = irq19_tp_callback,
    .ipl = (2),
    // ...
};
```

---

## 8. 割込み設定コード例

### 8.1 外部割込み（タッチパネル）の初期化

```c
// common_init.c での初期化
fsp_err_t icu_initialize(void)
{
    fsp_err_t fsp_err = FSP_SUCCESS;

    // 外部割込みオープン
    fsp_err = R_ICU_ExternalIrqOpen(&g_external_irq19_ctrl,
                                     &g_external_irq19_cfg);
    if (FSP_SUCCESS != fsp_err)
    {
        return fsp_err;
    }

    // 割込み有効化
    fsp_err = R_ICU_ExternalIrqEnable(&g_external_irq19_ctrl);

    return fsp_err;
}
```

### 8.2 GPTタイマーの初期化

```c
// common_init.c での初期化
static fsp_err_t gpt_initialize(void)
{
    fsp_err_t fsp_err = FSP_SUCCESS;

    // タイマーオープン
    fsp_err = R_GPT_Open(g_blinker_blue.p_ctrl, g_blinker_blue.p_cfg);
    if (FSP_SUCCESS != fsp_err)
    {
        return fsp_err;
    }

    // 周期設定
    R_GPT_PeriodSet(g_blinker_blue.p_ctrl, g_pwm_rates[g_board_status.led_frequency]);

    // タイマー開始
    R_GPT_Start(g_blinker_blue.p_ctrl);

    return fsp_err;
}
```

### 8.3 GLCDCの初期化

```c
// display_thread_entry.c での初期化
void display_thread_entry(void *pvParameters)
{
    // ディスプレイオープン（コールバック登録含む）
    R_GLCDC_Open(&g_plcd_display_ctrl, &g_plcd_display_cfg);

    // ディスプレイ開始（割込み有効化含む）
    R_GLCDC_Start(&g_plcd_display_ctrl);

    // ...
}
```

---

## 9. 割込みカテゴリ別サマリー

### 9.1 通信系割込み（8個）

| ペリフェラル | 割込み数 | 用途 |
|-------------|---------|------|
| IIC1 | 4 | タッチパネル/カメラI2C通信 |
| SCI8 | 4 | UARTデバッグコンソール |

### 9.2 タイマー系割込み（3個）

| ペリフェラル | 割込み数 | 用途 |
|-------------|---------|------|
| GPT1/3/5 | 3 | LED点滅/PWM制御 |

### 9.3 グラフィックス系割込み（4個）

| ペリフェラル | 割込み数 | 用途 |
|-------------|---------|------|
| GLCDC | 3 | VSYNC/アンダーフロー検出 |
| DRW | 1 | 2D描画完了 |

### 9.4 カメラ系割込み（7個）

| ペリフェラル | 割込み数 | 用途 |
|-------------|---------|------|
| VIN | 2 | フレーム完了/エラー |
| MIPI-CSI | 5 | データ受信/エラー検出 |

### 9.5 ADC系割込み（18個）

| ペリフェラル | 割込み数 | 用途 |
|-------------|---------|------|
| ADC_B | 18 | 温度センサ読み取り |

### 9.6 外部割込み（3個）

| ペリフェラル | 割込み数 | 用途 |
|-------------|---------|------|
| ICU_IRQ | 3 | スイッチ2個 + タッチパネル |

---

## 10. まとめ

### 10.1 割込み設計の特徴

1. **階層的優先度設計**: ユーザー入力（タッチ）が最高優先度
2. **FreeRTOS統合**: FromISR版API使用による適切なタスク通知
3. **FSPコールバック機構**: 統一的な割込みハンドリング
4. **リソース効率**: 必要な割込みのみ有効化

### 10.2 割込み数統計

| カテゴリ | 割込み数 |
|----------|---------|
| 通信系 | 8 |
| タイマー系 | 3 |
| グラフィックス系 | 4 |
| カメラ系 | 7 |
| ADC系 | 18 |
| 外部割込み | 3 |
| **合計** | **43** |

### 10.3 重要な設計判断

- **タッチパネル優先度2**: 即座のユーザー応答
- **I2C受信優先度4**: データ取りこぼし防止
- **GLCDC/VIN優先度12**: フレームレート維持で十分（60fps）
- **セマフォ/イベントグループ使用**: 効率的なタスク同期
