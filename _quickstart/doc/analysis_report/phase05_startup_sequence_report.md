# Phase 5: 起動シーケンス解析 - 解析レポート

## 解析日時
2026-01-24

## 概要

EK-RA8P1 クイックスタートプロジェクトのリセットベクタからmain関数までの起動シーケンスを詳細に解析しました。
Cortex-M85コアの高度な機能（I-Cache、D-Cache、TCM、FPU）を活用した初期化処理を行っています。

---

## 1. リセットベクタとベクタテーブル

### 1.1 ベクタテーブル定義

**ファイル**: `ra/fsp/src/bsp/cmsis/Device/RENESAS/Source/startup.c`

```c
BSP_DONT_REMOVE const fsp_vector_t g_vector_table[BSP_CORTEX_VECTOR_TABLE_ENTRIES]
    BSP_PLACE_IN_SECTION(BSP_SECTION_FIXED_VECTORS) =
{
    (fsp_vector_t)(&g_main_stack[0] + BSP_CFG_STACK_MAIN_BYTES),  // [0] MSP初期値
    Reset_Handler,                                                  // [1] リセットハンドラ
    NMI_Handler,                                                    // [2] NMI
    HardFault_Handler,                                              // [3] ハードフォルト
    MemManage_Handler,                                              // [4] メモリ管理
    BusFault_Handler,                                               // [5] バスフォルト
    UsageFault_Handler,                                             // [6] 使用フォルト
    SecureFault_Handler,                                            // [7] セキュアフォルト
    0, 0, 0,                                                        // [8-10] 予約
    SVC_Handler,                                                    // [11] SVCコール
    DebugMon_Handler,                                               // [12] デバッグモニタ
    0,                                                              // [13] 予約
    PendSV_Handler,                                                 // [14] PendSV
    SysTick_Handler,                                                // [15] SysTick
};
```

### 1.2 ベクタテーブル配置

| 項目 | 値 |
|------|-----|
| **配置セクション** | `.fixed_vectors` |
| **配置アドレス** | 0x02000000 (内蔵フラッシュ先頭) |
| **エントリ数** | 16 (Cortex-M標準例外) |

---

## 2. スタックポインタの初期化

### 2.1 メインスタック (MSP)

```c
// スタック領域定義
BSP_DONT_REMOVE static uint8_t g_main_stack[BSP_CFG_STACK_MAIN_BYTES + BSP_TZ_STACK_SEAL_SIZE]
    BSP_ALIGN_VARIABLE(BSP_STACK_ALIGNMENT)
    BSP_PLACE_IN_SECTION(BSP_SECTION_STACK);

// スタックシール値（TrustZone用）
#define BSP_TZ_STACK_SEAL_VALUE    (0xFEF5EDA5U)
```

### 2.2 スタック構成

| 項目 | 値 |
|------|-----|
| **MSP初期値** | `&g_main_stack[0] + BSP_CFG_STACK_MAIN_BYTES` |
| **アライメント** | 8バイト境界 |
| **スタックシール** | TrustZone有効時に0xFEF5EDA5を設定 |
| **PSP** | FreeRTOSがタスクごとに管理 |

---

## 3. Reset_Handler（エントリポイント）

**ファイル**: `ra/fsp/src/bsp/cmsis/Device/RENESAS/Source/startup.c`

```c
void Reset_Handler (void)
{
    /* Initialize system using BSP. */
    SystemInit();

    /* Call user application. */
    main();

    while (1)
    {
        /* Infinite Loop. */
    }
}
```

**特徴**:
- MSPはハードウェアがベクタテーブルから自動設定
- すべての初期化はSystemInit()で実行
- main()からリターンした場合は無限ループ

---

## 4. SystemInit（システム初期化）

**ファイル**: `ra/fsp/src/bsp/cmsis/Device/RENESAS/Source/system.c`

### 4.1 Phase 1: CPU・FPU設定

```c
// I-Cache、分岐予測、LOB拡張の有効化
SCB->CCR = 0x000E0201;
__DSB();
__ISB();

// FPU有効化（CP10, CP11）
SCB->CPACR = (CP_MASK);
```

