# Phase 10: LCD表示の仕組み解析レポート

## 概要

本レポートでは、EK-RA8P1 QuickstartプロジェクトにおけるLCD画像表示の仕組みを詳細に解析し、描画処理の全体像を理解する。

---

## 1. ハードウェア構成

### 1.1 LCDパネルの仕様

| 項目 | 値 |
|------|-----|
| 解像度 | 1024 x 600 ピクセル |
| インターフェース | Parallel RGB (24-bit) |
| 色深度 | 16 bpp (RGB565) / 32 bpp (ARGB8888) |
| リフレッシュレート | 約60 Hz |

### 1.2 GLCDC（Graphics LCD Controller）設定

`ra_gen/common_data.c` および `ra_gen/common_data.h` より：

| 項目 | 設定値 |
|------|--------|
| 出力解像度 | 1024 x 600 |
| Layer1ピクセルフォーマット | RGB565 (16-bit) |
| Layer2ピクセルフォーマット | ARGB4444 (16-bit) |
| 出力インターフェース | Parallel RGB (TCON) |
| レイヤー数 | 2 (Layer1: 背景/カメラ, Layer2: 前景/UI) |

### 1.3 2D Drawing Engine (DRW/Dave2D)

| 項目 | 状態 |
|------|------|
| 使用有無 | **有** |
| ライブラリ | TES Dave/2D |
| 対応機能 | 直線描画、矩形塗りつぶし、ブリットコピー、アルファブレンド |
| IPL | 2 |

---

## 2. フレームバッファ

### 2.1 フレームバッファ構成

