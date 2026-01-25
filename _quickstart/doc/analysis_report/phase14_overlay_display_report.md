# Phase 14: オーバーレイ表示の仕組み 解析レポート

## 1. エグゼクティブサマリー

本解析では、EK-RA8P1 quickstartプロジェクトにおけるカメラ画像上へのタッチ位置オーバーレイ表示の仕組みを詳細に調査した。GLCDCのハードウェアレイヤー合成とDave/2D（DRW）2Dグラフィックスエンジンを組み合わせた効率的なオーバーレイ実装を解明した。

## 2. オーバーレイの実現方法

### 2.1 実現方式の特定

| 方式 | 説明 | 使用有無 |
|------|------|----------|
| GLCDCレイヤー | ハードウェアレイヤー合成 | **使用** |
| アルファブレンディング | 透明度による合成 | **使用** (ARGB4444) |
| カラーキー | 特定色を透明として扱う | 未使用 |
| 直接描画 | フレームバッファへ直接描画 | 未使用 |

**実際の方式**: GLCDCの2レイヤー構成 + Dave/2D (DRW) ハードウェアアクセラレータによる描画

### 2.2 アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────┐
│                    LCD Panel (1024 x 600)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │ ハードウェア合成
                           │
┌──────────────────────────┴──────────────────────────────────┐
│                         GLCDC                                │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Layer 2 (前面): ARGB4444 - オーバーレイ / GUI          │ │
│  │  - タッチマーカー（円形）                                │ │
│  │  - テキスト、アイコン                                   │ │
│  │  - ポップアップメニュー                                  │ │
│  │  - 透明部分はLayer 1が透過                              │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Layer 1 (背面): RGB565 - カメラ映像                    │ │
│  │  - VINからの直接バッファ参照                            │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Background: 黒 (ARGB: 255, 0, 0, 0)                    │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 3. レイヤー構成

### 3.1 レイヤー設定詳細

| レイヤー | 用途 | フォーマット | サイズ | 座標オフセット | 優先度 |
|---------|------|-------------|--------|----------------|--------|
| Background | 背景色 | ARGB | - | - | 最低 |
| Layer 1 (input[0]) | カメラ画像 | RGB565 | 768 x 450 | (128, 75) | 中 |
| Layer 2 (input[1]) | オーバーレイ | ARGB4444 | 1024 x 600 | (0, 0) | 最高 |

### 3.2 Layer 1 (カメラ映像レイヤー) 設定

**出典**: `ra_gen/common_data.c:432-443`
```c
.input[0] =
{
    .p_base              = (uint32_t *)&fb_background[0],
    .hsize = DISPLAY_HSIZE_INPUT0,      // 768
    .vsize = DISPLAY_VSIZE_INPUT0,      // 450
    .hstride = DISPLAY_BUFFER_STRIDE_PIXELS_INPUT0,
    .format = DISPLAY_IN_FORMAT_16BITS_RGB565,
    .line_descending_enable = false,
    .lines_repeat_enable = false,
    .lines_repeat_times = 0
},

.layer[0] =
{
    .coordinate = { .x = 128, .y = 75 },  // 画面中央寄りに配置
    .fade_control = DISPLAY_FADE_CONTROL_NONE,
    .fade_speed = 255
},
```

### 3.3 Layer 2 (オーバーレイレイヤー) 設定

**出典**: `ra_gen/common_data.c:445-468`
```c
.input[1] =
{
    .p_base              = (uint32_t *)&fb_foreground[0],
    .hsize = DISPLAY_HSIZE_INPUT1,      // 1024
    .vsize = DISPLAY_VSIZE_INPUT1,      // 600
    .hstride = DISPLAY_BUFFER_STRIDE_PIXELS_INPUT1,
    .format = DISPLAY_IN_FORMAT_16BITS_ARGB4444,  // アルファチャネル付き
    .line_descending_enable = false,
    .lines_repeat_enable = false,
    .lines_repeat_times = 0
},

.layer[1] =
{
    .coordinate = { .x = 0, .y = 0 },  // 画面左上から全画面
    .fade_control = DISPLAY_FADE_CONTROL_NONE,
    .fade_speed = 255
},
```

### 3.4 背景色設定

**出典**: `ra_gen/common_data.c:482-484`
```c
.bg_color =
{
    .byte = { .a = 255, .r = 0, .g = 0, .b = 0 }  // 黒（完全不透明）
},
```