| 設定項目 | 説明 |
|----------|------|
| I-Cache | 有効 |
| 分岐予測 | 有効 |
| LOB拡張 | 有効（ループ最適化） |
| D-Cache | ライトスルーモード |
| FPU | ハードウェアFPU有効 |

### 4.2 Phase 2: VTORとスタックシール

```c
// ベクタテーブルオフセット設定
SCB->VTOR = (uint32_t) &__VECTOR_TABLE;

// スタックシール（TrustZone）
*p_main_stack_top = BSP_TZ_STACK_SEAL_VALUE;
```

### 4.3 Phase 3: クロック初期化

```c
// リセット時ウォームスタートコールバック
R_BSP_WarmStart(BSP_WARM_START_RESET);

// クロック初期化
bsp_clock_init();

// クロック初期化後コールバック
R_BSP_WarmStart(BSP_WARM_START_POST_CLOCK);
```

#### クロック初期化順序（bsp_clock_init）

1. CGC/LPM保護レジスタのアンロック
2. フラッシュキャッシュ設定
3. メインオシレータ起動・安定化待ち
4. サブクロック(SOSC)起動
5. HOCO起動・FLL設定
6. MOCO/LOCO起動
7. PLL2/PLL3設定・ロック待ち
8. 動作モード設定（高速/中速/低速）

### 4.4 Phase 4: スタック監視・TCM初期化

```c
// スタックオーバーフロー検出NMI有効化
R_MPU_SPMON->SP0CTL = 0x8000; // SPMON有効

// TCMゼロ初期化（ECC用）
memset_64(ITCM_START, 0, ITCM_SIZE);
memset_64(DTCM_START, 0, DTCM_SIZE);
```

### 4.5 Phase 5: ランタイム初期化

```c
// 内部メモリ初期化（.bss/.data）
SystemRuntimeInit(0);

// システムクロック変数更新
SystemCoreClockUpdate();

// PFSレジスタ設定有効化
R_PMISC->PWPR = 0x40;  // PFSWE有効
```

#### .bssゼロクリアと.dataコピー

リンカ生成の初期化テーブル（`g_init_info`）を使用：

```c
typedef struct st_bsp_init_info {
    uint32_t zero_count;                         // BSSセクション数
    bsp_init_zero_info_t const *const p_zero_list;
    uint32_t copy_count;                         // DATAセクション数
    bsp_init_copy_info_t const *const p_copy_list;
} bsp_init_info_t;
```

### 4.6 Phase 6: キャッシュ・電源ドメイン

```c
// MPU設定（キャッシュ領域）
bsp_init_mpu();

// D-Cache有効化
SCB_EnableDCache();

// グラフィックス電源ドメインON
// NPU電源ドメインON（使用時）
```

### 4.7 Phase 7: 最終初期化

```c
// C言語ランタイム完全初期化後
R_BSP_WarmStart(BSP_WARM_START_POST_C);

// 外部メモリ初期化（SDRAM, OSPI）
SystemRuntimeInit(1);

// C++静的コンストラクタ
for (p = __init_array_start; p < __init_array_end; p++)
    (*p)();

// ELCイベント設定
bsp_irq_cfg();

// ボード初期化
bsp_init(NULL);
```

---

## 5. main関数

**ファイル**: `ra_gen/main.c`

```c
int main(void)
{
    g_fsp_common_thread_count = 0;
    g_fsp_common_initialized = false;

    /* RTOS同期用セマフォ作成 */
    g_fsp_common_initialized_semaphore =
        xSemaphoreCreateCountingStatic(256, 1, ...);

    /* タスク作成 */
    display_thread_create();
    camera_thread_create();
    main_menu_thread_create();
    board_mon_thread_create();
    tp_thread_create();
    blinky_thread_create();

    /* FreeRTOSスケジューラ起動 */
    vTaskStartScheduler();

    return 0;
}
```

---

## 6. 呼び出しフロー図

