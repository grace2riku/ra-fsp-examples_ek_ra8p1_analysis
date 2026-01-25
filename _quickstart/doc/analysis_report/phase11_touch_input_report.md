# Phase 11: タッチ入力解析レポート

## 1. 概要

本レポートは、EK-RA8P1評価ボードのLCD画面タッチ入力システムを詳細に解析した結果をまとめたものである。タッチコントローラIC、ドライバ実装、座標変換、イベント処理フローの全体像を明らかにする。

## 2. タッチハードウェア

### 2.1 タッチコントローラIC

| 項目 | 値 |
|------|-----|
| IC型番 | FT5316 |
| メーカー | FocalTech |
| 接続方式 | I2C |
| I2Cアドレス | 0x38 (7ビット) |
| タッチ方式 | 静電容量式 |
| マルチタッチ対応 | 有（最大5点） |

**定義箇所**: `touch_FT5316.h:33`
```c
#define FT5316_SLAVE_ADDR   (0x38)    // 7-bit address
```

### 2.2 接続ピン

| 信号 | ピン番号 | 説明 |
|------|----------|------|
| SDA | P511 (SYS_I2C_SDA) | I2Cデータ（IIC1チャネル経由） |
| SCL | P512 (SYS_I2C_SCL) | I2Cクロック |
| INT | P111 (MIPI_INT) | 割込み出力（IRQ19） |
| RST | P606 (LCD_RESET) | リセット入力（LCD共用） |

**ピン定義箇所**: `bsp_pin_cfg.h:92-103`
```c
#define SYS_I2C_SDA (BSP_IO_PORT_05_PIN_11)
#define SYS_I2C_SCL (BSP_IO_PORT_05_PIN_12)
#define LCD_RESET (BSP_IO_PORT_06_PIN_06)
```

### 2.3 I2C設定

**設定箇所**: `hal_data.c:3829-3866`

| 設定項目 | 値 |
|----------|-----|
| I2Cチャネル | IIC1 |
| 通信速度 | Standard Mode（約97.8kHz） |
| アドレスモード | 7ビット |
| 割込み優先度 | 4 |

```c
const i2c_master_cfg_t g_board_i2c_master_cfg =
{ .channel = 1, .rate = I2C_MASTER_RATE_STANDARD, .slave = 0x00, .addr_mode = I2C_MASTER_ADDR_MODE_7BIT,
  ...
  .ipl = (4),
```

## 3. タッチドライバ

### 3.1 タッチデータ取得方法

| 方式 | 説明 |
|------|------|
| **割込み方式** | INTピン（IRQ19）からの立ち上がりエッジ割込みで通知 |
| **補完ポーリング** | 割込みレート補完のため、前回タッチ時にI2C読み出しも実施 |

**実際の方式**: 割込み + ポーリング併用（ハイブリッド）

**実装詳細**: `tp_thread_entry.c:453-484`
```c
static bool touchpad_is_pressed(void)
{
    BaseType_t status;
    bool touch_pressed = false;
    static bool was_pressed = false;

    status = xSemaphoreTake( g_irq_binary_semaphore, 0 );

    if (pdTRUE == status)
    {
        touch_pressed = true;
    }

    /* kludge to handle low interrupt rate of FT5316 until we find out how to improve the interrupt rate */
    if (was_pressed && !touch_pressed)
    {
        /* send I2C command to read touch points */
        if (read_ft5316_tp_status() > 0)
        {
            touch_pressed = true;
        }
    }

    was_pressed = touch_pressed;
    return (touch_pressed);
}
```

### 3.2 レジスタマップ

| レジスタ名 | アドレス | 説明 |
|-----------|----------|------|
| TD_STATUS | 0x02 | タッチポイント数 |
| TOUCHn_XH | 0x03 + 6*n | X座標上位（イベントフラグ含む） |
| TOUCHn_XL | 0x04 + 6*n | X座標下位 |
| TOUCHn_YH | 0x05 + 6*n | Y座標上位 |
| TOUCHn_YL | 0x06 + 6*n | Y座標下位 |
| ID_G_CIPHER | 0xA3 | チップベンダーID |
| ID_G_MODE | 0xA4 | 割込みステータス |
| ID_G_VENDOR_ID | 0xA8 | CTMPベンダーID |