```
外部SDRAM領域 (.sdram / .sdram_noinit)
┌─────────────────────────────────────────────────────────────────┐
│ fb_background[1]                                                │
│   Layer1用フレームバッファ (カメラ画像)                          │
│   サイズ: 768 x 450 x 2 = 691,200 bytes                        │
│   フォーマット: RGB565 (16-bit)                                 │
│   アドレス: .sdram_noinit セクション                            │
├─────────────────────────────────────────────────────────────────┤
│ fb_foreground[0]                                                │
│   Layer2用フレームバッファ #0 (ダブルバッファ)                   │
│   サイズ: 1024 x 600 x 2 = 1,228,800 bytes                     │
│   フォーマット: ARGB4444 (16-bit)                               │
│   アドレス: .sdram_noinit セクション                            │
├─────────────────────────────────────────────────────────────────┤
│ fb_foreground[1]                                                │
│   Layer2用フレームバッファ #1 (ダブルバッファ)                   │
│   サイズ: 1024 x 600 x 2 = 1,228,800 bytes                     │
│   フォーマット: ARGB4444 (16-bit)                               │
│   アドレス: .sdram_noinit セクション                            │
├─────────────────────────────────────────────────────────────────┤
│ vin_image_buffer_1/2/3                                          │
│   VIN(カメラ)キャプチャ用バッファ (トリプルバッファ)             │
│   サイズ: 768 x 450 x 2 x 3 = 2,073,600 bytes                  │
│   アドレス: .sdram_noinit セクション                            │
├─────────────────────────────────────────────────────────────────┤
│ 画像リソース領域                                                 │
│   g_sdram_buffer_welcome, g_sdram_buffer_led 等                │
│   アドレス: .sdram セクション                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 フレームバッファサイズ計算

**Layer1 (カメラ/背景)**:
```
DISPLAY_HSIZE_INPUT0 = 768
DISPLAY_VSIZE_INPUT0 = 450
DISPLAY_BITS_PER_PIXEL_INPUT0 = 16
DISPLAY_BUFFER_STRIDE_BYTES_INPUT0 = ((768 * 16 + 0x1FF) >> 9) << 6 = 1536 bytes
Buffer Size = 1536 * 450 = 691,200 bytes
```

**Layer2 (UI/前景)**:
```
DISPLAY_HSIZE_INPUT1 = 1024
DISPLAY_VSIZE_INPUT1 = 600
DISPLAY_BITS_PER_PIXEL_INPUT1 = 16
DISPLAY_BUFFER_STRIDE_BYTES_INPUT1 = ((1024 * 16 + 0x1FF) >> 9) << 6 = 2048 bytes
Buffer Size = 2048 * 600 = 1,228,800 bytes
ダブルバッファ合計 = 2,457,600 bytes
```

### 2.3 ピクセルフォーマット

| レイヤー | フォーマット | ビット配置 | 用途 |
|---------|-------------|-----------|------|
| Layer1 | RGB565 | R:5, G:6, B:5 | カメラ画像 |
| Layer2 | ARGB4444 | A:4, R:4, G:4, B:4 | UI/オーバーレイ |

### 2.4 ダブルバッファリング

| 項目 | 状態/値 |
|------|---------|
| ダブルバッファ | **有効** (Layer2のみ) |
| バッファ0 | `fb_foreground[0]` |
| バッファ1 | `fb_foreground[1]` |
| 切り替えタイミング | VSYNCに同期 |
| 切り替え変数 | `s_drw_buf` (0/1トグル) |

**バッファ切り替えコード** (`display_thread_entry.c:262`):
```c
s_drw_buf = s_drw_buf ? 0 : 1;
```

---

## 3. 描画ライブラリ

### 3.1 使用ライブラリ

| ライブラリ | 使用状況 |
|-----------|----------|
| emWin | 不使用 |
| GUIX | 不使用 |
| LVGL | **部分的使用** (型定義のみ: lv_point_t, lv_indev_data_t) |
| Dave/2D (TES) | **使用** (メイン描画エンジン) |
| 独自実装 | **有** (フォントレンダリング) |

### 3.2 Dave/2D 主要API

| API | 機能 | 使用箇所 |
|-----|------|----------|
| `d2_opendevice()` | デバイスオープン | 初期化時 |
| `d2_inithw()` | ハードウェア初期化 | 初期化時 |
| `d2_newrenderbuffer()` | レンダーバッファ作成 | 初期化時 |
| `d2_selectrenderbuffer()` | レンダーバッファ選択 | 描画ループ毎 |
| `d2_framebuffer()` | フレームバッファ設定 | 描画ループ毎 |
| `d2_cliprect()` | クリッピング矩形設定 | 描画ループ毎 |
| `d2_setcolor()` | 描画色設定 | 各描画前 |
| `d2_setalpha()` | アルファ値設定 | 初期化時 |
| `d2_setblitsrc()` | ブリットソース設定 | 背景画像描画時 |
| `d2_blitcopy()` | 画像コピー | 背景/オーバーレイ描画 |
| `d2_renderbox()` | 矩形描画 | 塗りつぶし |
| `d2_renderline()` | 直線描画 | メニュー/閉じるアイコン |
| `d2_clear()` | 画面クリア | カメラデモ時 |
| `d2_executerenderbuffer()` | レンダーバッファ実行 | 描画完了時 |
| `d2_flushframe()` | フレームフラッシュ | 描画完了時 |

### 3.3 独自フォントライブラリ

| ファイル | 機能 |
|----------|------|
| `user_font_title_if.c` | タイトルフォント描画 (大文字/記号) |
| `user_font_body_if.c` | 本文フォント描画 (大文字/小文字/数字) |

**主要API**:
- `user_font_title_draw_line()` - タイトル行描画
- `user_font_body_draw_line()` - 本文行描画

---

## 4. 描画フロー

### 4.1 初期化フロー

```
システム起動
    │
    ▼
display_thread_entry() 開始
    │
    ├─→ バックライトOFF
    │
    ├─→ タッチスレッド初期化待ち
    │       while(false == ts_configured())
    │
    ├─→ フレームバッファクリア
    │       memset(fb_background, 0, ...)
    │       memset(user_clean_background, 0, ...)
    │
    ▼
initialise_primary_display()
    │
    ├─→ 画面サイズ取得 (1024x600)
    │
    ├─→ R_GLCDC_Open() - GLCDCドライバ初期化
    │
    └─→ R_GLCDC_Start() - GLCDC表示開始
    │
    ▼
Dave/2D初期化
    │
    ├─→ d2_opendevice(0)
    │
    ├─→ d2_inithw(0)
    │
    ├─→ d2_newrenderbuffer(20, 20)
    │
    ├─→ d2_selectrenderbuffer()
    │
    ├─→ d2_setblendmode(d2_bm_alpha, d2_bm_zero)
    │
    ├─→ d2_setalphamode(d2_am_constant)
    │
    └─→ d2_setalpha(0xff)
    │
    ▼