```
[リセット]
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ Reset_Handler                                                │
│ (MSPはベクタテーブルから自動設定済み)                        │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ SystemInit()                                                 │
├─────────────────────────────────────────────────────────────┤
│  ├─→ I-Cache、分岐予測、LOB拡張有効化 (SCB->CCR)            │
│  │                                                          │
│  ├─→ FPU有効化 (SCB->CPACR)                                 │
│  │                                                          │
│  ├─→ スタックシール設定 (TrustZone)                         │
│  │                                                          │
│  ├─→ VTOR設定 (ベクタテーブルオフセット)                    │
│  │                                                          │
│  ├─→ R_BSP_WarmStart(BSP_WARM_START_RESET)                  │
│  │                                                          │
│  ├─→ bsp_clock_init()                                       │
│  │       ├─→ CGC/LPMアンロック                              │
│  │       ├─→ メインオシレータ起動                           │
│  │       ├─→ HOCO/MOCO/LOCO起動                             │
│  │       ├─→ PLL2/PLL3設定                                  │
│  │       └─→ 動作モード設定                                 │
│  │                                                          │
│  ├─→ R_BSP_WarmStart(BSP_WARM_START_POST_CLOCK)             │
│  │                                                          │
│  ├─→ LVOCR設定 (I/O電圧)                                    │
│  │                                                          │
│  ├─→ スタック監視設定 (NMI有効)                             │
│  │                                                          │
│  ├─→ ITCM/DTCMゼロ初期化 (ECC)                              │
│  │                                                          │
│  ├─→ SystemRuntimeInit(0) - 内部メモリ                      │
│  │       ├─→ .bssゼロクリア                                 │
│  │       └─→ .dataコピー (ROM→RAM)                          │
│  │                                                          │
│  ├─→ SystemCoreClockUpdate()                                │
│  │                                                          │
│  ├─→ RTC初期化                                              │
│  │                                                          │
│  ├─→ PFSレジスタ設定有効化                                  │
│  │                                                          │
│  ├─→ MPU設定 (キャッシュ領域)                               │
│  │                                                          │
│  ├─→ D-Cache有効化                                          │
│  │                                                          │
│  ├─→ グラフィックス/NPU電源ドメインON                       │
│  │                                                          │
│  ├─→ R_BSP_WarmStart(BSP_WARM_START_POST_C)                 │
│  │                                                          │
│  ├─→ SystemRuntimeInit(1) - 外部メモリ (SDRAM, OSPI)        │
│  │                                                          │
│  ├─→ TLS初期化                                              │
│  │                                                          │
│  ├─→ I-Cache無効化                                          │
│  │                                                          │
│  ├─→ C++静的コンストラクタ実行 (__init_array)               │
│  │                                                          │
│  ├─→ bsp_irq_cfg() - ELCイベント設定                        │
│  │                                                          │
│  └─→ bsp_init(NULL) - ボード初期化                          │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ main()                                                       │
├─────────────────────────────────────────────────────────────┤
│  ├─→ グローバル変数初期化                                   │
│  │                                                          │
│  ├─→ RTOS同期セマフォ作成                                   │
│  │                                                          │
│  ├─→ タスク作成                                             │
│  │       ├─→ display_thread_create()                        │
│  │       ├─→ camera_thread_create()                         │
│  │       ├─→ main_menu_thread_create()                      │
│  │       ├─→ board_mon_thread_create()                      │
│  │       ├─→ tp_thread_create()                             │
│  │       └─→ blinky_thread_create()                         │
│  │                                                          │
│  └─→ vTaskStartScheduler()                                  │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ FreeRTOSカーネル動作開始                                     │
├─────────────────────────────────────────────────────────────┤
│  ├─→ 最初のタスク: g_hal_init() (FSPモジュール初期化)       │
│  │                                                          │
│  ├─→ 他のタスク: セマフォ待機                               │
│  │                                                          │
│  └─→ スケジューラがタスクを解放                             │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
[アプリケーション実行中]
```

---

## 7. 各関数の役割