**定義箇所**: `touch_FT5316.h:35-45`

### 3.3 座標データ形式

**タッチ座標データ構造体**: `touch_FT5316.h:51-61`
```c
typedef struct touch_coords
{
    uint16_t x[5];
    uint16_t y[5];
} st_touch_coords_t;

typedef struct all_touch_data
{
    uint8_t num_touches;
    st_touch_coords_t touch_data;
} st_touch_data_t;
```

**入力デバイスデータ構造体**: `common_utils.h:93-106`
```c
typedef struct {
    lv_point_t point[5];                /**< For LV_INDEV_TYPE_POINTER the currently pressed point*/
    uint32_t   number_of_coordinates;   /**<  */

    lv_indev_state_t state;             /**< LV_INDEV_STATE_RELEASED or LV_INDEV_STATE_PRESSED*/
    bool             continue_reading;  /**< If set to true, the read callback is invoked again,
                                             unless the device is in event-driven mode*/
} lv_indev_data_t;
```

### 3.4 座標変換処理

**座標読み取り処理**: `tp_thread_entry.c:496-549`

```c
static void touchpad_get_xy(touch_event_t * touch_event, lv_indev_data_t *data)
{
    ...
    for (uint8_t i = 0; i < tp_detected; i++)
    {
        read_reg8 ((uint8_t) (TOUCHn_XL + 6 * (i)), &reg_val_x_lsb, &g_board_i2c_master_ctrl, 1);
        read_reg8 ((uint8_t) (TOUCHn_YL + 6 * (i)), &reg_val_y_lsb, &g_board_i2c_master_ctrl, 1);
        read_reg8 ((uint8_t) (TOUCHn_XH + 6 * (i)), &reg_val_x_msb, &g_board_i2c_master_ctrl, 1);
        read_reg8 ((uint8_t) (TOUCHn_YH + 6 * (i)), &reg_val_y_msb, &g_board_i2c_master_ctrl, 1);

        /* extract touch point MSB and merge with LSB */
        reg_val_x = ((reg_val_x_msb & 0xF) << 8) | reg_val_x_lsb;
        reg_val_y = ((reg_val_y_msb & 0xF) << 8) | reg_val_y_lsb;

        data->point[i].x = reg_val_x;
        data->point[i].y = reg_val_y;
    }
    ...
}
```

| 処理 | 説明 |
|------|------|
| スケーリング | なし（タッチ座標=画面座標） |
| キャリブレーション | なし（工場出荷時キャリブレーション済み） |
| フィルタリング | なし |

## 4. タッチ座標領域の定義

### 4.1 ボタン/タッチ領域の定義方法

**領域構造体定義**: `tp_thread_entry.h:31-35`
```c
typedef struct
{
    lv_point_t top_left;
    lv_point_t bottom_right;
} st_bounding_box_t;
```

**領域定義の実例**: `tp_thread_entry.c:67-75`
```c
st_bounding_box_t hb_menu     = {{  12,  12}, {  60,  50}};
st_bounding_box_t hb_cancel   = {{ 970,  12}, { 1020,  50}};

st_bounding_box_t menu_cancel = {{ 840,  78}, { 880, 124}}; // over sized hit box as button is small
st_bounding_box_t kis_demo    = {{ 164, 180}, { 280, 305}};
st_bounding_box_t gs_demo     = {{ 314, 180}, { 430, 305}};
st_bounding_box_t led_demo    = {{ 454, 180}, { 570, 305}};
st_bounding_box_t lcd_demo    = {{ 598, 180}, { 712, 305}};
st_bounding_box_t ns_demo     = {{ 742, 180}, { 856, 305}};
```

### 4.2 画面ごとの領域定義