初期画面描画
    │
    ├─→ set_gs_page() - Getting Started画面設定
    │
    └─→ set_background() - 背景設定
    │
    ▼
メイン描画ループ開始
```

### 4.2 フレーム更新フロー

```
while(1)メインループ
    │
    ▼
graphics_wait_vsync()
    │   └─→ while(!g_vsync_flag); // VSYNC待機
    │
    ▼
d2_framebuffer() - 描画先設定
    │   └─→ graphics_get_draw_buffer() で現在の描画バッファ取得
    │
    ▼
d2_cliprect() - クリッピング設定 (0,0 - 1023,599)
    │
    ▼
set_background() - 背景状態更新
    │
    ▼
描画処理 (g_current_backgroud.update == false の場合)
    │
    ├─→ draw_background() - 背景画像描画
    │       └─→ d2_blitcopy() で1024x600画像コピー
    │
    ├─→ call_user_draw() - 画面別描画
    │       ├─→ Getting Started/KIS/EXT/NS: テキスト描画
    │       ├─→ LED Demo: menu_led_update_screen()
    │       └─→ LCD Demo: menu_lcd_update_screen()
    │
    ├─→ draw_menu_icon() - メニューアイコン描画
    │       └─→ d2_renderline() で3本線描画
    │
    ├─→ draw_exit_icon() - 閉じるアイコン描画
    │       └─→ d2_renderline() でX印描画
    │
    └─→ draw_overlay() - オーバーレイメニュー描画
            └─→ d2_blitcopy() でメニュー画像コピー
    │
    ▼
d2_executerenderbuffer() - レンダリング実行
    │
    ▼
d2_flushframe() - フレームフラッシュ
    │
    ▼
s_drw_buf トグル (0↔1)
    │
    ▼
R_GLCDC_BufferChange() - フレームバッファ切替
    │
    ├─→ Layer1: カメラバッファ更新
    │
    └─→ Layer2: graphics_get_back_buffer() で描画済みバッファを表示
    │
    ▼
vTaskDelay(10) - 10msウェイト
    │
    └─→ ループ先頭へ
```

---

## 5. レイヤー構成

### 5.1 GLCDCレイヤー設定

| レイヤー | 用途 | 解像度 | フォーマット | アドレス |
|---------|------|--------|-------------|----------|
| 背景 | 背景色 | - | - | - |
| Layer1 | カメラ画像 | 768x450 | RGB565 | `fb_background[0]` / VINバッファ |
| Layer2 | UI/オーバーレイ | 1024x600 | ARGB4444 | `fb_foreground[0/1]` |

### 5.2 レイヤー合成構造

```
┌─────────────────────────────────────────────────────────────┐
│                    表示出力 (1024 x 600)                     │
├─────────────────────────────────────────────────────────────┤
│  Layer2 (fb_foreground) - 1024x600, ARGB4444                │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ UI要素:                                                 ││
│  │ - メニューアイコン (左上)                               ││
│  │ - 閉じるアイコン (右上)                                 ││
│  │ - テキスト (Getting Started等)                         ││
│  │ - オーバーレイメニュー (中央)                           ││
│  │ ※ 透明部分(A=0)を通してLayer1が見える                  ││
│  └─────────────────────────────────────────────────────────┘│
│  ↓ アルファブレンド (ARGB4444のA値)                        │
├─────────────────────────────────────────────────────────────┤
│  Layer1 (fb_background/VINバッファ) - 768x450, RGB565      │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ コンテンツ:                                             ││
│  │ - カメラキャプチャ画像 (LCDデモ時)                      ││
│  │ - 背景画像 (Welcome/LED画面時)                         ││
│  └─────────────────────────────────────────────────────────┘│
│  ↓ 合成                                                     │
├─────────────────────────────────────────────────────────────┤
│  背景 (Background Color) - 黒 (0x000000)                   │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 カメラデモ時のレイヤー動作

```
カメラデモ (LCD_FULL_BG_CAMERA_BACKGROUND) 動作:

Layer1: VINバッファ (カメラ画像)
        │
        ├─→ VIN割込みでバッファ更新通知
        │       └─→ vin0_callback() → display_next_buffer_set()
        │
        └─→ R_GLCDC_BufferChange(LAYER_1, gp_camera_buffer)
                で表示バッファ切替

Layer2: 透明背景 + タイトルテキスト
        │
        ├─→ d2_clear(0x00000000) で透明クリア
        │
        ├─→ タイトル描画 ("Camera and Graphics Demonstration")
        │
        └─→ menu_lcd_update_screen() でFPS等表示
```

