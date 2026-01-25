# Phase 12: 画面遷移の仕組み解析レポート

## 1. 概要

本レポートは、EK-RA8P1評価ボードのQuick Startデモにおける画面遷移システムを詳細に解析した結果をまとめたものである。画面管理方式、遷移トリガー、背景切替処理、オーバーレイメニューの仕組みを明らかにする。

## 2. 画面管理方式

### 2.1 画面（Screen）の定義方法

本システムでは画面を「背景ID」として管理する。構造体ではなくマクロ定義による単純なID管理方式を採用。

**画面ID定義**: `common_init.h:68-77`

```c
/* Available option for LCD screens in the demo  */
#define LCD_FULL_BG_DEFAULT             (99)
#define LCD_FULL_BG_POPUP_MENU          (42)
#define LCD_FULL_BG_GETTING_STARTED     (9)
#define LCD_FULL_BG_KIT_INFORMATION     (1)
#define LCD_FULL_BG_USER_LED            (2)
#define LCD_FULL_BG_EXTERNAL_MEMORY     (3)
#define LCD_FULL_BG_CAMERA_BACKGROUND   (4)
#define LCD_FULL_BG_NEXT_STEPS          (5)
#define LCD_FULL_BG_WELCOME             (127)
```

### 2.2 画面状態管理構造体

**背景状態構造体**: `common_utils.h:121-128`

```c
typedef struct {
    int8_t active_demo;               // 現在アクティブなデモID
    bool_t update;                    // 背景更新フラグ
    bool_t menu_icon_active;          // メニューアイコン表示有無
    bool_t exit_icon_active;          // 終了アイコン表示有無
    st_rgb565_text_block_page_t *p_text_block;  // テキストページへのポインタ
    uint8_t * p_bg_src;               // 背景画像データへのポインタ
} st_drv_background_t;
```

**グローバルインスタンス**: `display_thread_entry.c:75`
```c
st_drv_background_t g_current_backgroud = {LCD_FULL_BG_DEFAULT, true, true, true, NULL, NULL};
```

### 2.3 画面の登録・管理

| 管理項目 | 変数名 | 説明 |
|----------|--------|------|
| 現在の画面 | `g_current_backgroud.active_demo` | 表示中のデモID |
| 選択中メニュー | `g_selected_menu` | メニュー番号 (1-5) |
| 次の遷移先 | `g_next_menu_selection` | ポップアップ選択結果 |
| 更新フラグ | `g_current_backgroud.update` | 背景再描画要求 |

### 2.4 メニュー関数テーブル

**メニュー関数テーブル定義**: `menu_main.c:87-95`

```c
static st_menu_fn_tbl_t s_menu_items[] =
{
    {"Kit Information",                 kis_display_menu},     // '1'
    {"User LED Control Page",           led_display_menu},     // '2'
    {"External Memory Read Write Demo", ext_display_menu},     // '3'
    {"Camera and LCD Demonstration",    lcd_display_menu},     // '4'
    {"Next Steps",                      ns_display_menu},      // '5'
    {"", NULL }
};
```

**メニュー関数テーブル構造体**: `common_init.h:138-142`
```c
typedef struct menu_fn_tbl
{
    char_t * p_name;                      /*<! Name of Test */
    test_fn ( * p_func)(void);            /*<! Pointer to Test Function */
} st_menu_fn_tbl_t;
```

## 3. 画面遷移のトリガー

### 3.1 タッチイベントによる遷移

**タッチ領域と遷移先の対応**: `tp_thread_entry.c:142-286`

| タッチ領域 | 操作 | 遷移先 | 選択コード |
|-----------|------|--------|-----------|
| hb_menu | メニューアイコンタップ | ポップアップメニュー表示 | LCD_FULL_BG_POPUP_MENU |
| hb_cancel | Xアイコンタップ | Welcome画面 | ' ' (空白) |
| menu_cancel | ポップアップ内キャンセル | 現在画面維持 | 'c' |
| kis_demo | KIS選択 | Kit Information画面 | '1' |
| gs_demo | GS選択 | Getting Started画面 | '9' |
| led_demo | LED選択 | LED Control画面 | '2' |
| lcd_demo | LCD選択 | Camera/LCD画面 | '4' |
| ns_demo | NS選択 | Next Steps画面 | '5' |

### 3.2 キーボード（シリアルコンソール）による遷移

**キー入力処理**: `menu_main.c:289-298`

```c
/* Cast, as compiler will assume calc is int */
c = (int8_t) (c - '0');
g_selected_menu = c;

if ((c > 0) && (c <= g_menu_limit))
{
     g_overlay_selected = false;
     s_menu_items[c - 1].p_func ();
```