| 画面名 | 領域数 | 定義場所 | 説明 |
|--------|--------|----------|------|
| メイン画面 | 2 | tp_thread_entry.c:67-68 | メニューアイコン、キャンセル |
| ポップアップメニュー | 6 | tp_thread_entry.c:70-75 | キャンセル、各デモ選択 |
| KIS デモ | 2 | 共通領域使用 | メニュー、キャンセル |
| LED デモ | 2+ | menu_led.c | メニュー、キャンセル、LED操作領域 |
| LCD デモ | 2 | 共通領域使用 | メニュー、キャンセル |
| NS デモ | 2 | 共通領域使用 | メニュー、キャンセル |

### 4.3 ヒットボックス一覧

| 領域名 | 左上座標 | 右下座標 | サイズ (W×H) |
|--------|----------|----------|--------------|
| hb_menu | (12, 12) | (60, 50) | 48×38 |
| hb_cancel | (970, 12) | (1020, 50) | 50×38 |
| menu_cancel | (840, 78) | (880, 124) | 40×46 |
| kis_demo | (164, 180) | (280, 305) | 116×125 |
| gs_demo | (314, 180) | (430, 305) | 116×125 |
| led_demo | (454, 180) | (570, 305) | 116×125 |
| lcd_demo | (598, 180) | (712, 305) | 114×125 |
| ns_demo | (742, 180) | (856, 305) | 114×125 |

## 5. タッチ判定処理

### 5.1 タッチ状態の種類

**タッチイベント定義**: `touch_FT5316.h:63-70`
```c
typedef enum
{
    TOUCH_EVENT_NONE,
    TOUCH_EVENT_DOWN,
    TOUCH_EVENT_HOLD,
    TOUCH_EVENT_MOVE,
    TOUCH_EVENT_UP
} touch_event_t;
```

**入力デバイス状態**: `common_utils.h:87-90`
```c
typedef enum {
    LV_INDEV_STATE_RELEASED = 0,
    LV_INDEV_STATE_PRESSED
} lv_indev_state_t;
```

### 5.2 領域判定ロジック

**判定処理実装**: `tp_thread_entry.c:142-286`

```c
int8_t process_menu_detection (lv_indev_data_t *data, bool_t enable_tp_read, int8_t active_menu_selection)
{
    ...
    if (1 == data->number_of_coordinates)
    {
        local.x = data->point[0].x;
        local.y = data->point[0].y;

        if (!g_overlay_selected)
        {
            /* Process Menu ICON detection */
            if ((local.x > hb_menu.top_left.x) && (local.x < hb_menu.bottom_right.x) &&
               (local.y > hb_menu.top_left.y) && (local.y < hb_menu.bottom_right.y))
            {
                g_overlay_selected = true;
                selection = LCD_FULL_BG_POPUP_MENU;
            }
            ...
        }
        ...
    }
    return (selection);
}
```

### 5.3 ジェスチャー認識

本実装では高度なジェスチャー認識は実装されていない。シンプルなタップ（タッチ＆リリース）検出のみ。

| ジェスチャー | 認識条件 | 実装状況 |
|-------------|----------|----------|
| タップ | タッチダウン後のリリース | ○ 実装済み |
| ロングプレス | - | × 未実装 |
| スワイプ | - | × 未実装 |
| ピンチ | - | × 未実装 |

## 6. イベント処理フロー

### 6.1 タッチ検出からイベント発火までのフロー