### 3.5 フレームバッファ構成

| バッファ | サイズ | 配置 | 用途 |
|----------|--------|------|------|
| fb_background[1] | 768 × 450 × 2 bytes | SDRAM (noinit) | Layer 1 (カメラ代替) |
| fb_foreground[2] | 1024 × 600 × 2 bytes | SDRAM (noinit) | Layer 2 (ダブルバッファ) |

**出典**: `ra_gen/common_data.c:313-322`
```c
#if GLCDC_CFG_LAYER_1_ENABLE
uint8_t fb_background[1][DISPLAY_BUFFER_STRIDE_BYTES_INPUT0 * DISPLAY_VSIZE_INPUT0]
    BSP_ALIGN_VARIABLE(64) BSP_PLACE_IN_SECTION(BSP_UNINIT_SECTION_PREFIX ".sdram_noinit");
#endif

#if GLCDC_CFG_LAYER_2_ENABLE
uint8_t fb_foreground[2][DISPLAY_BUFFER_STRIDE_BYTES_INPUT1 * DISPLAY_VSIZE_INPUT1]
    BSP_ALIGN_VARIABLE(64) BSP_PLACE_IN_SECTION(BSP_UNINIT_SECTION_PREFIX ".sdram_noinit");
#endif
```

## 4. Dave/2D (DRW) グラフィックスエンジン

### 4.1 Dave/2D初期化設定

**出典**: `src/display_thread_entry.c:193-208`
```c
gp_d2_handle = d2_opendevice(0);
d2_inithw(gp_d2_handle, 0);

renderbuffer = d2_newrenderbuffer(gp_d2_handle, 20, 20);
d2_selectrenderbuffer(gp_d2_handle, renderbuffer);

// アルファブレンディング設定
d2_setblendmode(gp_d2_handle, d2_bm_alpha, d2_bm_zero);
d2_setalphamode(gp_d2_handle, d2_am_constant);
d2_setalpha(gp_d2_handle, 0xff);

// アンチエイリアシング無効
d2_setantialiasing(gp_d2_handle, 0);

// ラインキャップ/ジョイン設定
d2_setlinecap(gp_d2_handle, d2_lc_butt);
d2_setlinejoin(gp_d2_handle, d2_lj_max);
```

### 4.2 アルファブレンディングパラメータ

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| blend mode (src) | d2_bm_alpha | ソースアルファを使用 |
| blend mode (dst) | d2_bm_zero | デスティネーションを0に |
| alpha mode | d2_am_constant | 定数アルファモード |
| alpha value | 0xff | 完全不透明 |

## 5. タッチマーカー描画

### 5.1 タッチマーカーの仕様

| 項目 | 値 |
|------|-----|
| 形状 | **円形** |
| 半径 | 80 ピクセル (5 pixels in display) |
| 最大同時表示数 | 5点 |

### 5.2 タッチマーカーのカラーパレット

**出典**: `src/menu_lcd.c:71-78`
```c
static const d2_color lcd_demo_clut[] =
{
    0x00FF0000, // RED     - 1番目のタッチポイント
    0x00F77500, // ORANGE  - 2番目のタッチポイント
    0x00FFD800, // YELLOW  - 3番目のタッチポイント
    0x0000FF00, // GREEN   - 4番目のタッチポイント
    0x000000FF, // BLUE    - 5番目のタッチポイント
};
```

| インデックス | 色 | RGB値 |
|-------------|-----|-------|
| 0 | 赤 | 0xFF0000 |
| 1 | オレンジ | 0xF77500 |
| 2 | 黄 | 0xFFD800 |
| 3 | 緑 | 0x00FF00 |
| 4 | 青 | 0x0000FF |

### 5.3 タッチマーカー描画処理

**出典**: `src/menu_lcd.c:209-231`
```c
void menu_lcd_update_screen(d2_device *handle)
{
    lv_indev_data_t data;

    /* Only if the overlay menu is not displayed */
    if (!g_overlay_selected && lcd_touch_ready)
    {
        touchpad_get_copy(&data);

        uint32_t limit = data.number_of_coordinates;

        for (uint8_t i = 0; i < 5; i++)
        {
            if (i < limit)
            {
                // 色を設定
                d2_setcolor(handle, 0, lcd_demo_clut[i]);

                // 円を描画 (座標は4ビット固定小数点)
                d2_rendercircle(handle,
                    (d2_point) data.point[i].x << 4,   // X座標
                    (d2_point) data.point[i].y << 4,   // Y座標
                    80 << 4,                           // 半径 (80ピクセル)
                    0 << 4);                           // 内径 (0 = 塗りつぶし)
            }
        }
    }
}
```