| キー | 遷移先 |
|------|--------|
| '1' | Kit Information画面 |
| '2' | LED Control画面 |
| '3' | External Memory画面 |
| '4' | Camera/LCD画面 |
| '5' | Next Steps画面 |
| ' ' (スペース) | メインメニューに戻る |

### 3.3 タイマーによる自動遷移

本システムでは明示的なタイマー遷移は実装されていない。

- スプラッシュ画面: なし（直接Getting Started画面を表示）
- スクリーンセーバー: なし
- 自動戻り: なし

### 3.4 外部イベントによる遷移

| イベント | 処理 |
|----------|------|
| カメラ検出 | `check_ov5640_camera_connected()` でメッセージ変更 |
| タッチコントローラ未検出 | 限定メニュー表示 (`s_menu_items_nd[]`) |

## 4. 画面切り替え処理

### 4.1 切り替えフロー

```
画面遷移要求（タッチ or キー入力）
        │
        ▼
process_menu_detection() で遷移先決定
        │
        ├─→ g_next_menu_selection に遷移先コードを設定
        │
        └─→ g_overlay_selected = false（ポップアップ閉じる）
        │
        ▼
現在デモ関数からリターン（ループ脱出）
        │
        ├─→ MENU_EXIT_CRTL または遷移コード検出
        │
        └─→ main_display_menu() へ戻る
        │
        ▼
main_display_menu() で新デモ関数を呼び出し
        │
        ├─→ s_menu_items[c - 1].p_func()
        │
        └─→ 新デモ内で dsp_set_background() 呼び出し
        │
        ▼
dsp_set_background() で g_current_backgroud を更新
        │
        ├─→ p_bg_src: 背景画像ポインタ設定
        │
        ├─→ p_text_block: テキストページ設定
        │
        ├─→ menu/exit_icon_active: アイコン表示設定
        │
        └─→ update = true: 更新フラグセット
        │
        ▼
display_thread_entry() のメインループで描画更新
        │
        ├─→ set_background() で更新フラグ処理
        │
        ├─→ draw_background() で背景描画
        │
        ├─→ call_user_draw() でデモ固有描画
        │
        └─→ draw_overlay() でポップアップ描画
```

### 4.2 遷移API

**背景設定関数**: `display_thread_entry.c:670-733`

```c
void dsp_set_background(uint32_t choice)
{
    g_current_backgroud.p_text_block = NULL;
    set_selected_menu_demo(choice);

    switch (choice)
    {
        case LCD_FULL_BG_USER_LED:
            g_current_backgroud.p_bg_src = (uint8_t *)&g_ref_001_user_led_background.pixel_data[0];
            g_current_backgroud.menu_icon_active = true;
            g_current_backgroud.exit_icon_active = true;
            break;

        case LCD_FULL_BG_CAMERA_BACKGROUND:
            g_current_backgroud.p_bg_src = (uint8_t *)&user_clean_background.pixel_data[0];
            g_current_backgroud.menu_icon_active = true;
            g_current_backgroud.exit_icon_active = true;
            break;

        // ... 他の画面も同様
    }

    g_current_backgroud.update = true;
}
```

**Getting Started画面設定**: `display_thread_entry.c:562-566`
```c
void set_gs_page(void)
{
    g_show_getting_started_page = true;
    dsp_set_background(LCD_FULL_BG_GETTING_STARTED);
}
```

## 5. 画面スタック/履歴

### 5.1 「戻る」機能の実装

| 項目 | 状態 |
|------|------|
| 履歴管理 | **無** |
| 最大履歴数 | N/A |
| 実装方式 | N/A |

本システムでは画面スタックは実装されていない。「戻る」操作は常に固定の遷移先へ移動する。

### 5.2 遷移パターン

| 現在画面 | 戻る操作 | 遷移先 |
|----------|----------|--------|
| 各デモ画面 | Xアイコンタップ | Welcome画面 |
| 各デモ画面 | スペースキー | メインメニュー表示 |
| ポップアップメニュー | キャンセル | 現在画面維持 |
| Getting Started | Xアイコン | Welcome画面 |

## 6. 画面ごとのリソース

### 6.1 リソース管理方式

| 方式 | 説明 |
|------|------|
| **静的配置** | 全リソースをコンパイル時に配置 |
| 動的ロード | 未使用 |

全ての画像・フォントリソースはコンパイル時にROMまたはOSPI Flashに配置。
実行時はSDRAMにコピーして使用。

### 6.2 画面別リソース一覧

