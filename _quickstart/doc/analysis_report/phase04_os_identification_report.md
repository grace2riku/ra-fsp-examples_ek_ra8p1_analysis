# Phase 4: OS特定 - 解析レポート

## 解析日時
2026-01-24

## 概要

EK-RA8P1 クイックスタートプロジェクトで使用しているRTOSを特定し、タスク構成と同期機構を解析しました。
本プロジェクトはFreeRTOSをベースとしたマルチタスクアプリケーションです。

---

## 1. RTOS識別結果

### 1.1 使用RTOS

| 項目 | 値 |
|------|-----|
| **RTOS名** | FreeRTOS (Amazon FreeRTOS) |
| **統合方法** | Renesas FSP (Flexible Software Platform) |
| **設定ファイル** | `ra_cfg/aws/FreeRTOSConfig.h` |

### 1.2 識別根拠

- `ra_cfg/aws/FreeRTOSConfig.h` の存在
- `ra/aws/FreeRTOS/` ディレクトリにFreeRTOSソースコード
- タスク生成に `xTaskCreateStatic()` を使用

---

## 2. タスク構成

### 2.1 タスク一覧

| タスク名 | 優先度 | スタック(bytes) | スタック(words) | エントリ関数 | 役割 |
|----------|--------|-----------------|-----------------|--------------|------|
| **TP Thread** | 15 (最高) | 4,096 | 1,024 | `tp_thread_entry()` | タッチパネル入力処理 |
| **Blinky Thread** | 15 (最高) | 512 | 128 | `blinky_thread_entry()` | LED点滅（プレースホルダ） |
| **Board Monitor Thread** | 8 (中) | 1,024 | 256 | `board_mon_thread_entry()` | ボード状態監視、温度、LED制御 |
| **Camera Thread** | 3 (低-中) | 10,240 | 2,560 | `camera_thread_entry()` | OV5640カメラ映像処理 |
| **Display Thread** | 2 (低) | 2,048 | 512 | `display_thread_entry()` | グラフィックス描画、画面管理 |
| **Main Menu Thread** | 1 (最低) | 8,096 | 2,024 | `main_menu_thread_entry()` | メインメニュー、アプリロジック |

### 2.2 システムタスク（自動生成）

| タスク名 | スタック | 説明 |
|----------|----------|------|
| **Idle Task** | 128 words | アイドル時に実行 |
| **Timer Task** | 128 words | ソフトウェアタイマー管理 |

### 2.3 メモリ使用量

- **ユーザータスク合計**: 約29.5KB（スタック）
- **システムタスク合計**: 約1KB（スタック）
- **ヒープサイズ**: 512KB（`configTOTAL_HEAP_SIZE`）

### 2.4 タスク優先度階層図

```
優先度 15 (最高): TP Thread, Blinky Thread
    ↓              [タッチ入力は即座に応答]
優先度 8 (中):     Board Monitor Thread
    ↓              [ボード状態監視]
優先度 3 (低-中):  Timer Task, Camera Thread
    ↓              [カメラ映像処理]
優先度 2 (低):     Display Thread
    ↓              [画面描画]
優先度 1 (最低):   Main Menu Thread
    ↓              [メインロジック]
優先度 0:          Idle Task
```

---

## 3. 同期・通信機構

### 3.1 イベントグループ

| イベントグループ | 用途 | 使用タスク |
|------------------|------|------------|
| `g_update_console_event` | コンソール更新通知 | Board Monitor, ISR |
| `g_update_display_event` | ディスプレイ更新同期 | Display, Camera |
| `g_i2c_event_group` | I2C通信同期 | TP, Board Monitor |

#### イベントビット（g_update_console_event）

```c
STATUS_UPDATE_KIS_INFO     // Kit Information Screen更新
STATUS_UPDATE_TEMP_INFO    // 温度情報更新
STATUS_UPDATE_INTENSE_INFO // 輝度情報更新
STATUS_UPDATE_FREQ_INFO    // 周波数情報更新
```

### 3.2 セマフォ

#### バイナリセマフォ

| セマフォ | 用途 | 使用箇所 |
|----------|------|----------|
| `g_irq_binary_semaphore` | タッチパネル割込み通知 | TP Thread, IRQ19 |

#### カウンティングセマフォ

| セマフォ | 最大カウント | 初期値 | 用途 |
|----------|-------------|--------|------|
| `g_fsp_common_initialized_semaphore` | 256 | 1 | 共通初期化の同期 |