### 5.4 座標系と固定小数点

Dave/2Dは4ビット固定小数点座標を使用：
- 実際の座標値を左に4ビットシフト (`<< 4`)
- 例: X=100 → `100 << 4` = 1600

## 6. オーバーレイ描画シーケンス

### 6.1 メインループ描画フロー

```
display_thread_entry()
    │
    └─→ while(1) ループ
         │
         ├─→ graphics_wait_vsync()         // VSync待ち
         │
         ├─→ d2_selectrenderbuffer()       // レンダーバッファ選択
         │
         ├─→ d2_framebuffer()              // ターゲットバッファ設定
         │       └─→ fb_foreground[s_drw_buf]
         │
         ├─→ d2_cliprect()                 // クリップ領域設定
         │       └─→ (0, 0) - (1023, 599)
         │
         ├─→ set_background()              // 背景画像ソース設定
         │
         ├─→ draw_background()             // 背景を描画
         │       └─→ d2_blitcopy() 1024x600
         │
         ├─→ call_user_draw()              // 画面固有の描画
         │       └─→ case LCD_FULL_BG_CAMERA_BACKGROUND:
         │               d2_clear(0x00000000)  // 透明クリア
         │               menu_lcd_update_screen()
         │                   └─→ タッチマーカー描画
         │
         ├─→ draw_menu_icon()              // MENUアイコン
         │
         ├─→ draw_exit_icon()              // 終了アイコン
         │
         ├─→ draw_overlay()                // ポップアップメニュー
         │       └─→ g_overlay_selected時のみ描画
         │
         ├─→ d2_executerenderbuffer()      // 描画コマンド実行
         │
         ├─→ d2_flushframe()               // フレーム完了待ち
         │
         ├─→ s_drw_buf = s_drw_buf ? 0 : 1 // バッファ切り替え
         │
         ├─→ R_GLCDC_BufferChange(LAYER_1) // カメラバッファ更新
         │
         └─→ R_GLCDC_BufferChange(LAYER_2) // フォアグラウンド更新
```

### 6.2 タッチ座標からオーバーレイ描画までのフロー

```
┌───────────────────┐
│   タッチ検出      │ (tp_thread)
│   IRQ19 割り込み  │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   座標取得        │
│   touchpad_read() │
│   FT5316 I2C読取  │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   座標コピー      │
│ touchpad_get_copy │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   描画バッファ    │ (display_thread)
│   クリア          │
│   d2_clear()      │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   マーカー描画    │
│ d2_setcolor()     │
│ d2_rendercircle() │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   レンダリング    │
│  実行             │
│d2_executerender.. │
│d2_flushframe()    │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│   表示更新        │
│R_GLCDC_Buffer..   │
│  (VSync同期)      │
└───────────────────┘
```

## 7. ポップアップメニューオーバーレイ

### 7.1 オーバーレイ描画処理

**出典**: `src/display_thread_entry.c:358-377`
```c
static void draw_overlay(void)
{
    if (true == g_overlay_selected)
    {
        // オーバーレイ画像をブリットソースに設定
        d2_setblitsrc(gp_d2_handle,
            (uint8_t *)&g_menu_overlay_image.pixel_data[0],
            g_menu_overlay_image.width,
            g_menu_overlay_image.width,
            g_menu_overlay_image.height,
            d2_mode_rgb565);

        // オーバーレイ画像を描画
        d2_blitcopy(gp_d2_handle,
            (d2_s32) g_menu_overlay_image.width,
            (d2_s32) g_menu_overlay_image.height,
            (d2_blitpos) 0, (d2_blitpos) 0,
            (d2_width) ((uint32_t)(g_menu_overlay_image.width) << 4),
            (d2_width) ((uint32_t)(g_menu_overlay_image.height) << 4),
            (d2_point) (128 << 4),   // 表示位置X
            (d2_point) (74 << 4),    // 表示位置Y
            d2_tm_filter);

        /* 背景ブリットバッファをリセット */
        graphics_reset_blit_buffer();
    }
}
```