| 画面ID | 画面名 | 背景画像 | テキストページ | フォント |
|--------|--------|----------|----------------|----------|
| 127 | Welcome | g_ref_000_welcome_screen | なし | なし |
| 9 | Getting Started | user_clean_background | gs_page | Title/Body |
| 1 | Kit Information | user_clean_background | g_kis_page | Title/Body |
| 2 | LED Control | g_ref_001_user_led_background | なし | なし |
| 3 | External Memory | user_clean_background | g_ext_page | Title/Body |
| 4 | Camera/LCD | user_clean_background | なし | Title |
| 5 | Next Steps | user_clean_background | g_ns_page | Title/Body |
| 42 | ポップアップ | g_menu_overlay_image | なし | なし |

### 6.3 背景画像配置

**SDRAMバッファ定義**: `display_thread_entry.c:63-64`
```c
uint8_t sdram_buffer[DISPLAY_BUFFER_STRIDE_BYTES_INPUT1 * DISPLAY_VSIZE_INPUT1 * 2]
    BSP_ALIGN_VARIABLE(64) BSP_PLACE_IN_SECTION(".sdram") = "";
st_full_image_rgb565_t user_clean_background
    __attribute__ ((section (".sdram"))) = {1024, 600, 2, {""}};
```

**OSPI Flashからコピー**: `main_menu_thread_entry.c:98-99`
```c
memcpy(&g_sdram_buffer_welcome[0], &g_ref_000_welcome_screen.pixel_data[0], sizeof(g_sdram_buffer_welcome));
memcpy(&g_sdram_buffer_led[0], &g_ref_001_user_led_background.pixel_data[0], sizeof(g_sdram_buffer_led));
```

## 7. オーバーレイ（ポップアップメニュー）

### 7.1 オーバーレイ表示フラグ

**フラグ定義**: `common_utils.c:57`
```c
bool_t g_overlay_selected = false;
```

### 7.2 オーバーレイ描画処理

**描画関数**: `display_thread_entry.c:358-377`

```c
static void draw_overlay(void)
{
    if (true == g_overlay_selected)
    {
        d2_setblitsrc(gp_d2_handle, (uint8_t *)&g_menu_overlay_image.pixel_data[0],
                      g_menu_overlay_image.width,
                      g_menu_overlay_image.width, g_menu_overlay_image.height, d2_mode_rgb565);

        d2_blitcopy(gp_d2_handle,
                (d2_s32) g_menu_overlay_image.width, (d2_s32) g_menu_overlay_image.height,
                (d2_blitpos) 0, (d2_blitpos) 0,
                (d2_width) ((uint32_t)(g_menu_overlay_image.width) << 4),
                (d2_width) ((uint32_t)(g_menu_overlay_image.height) << 4),
                (d2_point) (128 << 4), (d2_point) (74 << 4),  // 表示位置
                d2_tm_filter);

        graphics_reset_blit_buffer();
    }
}
```

### 7.3 オーバーレイ表示位置

| 項目 | 値 |
|------|-----|
| X座標 | 128ピクセル |
| Y座標 | 74ピクセル |
| 画像サイズ | menu_overlay.cで定義 |

## 8. 遷移アニメーション

| アニメーション | 実装状況 |
|----------------|----------|
| フェード | **なし** |
| スライド | **なし** |
| なし（即時切替） | **○** |

本システムでは遷移アニメーションは実装されておらず、即時切り替えとなる。

## 9. 画面遷移状態遷移図

```
                                    ┌─────────────────────────────────────────────────┐
                                    │                                                 │
                                    ▼                                                 │
           ┌──────────────────────────────────────────────────────┐                   │
           │                    Welcome 画面                       │                   │
           │                (LCD_FULL_BG_WELCOME)                 │                   │
           └─────────────────────────┬────────────────────────────┘                   │
                                     │                                                 │
                              メニューアイコン                                          │
                               タッチ                                                  │
                                     │                                                 │
           ┌─────────────────────────▼────────────────────────────┐                   │
           │               Getting Started 画面                    │◄─────────────────┐│
           │            (LCD_FULL_BG_GETTING_STARTED)              │                  ││
           └─────────────────────────┬────────────────────────────┘                  ││
                                     │                                                ││
                              メニューアイコン                                         ││
                               タッチ                                                 ││
                                     │                                                ││
           ┌─────────────────────────▼────────────────────────────┐                  ││
           │              ポップアップメニュー                       │                  ││
           │             (g_overlay_selected=true)                │                  ││
           └────────┬────────┬────────┬────────┬────────┬─────────┘                  ││
                    │        │        │        │        │                            ││
            '1'選択  │  '2'選択│  '3'選択│  '4'選択│  '5'選択                          ││
                    │        │        │        │        │                            ││
     ┌──────────────▼─┐  ┌───▼────┐  ┌▼───────┐  ┌▼───────┐  ┌────▼─────┐           ││
     │ Kit Information│  │ LED    │  │External│  │Camera/ │  │Next Steps│           ││
     │   (ID=1)       │  │Control │  │Memory  │  │LCD     │  │  (ID=5)  │           ││
     │                │  │(ID=2)  │  │(ID=3)  │  │(ID=4)  │  │          │           ││
     └───────┬────────┘  └────┬───┘  └───┬────┘  └────┬───┘  └────┬─────┘           ││
             │                │          │            │           │                  ││
             │                │          │            │           │                  ││
             └────────────────┴──────────┴────────────┴───────────┘                  ││
                                         │                                            ││
                                    Xアイコン                                          ││
                                     タッチ                                           ││
                                         │                                            ││
                                         └────────────────────────────────────────────┘│
                                                                                       │
                              (ポップアップからGS選択)                                   │
                                         └─────────────────────────────────────────────┘
```