```
タッチパネル上でタッチ発生
        │
        ▼
FT5316 タッチコントローラ検出
        │
        ▼
INTピン (P111/IRQ19) がHIGHに遷移
        │
        ▼
r_icu_isr() [vector_data.c:36]
        │
        ▼
irq19_tp_callback() [touch_FT5316.c:134-151]
        │
        ├── s_touch_panel_int = true
        │
        └── xSemaphoreGiveFromISR(g_irq_binary_semaphore)
                │
                ▼
        portYIELD_FROM_ISR() でタスク切り替え
                │
                ▼
tp_thread (touchpad_read ループ) [tp_thread_entry.c:324-329]
        │
        ▼
touchpad_is_pressed() [tp_thread_entry.c:453-484]
        │
        ├── xSemaphoreTake(g_irq_binary_semaphore)
        │
        └── 補完ポーリング (必要に応じて)
                │
                ▼
touchpad_get_xy() [tp_thread_entry.c:496-549]
        │
        ├── I2C経由でFT5316からタッチデータ読み出し
        │
        └── lv_indev_data_t構造体にデータ格納
                │
                ▼
touchpad_read() [tp_thread_entry.c:556-592]
        │
        ├── タッチ状態判定 (PRESSED/RELEASED)
        │
        └── g_tp_copy_data にコピー (タスククリティカル)
                │
                ▼
各デモスレッド (menu_main, menu_led, etc.)
        │
        ├── touchpad_get_copy() でタッチデータ取得
        │
        └── process_menu_detection() で領域判定
                │
                ▼
アプリケーション処理 (デモ切り替え等)
```

### 6.2 シーケンス図

```
  ユーザー    FT5316    IRQ19/ICU    tp_thread    main_menu_thread
     │          │           │            │              │
     │ タッチ   │           │            │              │
     │─────────>│           │            │              │
     │          │ INT出力   │            │              │
     │          │──────────>│            │              │
     │          │           │ ISR発火    │              │
     │          │           │───────────>│              │
     │          │           │ セマフォ   │              │
     │          │           │ Give       │              │
     │          │           │            │ Wake         │
     │          │           │            │──────┐       │
     │          │           │            │      │       │
     │          │ I2C Read  │            │<─────┘       │
     │          │<──────────────────────│              │
     │          │ データ    │            │              │
     │          │──────────────────────>│              │
     │          │           │            │ データ更新   │
     │          │           │            │──────────────>│
     │          │           │            │              │ 領域判定
     │          │           │            │              │ イベント処理
```

### 6.3 スレッド構成

| スレッド名 | 優先度 | スタックサイズ | 役割 |
|------------|--------|----------------|------|
| tp_thread | 15 | 4096バイト | タッチデータ取得・更新 |
| main_menu_thread | 1 | 4096バイト | メインメニュー、タッチイベント処理 |

**スレッド定義**: `tp_thread.c:30-44`
```c
tp_thread = xTaskCreateStatic (
                               tp_thread_func,
                               (const char*) "TP Thread", 4096 / 4, // In words, not bytes
                               (void*) &tp_thread_parameters, //pvParameters
                               15,
                               (StackType_t*) &tp_thread_stack,
                               (StaticTask_t*) &tp_thread_memory
                               );
```

## 7. 割込み設定

### 7.1 外部割込み（IRQ19）設定

**設定箇所**: `common_data.c:610-627`

| 設定項目 | 値 |
|----------|-----|
| チャネル | 19 |
| トリガ | 立ち上がりエッジ (RISING) |
| デジタルフィルタ | 無効 |
| 割込み優先度 | 2 |
| コールバック | irq19_tp_callback |

```c
const external_irq_cfg_t g_external_irq19_cfg =
{ .channel = 19, .trigger = EXTERNAL_IRQ_TRIG_RISING, .filter_enable = false, .clock_source_div =
          EXTERNAL_IRQ_CLOCK_SOURCE_DIV_64,
  .p_callback = irq19_tp_callback,
  ...
  .ipl = (2),
```

### 7.2 割込みコールバック

**コールバック実装**: `touch_FT5316.c:134-151`

```c
void irq19_tp_callback(external_irq_callback_args_t *p_args)
{
    UNUSED_PARAM(p_args);

    s_touch_panel_int = true;

    /* tell touch pad thread */
    BaseType_t xHigherPriorityTaskWoken;
    BaseType_t xResult;

    xResult = xSemaphoreGiveFromISR(g_irq_binary_semaphore, &xHigherPriorityTaskWoken);
    s_irq19_valid = true;

    if (pdFAIL != xResult)
    {
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
}
```