### 3.3 クリティカルセクション

```c
// タッチパネルデータの排他アクセス
taskENTER_CRITICAL();
// g_tp_copy_data の読み書き
taskEXIT_CRITICAL();
```

### 3.4 ISRからの同期

```c
// ボタン割込みからイベント通知
xEventGroupSetBitsFromISR(g_update_console_event, STATUS_UPDATE_FREQ_INFO, &xHigherPriorityTaskWoken);
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

### 3.5 タスク遅延

| タスク | 遅延時間 | 用途 |
|--------|----------|------|
| Blinky | 1000ms | LED点滅周期 |
| Camera | 100ms | フレーム処理間隔 |
| Main Menu | 100ms | メニュー更新間隔 |
| Board Monitor | 50ms | 状態監視間隔 |
| Display | 10ms | 描画更新間隔 |
| TP | 10ms | タッチ入力ポーリング |

---

## 4. FreeRTOS設定

### 4.1 基本設定

| 設定項目 | 値 | 説明 |
|----------|-----|------|
| `configTICK_RATE_HZ` | 1000 | システムティック周期（1ms） |
| `configMAX_PRIORITIES` | 16 | 最大優先度数（0-15） |
| `configMINIMAL_STACK_SIZE` | 128 | 最小スタックサイズ（ワード） |
| `configTOTAL_HEAP_SIZE` | 0x80000 | ヒープサイズ（512KB） |
| `configMAX_TASK_NAME_LEN` | 32 | タスク名最大長 |

### 4.2 スケジューラ設定

| 設定項目 | 値 | 説明 |
|----------|-----|------|
| `configUSE_PREEMPTION` | 1 | プリエンプティブスケジューリング有効 |
| `configUSE_TIME_SLICING` | 0 | タイムスライシング無効 |

### 4.3 メモリ管理

| 設定項目 | 値 | 説明 |
|----------|-----|------|
| `configSUPPORT_STATIC_ALLOCATION` | 1 | 静的メモリ割当て有効 |
| `configSUPPORT_DYNAMIC_ALLOCATION` | 1 | 動的メモリ割当て有効 |
| `configAPPLICATION_ALLOCATED_HEAP` | 0 | カーネルがヒープ管理 |

### 4.4 同期機構設定

| 設定項目 | 値 | 説明 |
|----------|-----|------|
| `configUSE_MUTEXES` | 0 | ミューテックス無効 |
| `configUSE_RECURSIVE_MUTEXES` | 0 | 再帰ミューテックス無効 |
| `configUSE_COUNTING_SEMAPHORES` | 1 | カウンティングセマフォ有効 |
| `configUSE_TASK_NOTIFICATIONS` | 1 | タスク通知有効 |

### 4.5 タイマー設定

| 設定項目 | 値 | 説明 |
|----------|-----|------|
| `configUSE_TIMERS` | 1 | ソフトウェアタイマー有効 |
| `configTIMER_TASK_PRIORITY` | 3 | タイマータスク優先度 |
| `configTIMER_QUEUE_LENGTH` | 10 | タイマーキュー長 |
| `configTIMER_TASK_STACK_DEPTH` | 128 | タイマースタック（ワード） |

### 4.6 フック関数

| 設定項目 | 値 | 説明 |
|----------|-----|------|
| `configUSE_IDLE_HOOK` | 1 | アイドルフック有効 |
| `configCHECK_FOR_STACK_OVERFLOW` | 0 | スタックオーバーフローチェック無効 |

### 4.7 割込み設定

| 設定項目 | 説明 |
|----------|------|
| `configMAX_SYSCALL_INTERRUPT_PRIORITY` | FreeRTOS API呼び出し可能な最高割込み優先度 |
| `configLIBRARY_LOWEST_INTERRUPT_PRIORITY` | 最低割込み優先度 |

---

## 5. OS初期化フロー

### 5.1 初期化シーケンス

```
1. main() 開始
    │
    ├── グローバル変数初期化
    │   ├── g_fsp_common_thread_count = 0
    │   └── g_fsp_common_initialized = false
    │
    ├── カウンティングセマフォ作成
    │   └── g_fsp_common_initialized_semaphore
    │
    ├── タスク作成（順次）
    │   ├── display_thread_create()
    │   ├── camera_thread_create()
    │   ├── main_menu_thread_create()
    │   ├── board_mon_thread_create()
    │   ├── tp_thread_create()
    │   └── blinky_thread_create()
    │
    └── vTaskStartScheduler()
        └── スケジューラ開始