| 関数名 | ファイル | 役割 |
|--------|----------|------|
| `Reset_Handler` | `startup.c` | エントリポイント、SystemInit呼び出し |
| `SystemInit` | `system.c` | 全システム初期化のオーケストレーション |
| `bsp_clock_init` | `bsp_clocks.c` | クロックシステム初期化 |
| `SystemRuntimeInit` | `system.c` | .bss/.data初期化 |
| `SystemCoreClockUpdate` | `system.c` | SystemCoreClock変数更新 |
| `R_BSP_WarmStart` | ユーザー定義 | 起動段階別コールバック |
| `bsp_init_mpu` | `system.c` | MPU設定（キャッシュ領域） |
| `bsp_irq_cfg` | `bsp_irq.c` | ELC割込みイベント設定 |
| `bsp_init` | `board_init.c` | ボード固有初期化 |
| `main` | `main.c` | タスク作成、RTOS起動 |

---

## 8. R_BSP_WarmStartコールバック

### 8.1 コールバックタイミング

| タイミング | 状態 | 使用可能な機能 |
|------------|------|----------------|
| `BSP_WARM_START_RESET` | クロック未設定 | 最小限のレジスタアクセス |
| `BSP_WARM_START_POST_CLOCK` | クロック設定完了 | クロック依存処理可能 |
| `BSP_WARM_START_POST_C` | Cランタイム完全初期化 | すべての機能使用可能 |

### 8.2 実装例（hal_entry.c）

```c
void R_BSP_WarmStart(bsp_warm_start_event_t event)
{
    if (BSP_WARM_START_RESET == event)
    {
        // データフラッシュ読み取り有効化
    }

    if (BSP_WARM_START_POST_C == event)
    {
        // I/Oポート設定
        R_IOPORT_Open(&g_ioport_ctrl, g_ioport.p_cfg);

        // SDRAM初期化（使用時）
        #if (1 == USE_SDRAM)
        bsp_sdram_init(&g_sdram_cfg);
        #endif
    }
}
```

---

## 9. メモリ初期化構造

### 9.1 リンカ生成初期化テーブル

**ファイル**: `Debug/bsp_linker_info.h`

```c
// 初期化対象メモリ領域
- DTCM (Data Tightly Coupled Memory) - 64ビットアライン
- ITCM (Instruction TCM) - 64ビットアライン
- 内蔵RAM
- SDRAM (外部)
- OSPI0 CS0/CS1 (外部)
- OSPI1 CS0/CS1 (外部)
- 非キャッシュ領域
```

### 9.2 初期化順序

1. **SystemRuntimeInit(0)**: 内部メモリ（DTCM, ITCM, RAM）
2. **SystemRuntimeInit(1)**: 外部メモリ（SDRAM, OSPI）

---

## 発見事項

### 1. Cortex-M85固有の最適化

- I-Cache、D-Cache、分岐予測が有効
- LOB（Low Overhead Branch）拡張でループ最適化
- TCMにECC初期化（ゼロクリア必須）

### 2. 多段階ウォームスタート

- 3つのコールバックポイントで柔軟な初期化
- データフラッシュ、I/O、SDRAMの適切なタイミングでの初期化

### 3. メモリ初期化の自動化

- リンカ生成テーブルによる.bss/.data初期化
- 内部/外部メモリの分離処理

### 4. RTOS統合

- タスク作成後にスケジューラ起動
- 最初のタスクがg_hal_init()でFSPモジュール初期化

---

## 関連ファイル

| ファイル | 役割 |
|----------|------|
| `ra/fsp/src/bsp/cmsis/Device/RENESAS/Source/startup.c` | スタートアップコード |
| `ra/fsp/src/bsp/cmsis/Device/RENESAS/Source/system.c` | システム初期化 |
| `ra/fsp/src/bsp/mcu/all/bsp_clocks.c` | クロック初期化 |
| `ra_gen/vector_data.c` | ベクタテーブル（生成） |
| `Debug/bsp_linker_info.h` | リンカ初期化テーブル（生成） |
| `ra_gen/main.c` | メインエントリ（生成） |
| `src/hal_entry.c` | WarmStartコールバック |

---

## 次のPhaseへの申し送り

### Phase 6（周辺機能解析）への情報

- クロック初期化完了後に周辺機能が使用可能
- g_hal_init()でFSPモジュールが初期化される
- I/OポートはR_BSP_WarmStart(POST_C)で設定
- SDRAMは外部メモリ初期化前に設定が必要