## 8. タッチシステムブロック図

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Application Layer                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  menu_main   │  │  menu_led    │  │  menu_lcd    │  │  menu_kis    │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                 │                 │                 │              │
│         └────────────────┬┴─────────────────┴─────────────────┘              │
│                          │                                                   │
│                          ▼                                                   │
│              ┌───────────────────────┐                                       │
│              │ process_menu_detection │ タッチ領域判定                        │
│              └───────────┬───────────┘                                       │
│                          │                                                   │
├──────────────────────────┼───────────────────────────────────────────────────┤
│                          ▼                                                   │
│              ┌───────────────────────┐                                       │
│              │   touchpad_get_copy   │ タッチデータ取得API                    │
│              └───────────┬───────────┘                                       │
│                          │                                                   │
│        Touch Panel Layer │                                                   │
│              ┌───────────┴───────────┐                                       │
│              │      tp_thread        │ タッチパネルスレッド                   │
│              │  ┌─────────────────┐  │                                       │
│              │  │  touchpad_read  │  │ 10ms周期ポーリング                     │
│              │  └────────┬────────┘  │                                       │
│              └───────────┼───────────┘                                       │
│                          │                                                   │
├──────────────────────────┼───────────────────────────────────────────────────┤
│                          ▼                                                   │
│              ┌───────────────────────┐                                       │
│              │   touchpad_get_xy     │ I2C通信処理                            │
│              └───────────┬───────────┘                                       │
│                          │                                                   │
│        Driver Layer      │                                                   │
│              ┌───────────┴───────────┐                                       │
│              │    touch_FT5316.c     │ FT5316タッチドライバ                   │
│              │  ┌─────────────────┐  │                                       │
│              │  │   read_reg8     │  │ レジスタ読み取り                       │
│              │  └────────┬────────┘  │                                       │
│              └───────────┼───────────┘                                       │
│                          │                                                   │
├──────────────────────────┼───────────────────────────────────────────────────┤
│                          ▼                                                   │
│        HAL Layer   ┌──────────────────┐    ┌──────────────────┐              │
│                    │  R_IIC_MASTER    │    │  R_ICU_ExternalIrq │            │
│                    │    (I2C HAL)     │    │    (IRQ HAL)      │             │
│                    └────────┬─────────┘    └────────┬─────────┘              │
│                             │                       │                        │
├─────────────────────────────┼───────────────────────┼────────────────────────┤
│                             ▼                       ▼                        │
│        Hardware      ┌──────────────┐        ┌──────────────┐                │
│                      │    IIC1      │        │   ICU IRQ19  │                │
│                      │ (SDA/SCL)    │        │   (P111)     │                │
│                      └──────┬───────┘        └──────┬───────┘                │
│                             │                       │                        │
│                             └───────────┬───────────┘                        │
│                                         ▼                                    │
│                               ┌──────────────────┐                           │
│                               │     FT5316       │                           │
│                               │ Touch Controller │                           │
│                               └────────┬─────────┘                           │
│                                        │                                     │
│                               ┌────────┴─────────┐                           │
│                               │  Capacitive LCD  │                           │
│                               │   Touch Panel    │                           │
│                               │   (1024×600)     │                           │
│                               └──────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 9. カスタマイズガイド

### 9.1 新しいタッチ領域の追加方法

#### 手順1: 領域定義の追加

`tp_thread_entry.c` に新しい `st_bounding_box_t` 変数を追加:

```c
// 例: 新しいボタン領域を追加
st_bounding_box_t my_button = {{ 100, 400}, { 200, 450}};  // (100,400)-(200,450)
```

#### 手順2: 判定ロジックの追加

`process_menu_detection()` 関数内に判定処理を追加:

```c
/* My Button detection */
if ((local.x > my_button.top_left.x) && (local.x < my_button.bottom_right.x) &&
   (local.y > my_button.top_left.y) && (local.y < my_button.bottom_right.y))
{
    selection = 'M';  // 任意の選択コードを返す
}
```