---

## 6. 描画システムブロック図

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           アプリケーション層                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐│
│  │ menu_main.c │ │ menu_led.c  │ │ menu_lcd.c  │ │ menu_kis/ext/ns.c      ││
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └───────────┬─────────────┘│
│         │               │               │                    │              │
└─────────┴───────────────┴───────────────┴────────────────────┴──────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         display_thread_entry.c                               │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ 描画制御:                                                              │ │
│  │ - graphics_wait_vsync() - VSYNC同期                                   │ │
│  │ - graphics_get_draw_buffer() - ダブルバッファ管理                     │ │
│  │ - dsp_set_background() - 画面切替                                     │ │
│  │ - call_user_draw() - 画面別描画呼び出し                               │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Dave/2D ライブラリ                              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │ 描画API                 │  │ ブリットAPI            │                   │
│  │ - d2_renderbox()        │  │ - d2_setblitsrc()      │                   │
│  │ - d2_renderline()       │  │ - d2_blitcopy()        │                   │
│  │ - d2_clear()            │  │                        │                   │
│  └─────────────────────────┘  └─────────────────────────┘                   │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │ レンダーバッファ管理     │  │ フレームバッファ設定    │                   │
│  │ - d2_newrenderbuffer()  │  │ - d2_framebuffer()     │                   │
│  │ - d2_selectrenderbuffer()│ │ - d2_executerenderbuffer()│                │
│  │ - d2_flushframe()       │  │                        │                   │
│  └─────────────────────────┘  └─────────────────────────┘                   │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FSP HAL (r_glcdc.c, r_drw.c)                        │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │ GLCDC API               │  │ DRW API                │                   │
│  │ - R_GLCDC_Open()        │  │ (Dave/2D内部使用)      │                   │
│  │ - R_GLCDC_Start()       │  │                        │                   │
│  │ - R_GLCDC_BufferChange()│  │                        │                   │
│  └─────────────────────────┘  └─────────────────────────┘                   │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ハードウェア                                    │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │ GLCDC                   │  │ DRW (2D Engine)        │                   │
│  │ - Layer1/Layer2合成     │  │ - ハードウェア描画     │                   │
│  │ - VSYNC生成             │  │ - ブリット処理         │                   │
│  │ - RGB出力               │  │                        │                   │
│  └───────────┬─────────────┘  └─────────────────────────┘                   │
│              │                                                              │
│              ▼                                                              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐                   │
│  │ 外部SDRAM               │  │ LCDパネル              │                   │
│  │ - フレームバッファ       │  │ - 1024x600            │                   │
│  │ - 画像リソース          │  │ - Parallel RGB        │                   │
│  └─────────────────────────┘  └─────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 画面一覧と描画内容

| 画面ID | 画面名 | Layer1内容 | Layer2内容 |
|--------|--------|-----------|-----------|
| 0 | Getting Started | 黒背景 | テキスト + メニュー/閉じるアイコン |
| 1 | Kit Information | 黒背景 | キット情報テキスト |
| 2 | User LED | LED背景画像 | LED制御UI |
| 3 | External Memory | 黒背景 | メモリ読み書きテキスト |
| 4 | Camera/LCD | カメラ画像 | タイトル + FPS表示 |
| 5 | Next Steps | 黒背景 | ガイダンステキスト |
| 6 | Welcome | ウェルカム画像 | メニューアイコンのみ |

---

## 8. パフォーマンス情報

### 8.1 フレームレート

- **目標フレームレート**: 60 Hz (VSYNCに同期)
- **実効フレームレート**: 約50-60 FPS
- **ループ遅延**: 10ms (`vTaskDelay(10)`)

### 8.2 描画最適化手法

1. **ダブルバッファリング**
   - Layer2のみダブルバッファ使用
   - VSYNCに同期してバッファ切替
   - ティアリング防止

2. **ハードウェアアクセラレーション**
   - Dave/2D (DRW) による描画処理
   - CPU負荷軽減