```

### 5.2 共通リソース初期化（g_common_init）

```
g_common_init()
    ├── g_update_console_event 作成
    ├── g_update_display_event 作成
    ├── g_irq_binary_semaphore 作成
    └── g_i2c_event_group 作成
```

### 5.3 最初のスレッド初期化

```
rtos_startup_common_init()
    │
    ├── セマフォ取得
    │
    ├── 最初のスレッドの場合:
    │   ├── g_hal_init() 呼び出し
    │   │   └── ボード固有ハードウェア初期化
    │   └── セマフォ解放
    │
    └── 他のスレッド:
        └── セマフォ待機後に開始
```

---

## 6. タスク間相互作用図

```
┌─────────────────────────────────────────────────────────────────┐
│                      割込みハンドラ                              │
│  IRQ19(タッチ) → g_irq_binary_semaphore                         │
│  IRQ12(SW1)   → g_update_console_event (FREQ_INFO)              │
│  IRQ13(SW2)   → g_update_console_event (INTENSE_INFO)           │
│  VIN callback → display_next_buffer_set()                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       タスク層                                   │
│                                                                  │
│  ┌────────────┐    g_tp_copy_data    ┌────────────┐             │
│  │  TP Thread │ ──────────────────→ │  Display   │             │
│  │ (優先度15) │   (critical section) │  Thread    │             │
│  └────────────┘                      │ (優先度2)  │             │
│        ↑                             └────────────┘             │
│  g_irq_binary_semaphore                    ↑                    │
│                                            │                    │
│  ┌────────────┐                      ┌────────────┐             │
│  │   Camera   │  ────────────────→  │  Display   │             │
│  │   Thread   │   VIN callback       │  Buffer    │             │
│  │ (優先度3)  │                      └────────────┘             │
│  └────────────┘                                                 │
│                                                                  │
│  ┌────────────┐  g_update_console_event  ┌────────────┐         │
│  │   Board    │ ←──────────────────────│    ISR     │         │
│  │  Monitor   │                         │ (SW1/SW2)  │         │
│  │ (優先度8)  │                         └────────────┘         │
│  └────────────┘                                                 │
│                                                                  │
│  ┌────────────┐                                                 │
│  │ Main Menu  │   メインアプリケーションロジック                 │
│  │ (優先度1)  │                                                 │
│  └────────────┘                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 設計上の特徴

### 7.1 静的メモリ割当て

- すべてのタスク、セマフォ、イベントグループで静的割当てを使用
- メモリフラグメンテーションを防止
- 決定論的な動作を保証

### 7.2 優先度設計

- **タッチ入力（最高優先度）**: 即座に応答が必要
- **ボード監視（中優先度）**: 定期的な状態確認
- **カメラ/ディスプレイ（低優先度）**: バックグラウンド処理
- **メインメニュー（最低優先度）**: 他のタスクを妨げない

### 7.3 ミューテックス未使用

- イベントグループとセマフォで同期
- よりシンプルで決定論的な動作
- 優先度逆転の問題を回避

---

## 発見事項

### 1. 設計の良い点

- 静的メモリ割当てによる安定性
- 適切な優先度階層設計
- ISRからの効率的な同期機構

### 2. 注意点

- スタックオーバーフローチェックが無効（`configCHECK_FOR_STACK_OVERFLOW=0`）
- 本番環境では有効化を検討
- カメラスレッドの大きなスタック（10KB）は画像処理に必要

### 3. 拡張可能性

- 16優先度レベルのうち一部のみ使用
- ヒープに512KBの余裕あり
- 追加タスクの実装が可能

---

## 関連ファイル

- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/ra_cfg/aws/FreeRTOSConfig.h` - FreeRTOS設定
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/ra_gen/main.c` - OS初期化
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/ra_gen/*_thread.c` - タスク生成
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/ra_gen/common_data.c` - 同期機構定義
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/src/*_thread_entry.c` - タスクエントリ

---

## 次のPhaseへの申し送り

### Phase 5（コード構造解析）への情報

- FreeRTOSベースのマルチタスク構成
- 6つのユーザータスクが協調動作
- 主要な同期機構: イベントグループ、バイナリセマフォ
- タッチ→ディスプレイ、カメラ→ディスプレイのデータフロー
- ISRからのイベント通知パターン