#### 手順3: アプリケーション処理の追加

対応するデモ/メニューファイルで選択コードを処理:

```c
switch (on_screen_selection)
{
    case 'M':
        // my_button が押された時の処理
        handle_my_button_press();
        break;
}
```

### 9.2 タッチ感度の調整方法

FT5316の感度はコントローラ内部で管理されており、ファームウェアからの直接調整は限定的。

#### レジスタによる調整（可能な場合）

`write_reg8()` を使用してFT5316レジスタを設定:

```c
// 例: 感度関連レジスタへの書き込み（レジスタアドレスはデータシート参照）
write_reg8(&g_board_i2c_master_ctrl, SENSITIVITY_REG, value);
```

#### ソフトウェアフィルタリング

`touchpad_get_xy()` にフィルタリング処理を追加:

```c
// 簡易ローパスフィルタの例
static uint16_t last_x = 0, last_y = 0;
data->point[i].x = (reg_val_x + last_x) / 2;
data->point[i].y = (reg_val_y + last_y) / 2;
last_x = reg_val_x;
last_y = reg_val_y;
```

### 9.3 ジェスチャー追加方法

#### ロングプレス検出の例

```c
// tp_thread_entry.c に追加
static uint32_t touch_start_time = 0;
static bool long_press_detected = false;

void check_long_press(lv_indev_data_t *data)
{
    if (data->state == LV_INDEV_STATE_PRESSED)
    {
        if (touch_start_time == 0)
        {
            touch_start_time = xTaskGetTickCount();
        }
        else if ((xTaskGetTickCount() - touch_start_time) > pdMS_TO_TICKS(1000))
        {
            if (!long_press_detected)
            {
                long_press_detected = true;
                // ロングプレス処理を実行
                on_long_press(data->point[0].x, data->point[0].y);
            }
        }
    }
    else
    {
        touch_start_time = 0;
        long_press_detected = false;
    }
}
```

#### スワイプ検出の例

```c
// 開始位置と終了位置を比較
static lv_point_t swipe_start = {0, 0};
static bool swipe_tracking = false;
#define SWIPE_THRESHOLD 100  // ピクセル

void check_swipe(lv_indev_data_t *data)
{
    if (data->state == LV_INDEV_STATE_PRESSED && !swipe_tracking)
    {
        swipe_start = data->point[0];
        swipe_tracking = true;
    }
    else if (data->state == LV_INDEV_STATE_RELEASED && swipe_tracking)
    {
        int32_t dx = data->point[0].x - swipe_start.x;
        int32_t dy = data->point[0].y - swipe_start.y;

        if (dx > SWIPE_THRESHOLD) on_swipe_right();
        else if (dx < -SWIPE_THRESHOLD) on_swipe_left();
        else if (dy > SWIPE_THRESHOLD) on_swipe_down();
        else if (dy < -SWIPE_THRESHOLD) on_swipe_up();

        swipe_tracking = false;
    }
}
```

## 10. デバッグ方法

### 10.1 タッチ座標の確認方法

#### コンソール出力

`touchpad_read()` または `touchpad_get_xy()` にデバッグ出力を追加:

```c
#include "jlink_console.h"

// touchpad_get_xy() 内に追加
if (tp_detected > 0)
{
    char debug_buf[64];
    sprintf(debug_buf, "Touch: (%d, %d) count=%d\r\n",
            data->point[0].x, data->point[0].y, tp_detected);
    print_to_console(debug_buf);
}
```

#### ヒットボックス可視化

`draw_hitboxes()` 関数が実装されており、Dave/2Dで領域を描画可能:

```c
void draw_hitboxes(void)
{
    if(false == g_overlay_selected)
    {
        if (g_current_backgroud.menu_icon_active)
        {
            d2_setcolor(gp_d2_handle, 0, 0x00FF0000);  // 赤色
            d2_renderbox( gp_d2_handle,
                (d2_point)hb_menu.top_left.x << 4,
                (d2_point)hb_menu.top_left.y << 4,
                (d2_point)(hb_menu.bottom_right.x - hb_menu.top_left.x) << 4,
                (d2_point)(hb_menu.bottom_right.y - hb_menu.top_left.y) << 4);
        }
        ...
    }
}
```

### 10.2 タッチイベントのログ出力

#### 割込みカウンタ

```c
// touch_FT5316.c に追加
static volatile uint32_t irq_count = 0;

void irq19_tp_callback(external_irq_callback_args_t *p_args)
{
    UNUSED_PARAM(p_args);
    irq_count++;  // 割込み回数をカウント
    ...
}

uint32_t get_irq_count(void) { return irq_count; }
```

#### イベント履歴

```c
// デバッグ用イベント履歴
#define EVENT_HISTORY_SIZE 32
typedef struct {
    uint32_t timestamp;
    uint16_t x;
    uint16_t y;
    touch_event_t event;
} touch_event_log_t;

static touch_event_log_t event_history[EVENT_HISTORY_SIZE];
static uint8_t event_index = 0;

void log_touch_event(uint16_t x, uint16_t y, touch_event_t event)
{
    event_history[event_index].timestamp = xTaskGetTickCount();
    event_history[event_index].x = x;
    event_history[event_index].y = y;
    event_history[event_index].event = event;
    event_index = (event_index + 1) % EVENT_HISTORY_SIZE;
}
```

### 10.3 トラブルシューティング

| 症状 | 考えられる原因 | 対処法 |
|------|---------------|--------|
| タッチ反応なし | I2C通信エラー | I2C波形確認、プルアップ抵抗確認 |
| タッチ座標ズレ | キャリブレーション不良 | FT5316リセット、座標オフセット調整 |
| タッチ反応遅延 | 割込み処理遅延 | 優先度確認、I2C速度向上 |
| 誤タッチ | ノイズ/EMI | フィルタリング追加、シールド強化 |
| 断続的なタッチ検出 | 割込みレート不足 | ポーリング補完実装（実装済み） |

#### I2C通信確認

```c
// タッチコントローラ接続確認
uint8_t status = read_ft5316_tp_status();
if (status == 0xFF)
{
    // 通信エラー（未初期化値が返される）
    print_to_console("FT5316 communication error!\r\n");
}
```

## 11. 関連ファイル一覧

| ファイル | 役割 |
|----------|------|
| `src/touch_FT5316.c` | FT5316タッチドライバ実装 |
| `src/touch_FT5316.h` | タッチドライバヘッダ、レジスタ定義 |
| `src/tp_thread_entry.c` | タッチパネルスレッド、領域判定 |
| `src/tp_thread_entry.h` | タッチ領域構造体定義 |
| `src/common_utils.h` | lv_indev_data_t、lv_point_t定義 |
| `src/board_i2c_master.c` | I2Cマスタ通信処理 |
| `ra_gen/common_data.c` | 外部割込み設定、セマフォ定義 |
| `ra_gen/hal_data.c` | I2C HAL設定 |
| `ra_gen/tp_thread.c` | スレッド生成コード |

## 12. まとめ

EK-RA8P1のタッチ入力システムは以下の特徴を持つ:

1. **ハードウェア構成**: FT5316静電容量式タッチコントローラ（最大5点マルチタッチ対応）
2. **通信方式**: I2C（IIC1チャネル、Standard Mode約100kHz）
3. **割込み方式**: 外部割込み（IRQ19）+ ポーリング補完のハイブリッド
4. **タッチ領域**: st_bounding_box_t構造体による矩形領域定義
5. **イベント処理**: FreeRTOSセマフォによるスレッド間通知

シンプルなタップ検出を中心とした実装であり、ジェスチャー認識などの高度な機能は未実装だが、st_bounding_box_tとprocess_menu_detection()の拡張により容易にカスタマイズ可能な設計となっている。