3. **レンダーバッファ**
   - コマンドリストによるバッチ描画
   - `d2_executerenderbuffer()` で一括実行

4. **クリッピング**
   - 描画領域をLCD範囲に制限
   - 無駄な描画を防止

5. **ブリットコピー**
   - 背景画像の高速コピー
   - `d2_blitcopy()` によるハードウェア転送

### 8.3 メモリ効率

- **RGB565フォーマット**: 16-bit/ピクセルでメモリ節約
- **ARGB4444フォーマット**: 透明度対応しつつ16-bit/ピクセル
- **SDRAMセクション配置**: 高速アクセス

---

## 9. 主要コード例

### 9.1 描画ループ初期化

```c
// display_thread_entry.c:193-208
gp_d2_handle = d2_opendevice(0);
d2_inithw(gp_d2_handle, 0);

renderbuffer = d2_newrenderbuffer(gp_d2_handle, 20, 20);
d2_selectrenderbuffer(gp_d2_handle, renderbuffer);

d2_setblendmode(gp_d2_handle, d2_bm_alpha, d2_bm_zero);
d2_setalphamode(gp_d2_handle, d2_am_constant);
d2_setalpha(gp_d2_handle, 0xff);
d2_setantialiasing(gp_d2_handle, 0);
d2_setlinecap(gp_d2_handle, d2_lc_butt);
d2_setlinejoin(gp_d2_handle, d2_lj_max);
```

### 9.2 フレーム描画

```c
// display_thread_entry.c:229-274
graphics_wait_vsync();

d2_selectrenderbuffer(gp_d2_handle, renderbuffer);
d2_framebuffer(gp_d2_handle, graphics_get_draw_buffer(),
               LCD_HPIX, LCD_HPIX, LCD_VPIX * 2, EP_SCREEN_MODE_TP);

d2_cliprect(gp_d2_handle, 0, 0, LCD_HPIX-1, LCD_VPIX-1);

// 描画処理...

d2_executerenderbuffer(gp_d2_handle, renderbuffer, 0);
d2_flushframe(gp_d2_handle);

s_drw_buf = s_drw_buf ? 0 : 1;

R_GLCDC_BufferChange(g_plcd_display.p_ctrl,
                     graphics_get_back_buffer(),
                     DISPLAY_FRAME_LAYER_2);
```

### 9.3 画像コピー (ブリット)

```c
// display_thread_entry.c:362-372
d2_setblitsrc(gp_d2_handle,
              (uint8_t *)&g_menu_overlay_image.pixel_data[0],
              g_menu_overlay_image.width,
              g_menu_overlay_image.width,
              g_menu_overlay_image.height,
              d2_mode_rgb565);

d2_blitcopy(gp_d2_handle,
            (d2_s32) g_menu_overlay_image.width,
            (d2_s32) g_menu_overlay_image.height,
            (d2_blitpos) 0, (d2_blitpos) 0,
            (d2_width) ((uint32_t)(g_menu_overlay_image.width) << 4),
            (d2_width) ((uint32_t)(g_menu_overlay_image.height) << 4),
            (d2_point) (128 << 4), (d2_point) (74 << 4),
            d2_tm_filter);
```

---

## 10. まとめ

### 10.1 システム特徴

1. **2レイヤー構成**: カメラ画像とUIを独立管理
2. **ダブルバッファリング**: ティアリングフリー表示
3. **ハードウェアアクセラレーション**: Dave/2D + DRWエンジン
4. **VSYNC同期**: 安定した60Hz表示
5. **SDRAM活用**: 大容量フレームバッファ

### 10.2 技術スタック

| 層 | コンポーネント |
|----|---------------|
| アプリケーション | menu_*.c, display_thread_entry.c |
| 描画ライブラリ | Dave/2D (TES), 独自フォントライブラリ |
| HAL | FSP r_glcdc.c, r_drw.c |
| ハードウェア | GLCDC, DRW, 外部SDRAM, LCDパネル |

### 10.3 フレームバッファ使用量

| バッファ | サイズ | 用途 |
|----------|--------|------|
| fb_background | 691 KB | Layer1 (カメラ/背景) |
| fb_foreground x2 | 2.3 MB | Layer2 ダブルバッファ |
| VINバッファ x3 | 2.0 MB | カメラキャプチャ |
| **合計** | **約5 MB** | SDRAM使用量 |