### 7.2 オーバーレイ表示位置

| 項目 | 値 |
|------|-----|
| 表示開始X | 128 ピクセル |
| 表示開始Y | 74 ピクセル |

## 8. パフォーマンス最適化

### 8.1 ハードウェアアクセラレーション

| 機能 | 使用エンジン |
|------|-------------|
| 円描画 | Dave/2D (DRW) |
| 画像ブリット | Dave/2D (DRW) |
| レイヤー合成 | GLCDC |
| YUV→RGB変換 | VIN |

### 8.2 ダブルバッファリング

- Layer 2は `fb_foreground[2]` でダブルバッファ構成
- `s_drw_buf` で現在の描画先を管理
- VSync同期でティアリング防止

**出典**: `src/display_thread_entry.c:262, 273`
```c
s_drw_buf = s_drw_buf ? 0 : 1;

err = R_GLCDC_BufferChange(g_plcd_display.p_ctrl,
    graphics_get_back_buffer(),
    DISPLAY_FRAME_LAYER_2);
```

### 8.3 更新方式

| 方式 | 説明 | 採用 |
|------|------|------|
| 全画面更新 | 毎フレーム全画面を再描画 | **採用** |
| 部分更新 | 変更領域のみ更新 | 未採用 |

カメラ映像は常に更新されるため、全画面更新方式を採用。

### 8.4 VSync同期

**出典**: `src/display_thread_entry.c:229`
```c
graphics_wait_vsync();
```

- 垂直同期信号に同期してバッファ切り替え
- ティアリング（画面の裂け）を防止
- 60fps固定フレームレート

## 9. レイヤー合成図

```
出力画面 (1024 x 600)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ┌───────────────────────────────────────────────────────┐     │
│   │                                                       │     │
│   │   Layer 2 (ARGB4444) - フォアグラウンド              │     │
│   │   ┌─────────────────────────────────────────────┐     │     │
│   │   │                                             │     │     │
│   │   │  ┌────┐                          ┌────┐     │     │     │
│   │   │  │MENU│       タイトル           │ X  │     │     │     │
│   │   │  └────┘                          └────┘     │     │     │
│   │   │                                             │     │     │
│   │   │        ●  ●  ●  ●  ●                        │     │     │
│   │   │       タッチマーカー (5色の円)              │     │     │
│   │   │                                             │     │     │
│   │   │   透明部分 → Layer 1 が透過                 │     │     │
│   │   │                                             │     │     │
│   │   └─────────────────────────────────────────────┘     │     │
│   │                                                       │     │
│   └───────────────────────────────────────────────────────┘     │
│                                                                 │
│       ┌───────────────────────────────────────────┐             │
│       │                                           │             │
│       │   Layer 1 (RGB565) - カメラ映像          │             │
│       │   768 x 450 @ offset (128, 75)           │             │
│       │                                           │             │
│       │   VINバッファから直接参照                 │             │
│       │                                           │             │
│       └───────────────────────────────────────────┘             │
│                                                                 │
│   Background: 黒 (0, 0, 0)                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 10. 主要関数一覧

### 10.1 描画関連関数

| 関数名 | ファイル | 役割 |
|--------|----------|------|
| `display_thread_entry()` | display_thread_entry.c:173 | メイン描画ループ |
| `call_user_draw()` | display_thread_entry.c:293 | 画面固有の描画処理 |
| `draw_overlay()` | display_thread_entry.c:358 | ポップアップメニュー描画 |
| `draw_background()` | display_thread_entry.c:388 | 背景画像描画 |
| `draw_menu_icon()` | display_thread_entry.c | MENUアイコン描画 |
| `draw_exit_icon()` | display_thread_entry.c | 終了アイコン描画 |
| `menu_lcd_update_screen()` | menu_lcd.c:209 | タッチマーカー描画 |
| `graphics_reset_blit_buffer()` | display_thread_entry.c:131 | ブリットソース切替 |

### 10.2 Dave/2D API

| 関数名 | 役割 |
|--------|------|
| `d2_opendevice()` | デバイス初期化 |
| `d2_newrenderbuffer()` | レンダーバッファ作成 |
| `d2_selectrenderbuffer()` | レンダーバッファ選択 |
| `d2_framebuffer()` | ターゲットバッファ設定 |
| `d2_setblendmode()` | ブレンドモード設定 |
| `d2_setalphamode()` | アルファモード設定 |
| `d2_setalpha()` | アルファ値設定 |
| `d2_setcolor()` | 描画色設定 |
| `d2_rendercircle()` | 円描画 |
| `d2_renderbox()` | 矩形描画 |
| `d2_setblitsrc()` | ブリットソース設定 |
| `d2_blitcopy()` | 画像ブリット |
| `d2_clear()` | バッファクリア |
| `d2_executerenderbuffer()` | 描画コマンド実行 |
| `d2_flushframe()` | フレーム完了待ち |

### 10.3 GLCDC API

| 関数名 | 役割 |
|--------|------|
| `R_GLCDC_Open()` | GLCDCドライバ初期化 |
| `R_GLCDC_Start()` | 表示開始 |
| `R_GLCDC_BufferChange()` | バッファ切り替え |

## 11. カスタマイズガイド

### 11.1 タッチマーカーの形状変更

円形から矩形に変更する場合：

```c
// 円の代わりに矩形を描画
d2_renderbox(handle,
    (d2_point)(data.point[i].x - 40) << 4,  // 左上X
    (d2_point)(data.point[i].y - 40) << 4,  // 左上Y
    (d2_width)(80) << 4,                     // 幅
    (d2_width)(80) << 4);                    // 高さ