## 10. 画面一覧表

| 画面ID | 画面名 | マクロ名 | 遷移元 | 遷移先 | 主な機能 |
|--------|--------|----------|--------|--------|----------|
| 127 | Welcome | LCD_FULL_BG_WELCOME | 起動時、各デモ終了時 | Getting Started、ポップアップ | ウェルカム画像表示 |
| 9 | Getting Started | LCD_FULL_BG_GETTING_STARTED | Welcome、ポップアップ | ポップアップ | 操作説明テキスト表示 |
| 42 | ポップアップ | LCD_FULL_BG_POPUP_MENU | 任意画面 | 選択したデモ | メニュー選択UI |
| 1 | Kit Information | LCD_FULL_BG_KIT_INFORMATION | ポップアップ | ポップアップ、Welcome | MCU情報表示 |
| 2 | LED Control | LCD_FULL_BG_USER_LED | ポップアップ | ポップアップ、Welcome | LED制御デモ |
| 3 | External Memory | LCD_FULL_BG_EXTERNAL_MEMORY | ポップアップ | ポップアップ、Welcome | OSPI/SDRAMデモ |
| 4 | Camera/LCD | LCD_FULL_BG_CAMERA_BACKGROUND | ポップアップ | ポップアップ、Welcome | カメラ映像・タッチデモ |
| 5 | Next Steps | LCD_FULL_BG_NEXT_STEPS | ポップアップ | ポップアップ、Welcome | 次のステップ情報 |

## 11. デモ固有描画処理

**call_user_draw()関数**: `display_thread_entry.c:293-346`

```c
void call_user_draw(void)
{
    switch (g_current_backgroud.active_demo)
    {
        case 0: // Getting Started
        case LCD_FULL_BG_GETTING_STARTED:
        case LCD_FULL_BG_KIT_INFORMATION:
        case LCD_FULL_BG_EXTERNAL_MEMORY:
        case LCD_FULL_BG_NEXT_STEPS:
            // テキストページ描画
            if(g_current_backgroud.p_text_block != NULL)
            {
                set_display_image_text(g_current_backgroud.p_text_block);
            }
            break;

        case 2: // LED demo
            menu_led_update_screen(gp_d2_handle);
            break;

        case 4: // LCD demo
            menu_lcd_update_screen(gp_d2_handle);
            break;

        default:
            // 何も描画しない
    }
}
```

## 12. カスタマイズガイド

### 12.1 新しい画面の追加方法

#### 手順1: 画面IDの定義

`common_init.h` に新しいマクロを追加:

```c
#define LCD_FULL_BG_MY_SCREEN    (6)
```

#### 手順2: メニュー関数の作成

新しいファイル `menu_myscreen.c` を作成:

```c
#include "common_init.h"
#include "display_thread_entry.h"
#include "tp_thread_entry.h"

test_fn myscreen_display_menu(void)
{
    int8_t c = -1;
    lv_indev_data_t data = {};
    int8_t on_screen_selection = -1;

    // 背景設定
    dsp_set_background(LCD_FULL_BG_MY_SCREEN);

    start_key_check();

    while (CONNECTION_ABORT_CRTL != c)
    {
        // タッチ入力処理
        touchpad_get_copy(&data);
        on_screen_selection = process_menu_detection(&data, false, '6');

        // 画面固有の処理
        // ...

        // 終了条件
        if (key_pressed() || on_screen_selection != -1)
        {
            // 遷移処理
        }

        vTaskDelay(10);
    }

    return (0);
}
```

#### 手順3: メニューテーブルへの登録

`menu_main.c` の `s_menu_items[]` に追加:

```c
static st_menu_fn_tbl_t s_menu_items[] =
{
    {"Kit Information",                 kis_display_menu},
    {"User LED Control Page",           led_display_menu},
    {"External Memory Read Write Demo", ext_display_menu},
    {"Camera and LCD Demonstration",    lcd_display_menu},
    {"Next Steps",                      ns_display_menu},
    {"My New Screen",                   myscreen_display_menu},  // 追加
    {"", NULL }
};
```

#### 手順4: dsp_set_background()への追加

`display_thread_entry.c` の switch文に追加:

```c
case LCD_FULL_BG_MY_SCREEN:
    g_current_backgroud.p_bg_src = (uint8_t *)&user_clean_background.pixel_data[0];
    g_current_backgroud.p_text_block = &my_screen_page;  // テキストがある場合
    g_current_backgroud.menu_icon_active = true;
    g_current_backgroud.exit_icon_active = true;
    break;
```

#### 手順5: ポップアップメニューのヒットボックス追加（オプション）

`tp_thread_entry.c` に新しい領域を追加:

```c
st_bounding_box_t my_demo = {{ x1, y1}, { x2, y2}};
```

### 12.2 遷移条件の変更方法

#### 特定画面への直接遷移

```c
// 任意の場所から呼び出し可能
g_next_menu_selection = '6';  // 新画面のコード
```

#### 自動遷移の追加

```c
// タイマーコールバックで遷移
void auto_transition_callback(TimerHandle_t xTimer)
{
    g_next_menu_selection = LCD_FULL_BG_WELCOME;
}
```

### 12.3 アニメーションの追加方法（高度なカスタマイズ）

#### フェードイン効果の例

```c
void fade_in_transition(void)
{
    for (uint8_t alpha = 0; alpha <= 255; alpha += 15)
    {
        d2_setalpha(gp_d2_handle, alpha);
        draw_background();
        d2_executerenderbuffer(gp_d2_handle, renderbuffer, 0);
        d2_flushframe(gp_d2_handle);
        graphics_wait_vsync();
    }
}
```

#### スライド効果の例

```c
void slide_transition(int direction)
{
    for (int offset = 1024; offset >= 0; offset -= 64)
    {
        d2_clear(gp_d2_handle, 0x00000000);

        // 古い画面を左にスライドアウト
        d2_blitcopy(gp_d2_handle, 1024, 600, 0, 0,
                    1024 << 4, 600 << 4,
                    (-offset) << 4, 0 << 4, d2_tm_filter);

        // 新しい画面を右からスライドイン
        d2_blitcopy(gp_d2_handle, 1024, 600, 0, 0,
                    1024 << 4, 600 << 4,
                    (1024 - offset) << 4, 0 << 4, d2_tm_filter);

        d2_executerenderbuffer(gp_d2_handle, renderbuffer, 0);
        d2_flushframe(gp_d2_handle);
        graphics_wait_vsync();
    }
}
```

## 13. 関連ファイル一覧

| ファイル | 役割 |
|----------|------|
| `src/menu_main.c` | メインメニューコントローラ |
| `src/menu_main.h` | メニュー関数宣言 |
| `src/display_thread_entry.c` | 表示スレッド、背景切替、描画 |
| `src/display_thread_entry.h` | 表示関連宣言 |
| `src/common_init.h` | 画面ID定義、構造体定義 |
| `src/common_utils.h` | 共通型定義 |
| `src/tp_thread_entry.c` | タッチ領域判定、遷移トリガー |
| `src/menu_kis.c` | Kit Information画面 |
| `src/menu_led.c` | LED Control画面 |
| `src/menu_ext.c` | External Memory画面 |
| `src/menu_lcd.c` | Camera/LCD画面 |
| `src/menu_ns.c` | Next Steps画面 |
| `src/images/overlays/menu_overlay.c` | ポップアップメニュー画像 |

## 14. まとめ

EK-RA8P1のQuick Startデモにおける画面遷移システムは以下の特徴を持つ:

1. **シンプルな状態管理**: マクロ定義による画面ID管理
2. **関数ポインタテーブル**: メニュー選択と関数の対応
3. **グローバル状態**: `g_current_backgroud`構造体で現在状態を管理
4. **オーバーレイメニュー**: `g_overlay_selected`フラグでポップアップ制御
5. **即時遷移**: アニメーションなしの直接切り替え
6. **履歴なし**: 常に固定の遷移先へ移動

シンプルな設計により理解しやすく、新しい画面の追加も容易な構造となっている。より複雑なUI要件がある場合は、画面スタックやアニメーション機能の追加が必要となる。