```

### 11.2 タッチマーカーの色変更

`lcd_demo_clut[]` 配列の値を変更：

```c
static const d2_color lcd_demo_clut[] =
{
    0x00RRGGBB, // カスタム色1
    0x00RRGGBB, // カスタム色2
    // ...
};
```

### 11.3 タッチマーカーのサイズ変更

`d2_rendercircle()` の第4引数（半径）を変更：

```c
d2_rendercircle(handle,
    (d2_point) data.point[i].x << 4,
    (d2_point) data.point[i].y << 4,
    40 << 4,  // 半径を40ピクセルに変更
    0 << 4);
```

### 11.4 新しいオーバーレイ要素の追加

`call_user_draw()` 内の `case LCD_FULL_BG_CAMERA_BACKGROUND:` ブロックに追加：

```c
case LCD_FULL_BG_CAMERA_BACKGROUND:
    d2_clear(gp_d2_handle, 0x00000000);

    // カスタムオーバーレイ描画
    d2_setcolor(gp_d2_handle, 0, 0x00FFFFFF);  // 白色
    d2_renderbox(gp_d2_handle,
        (d2_point)(10) << 4, (d2_point)(10) << 4,
        (d2_width)(100) << 4, (d2_width)(50) << 4);

    menu_lcd_update_screen(gp_d2_handle);  // タッチマーカー
    break;
```

## 12. 解析所見

### 12.1 設計の特徴

1. **効率的なハードウェア活用**: GLCDCのレイヤー合成とDave/2Dのハードウェアアクセラレーションを組み合わせ
2. **柔軟なアルファブレンディング**: ARGB4444フォーマットによるピクセル単位の透明度制御
3. **ダブルバッファリング**: ティアリングのない滑らかな表示更新
4. **ゼロコピーカメラ表示**: VINバッファをGLCDCが直接参照

### 12.2 パフォーマンス考慮事項

1. **描画負荷**: 毎フレーム全画面を再描画するため、複雑なGUIでは負荷増加
2. **メモリ使用量**: ダブルバッファで約2.4MB (1024×600×2bytes×2バッファ)
3. **フレームレート**: 60fps固定（VSync同期）

### 12.3 制限事項

1. **最大タッチポイント**: 5点まで（FT5316の制限）
2. **レイヤー数**: 2レイヤー（GLCDCの制限）
3. **フォーマット**: Layer 2はARGB4444固定（16bit/pixel）

## 13. 参考ファイル一覧

| ファイル | 役割 |
|----------|------|
| `src/display_thread_entry.c` | メイン描画スレッド |
| `src/menu_lcd.c` | タッチマーカー描画、カメラデモUI |
| `ra_gen/common_data.c` | GLCDC/VIN設定 |
| `ra_gen/common_data.h` | フレームバッファ定義 |
| `ra/tes/dave2d/` | Dave/2Dライブラリ |
