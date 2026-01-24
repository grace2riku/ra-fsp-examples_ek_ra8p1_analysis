# Phase 3: コンパイル・リンクオプション詳細 - 解析レポート

## 解析日時
2026-01-24

## 概要

EK-RA8P1 クイックスタートプロジェクトのコンパイラ、アセンブラ、リンカのオプションを詳細に解析しました。
本プロジェクトは GCC ARM Embedded 13.2.1 ツールチェーンを使用し、Cortex-M85向けに最適化されています。

### プロジェクト基本情報

| 項目 | 値 |
|------|-----|
| **ツールチェーン** | GCC ARM Embedded 13.2.1 (arm-13-7) |
| **ターゲットCPU** | Cortex-M85 (`-mcpu=cortex-m85+nopacbti`) |
| **浮動小数点ABI** | ハードウェア (`-mfloat-abi=hard`) |
| **TrustZone** | 無効（`-mcmse`なし） |
| **C言語標準** | C99 (`-std=c99`) |
| **C++言語標準** | C++11 (`-std=c++11`) |

---

## 1. コンパイラオプション（C/C++）

### 1.1 Debug構成とRelease構成の比較

| オプション | Debug | Release | 備考 |
|-----------|-------|---------|------|
| 最適化レベル | `-O0` | `-O2` | Debug:なし、Release:速度優先 |
| デバッグ情報 | `-g` | `-g` | 両方でシンボル生成 |
| `_RA_DEBUG_BUILD_` | 定義あり | 定義なし | デバッグビルド識別 |
| 命令セット | デフォルト | Thumb | Releaseでコードサイズ削減 |

### 1.2 最適化オプション

```
-O0                    # Debug: 最適化なし（デバッグ容易性優先）
-O2                    # Release: 速度最適化
-fmessage-length=0     # エラーメッセージの行折り返しなし
-ffunction-sections    # 関数ごとにセクション分割（未使用関数削除用）
-fdata-sections        # データごとにセクション分割（未使用データ削除用）
-fno-strict-aliasing   # 厳密なエイリアシング規則を無効化
-fsigned-char          # charを符号付きとして扱う
```

### 1.3 警告オプション

#### 有効な警告

```
-Wall                  # 一般的な警告をすべて有効
-Wextra                # 追加の警告を有効
-Wunused               # 未使用変数・関数の警告
-Wuninitialized        # 未初期化変数の警告
-Wmissing-declarations # グローバル関数の宣言漏れ警告
-Wconversion           # 暗黙の型変換警告
-Wpointer-arith        # ポインタ演算の警告
-Wshadow               # 変数シャドウイングの警告
-Wlogical-op           # 疑わしい論理演算の警告
-Waggregate-return     # 構造体返却の警告
-Wfloat-equal          # 浮動小数点比較の警告
```

#### 抑制している警告

```
-Wno-stringop-overflow   # 文字列操作オーバーフロー警告を抑制
-Wno-format-truncation   # フォーマット切り詰め警告を抑制
```

### 1.4 CPU・FPU指定

```
-mcpu=cortex-m85+nopacbti  # Cortex-M85（PAC/BTI無効）
-mfloat-abi=hard           # ハードウェア浮動小数点ABI
```

**注意**: `+nopacbti`はPointer Authentication Code (PAC) と Branch Target Identification (BTI) を無効化しています。これはセキュリティ機能ですが、互換性のために無効化されています。

### 1.5 その他の重要オプション

```
-flax-vector-conversions   # ベクトル型の暗黙変換を許可（SIMD最適化用）
--param=min-pagesize=0     # 最小ページサイズを0に設定（組み込み向け）
```

### 1.6 プリプロセッサ定義

| マクロ | 値 | 説明 |
|--------|-----|------|
| `_RENESAS_RA_` | (定義のみ) | Renesas RAプラットフォーム識別 |
| `_RA_ORDINAL` | `1` | RAデバイス序数 |
| `_RA_CORE` | `CPU0` | 使用コア（Cortex-M85） |
| `_RA_DEBUG_BUILD_` | (定義のみ) | デバッグビルド識別（Debug構成のみ） |

### 1.7 インクルードパス

```
${ProjName}/src                              # アプリケーションソース
${ProjName}/ra/fsp/inc                       # FSP共通ヘッダ
${ProjName}/ra/fsp/inc/api                   # FSP APIヘッダ
${ProjName}/ra/fsp/inc/instances             # FSPインスタンスヘッダ
${ProjName}/ra/fsp/src/rm_freertos_port      # FreeRTOSポート
${ProjName}/ra/aws/FreeRTOS/FreeRTOS/Source/include  # FreeRTOSヘッダ
${ProjName}/ra/arm/CMSIS_6/CMSIS/Core/Include        # CMSIS-Coreヘッダ
${ProjName}/ra_gen                           # FSP生成コード
${ProjName}/ra_cfg/fsp_cfg/bsp               # BSP設定
${ProjName}/ra_cfg/fsp_cfg                   # FSP設定
${ProjName}/ra_cfg/aws                       # AWS/FreeRTOS設定
${ProjName}/ra/tes/dave2d/inc                # Dave2Dグラフィックス
```

---

## 2. アセンブラオプション

### 2.1 基本設定

| 項目 | 設定 |
|------|------|
| **ツール** | `arm-none-eabi-as` |
| **プリプロセッサ** | 有効（`.s`ファイルでCプリプロセッサ使用） |
| **CPU指定** | `-mcpu=cortex-m85+nopacbti`（コンパイラと同一） |

### 2.2 プリプロセッサ定義

コンパイラと同じ定義が適用されます：
- `_RENESAS_RA_`
- `_RA_ORDINAL=1`
- `_RA_CORE=CPU0`

---

## 3. リンカオプション

### 3.1 リンカスクリプト構成

```
script/fsp.ld                    # メインリンカスクリプト
  ├── INCLUDE memory_regions.ld  # メモリ領域定義（Debug/に自動生成）
  └── INCLUDE fsp_gen.ld         # FSPセクション定義（Debug/に自動生成）
```

### 3.2 リンカオプション一覧

```
-T "fsp.ld"                       # リンカスクリプト指定
-Xlinker --gc-sections            # 未使用セクション削除
--specs=nano.specs                # newlib-nano使用（サイズ削減）
-L"${ProjName}/script"            # ライブラリ検索パス
-Wl,-Map,"*.map"                  # Mapファイル生成
-Wl,--start-group ... --end-group # 循環依存解決
```

### 3.3 リンクするライブラリ

```
libgcc.a   # GCCランタイムライブラリ
libc.a     # C標準ライブラリ（newlib-nano）
libm.a     # 数学ライブラリ
```

### 3.4 メモリ配置（memory_regions.ld）

| 領域 | 開始アドレス | サイズ | 用途 |
|------|-------------|--------|------|
| **ITCM** | 0x00000000 | 128KB | 命令TCM（高速実行） |
| **DTCM** | 0x20000000 | 128KB | データTCM（高速アクセス） |
| **RAM** | 0x22000000 | 1,872KB | メインRAM |
| **FLASH** | 0x02000000 | 1MB | 内蔵フラッシュ |
| **DATA_FLASH** | 0x27000000 | 0 | データフラッシュ（未使用） |
| **SDRAM** | 0x68000000 | 128MB | 外部SDRAM |
| **OSPI1_CS0** | 0x70000000 | 128MB | OSPI1チップセレクト0 |
| **OSPI1_CS1** | 0x78000000 | 128MB | OSPI1チップセレクト1 |
| **OSPI0_CS0** | 0x80000000 | 256MB | OSPI0チップセレクト0 |
| **OSPI0_CS1** | 0x90000000 | 256MB | OSPI0チップセレクト1 |

### 3.5 エントリポイント

```
ENTRY(Reset_Handler)  # リンカスクリプトで定義
```

---

## 4. ポストビルド処理

### 4.1 HEXファイル生成

```bash
arm-none-eabi-objcopy -O ihex "quickstart_ek_ra8p1_ep.elf" "quickstart_ek_ra8p1_ep.hex"
```

### 4.2 サイズ表示

```bash
arm-none-eabi-size --format=berkeley "quickstart_ek_ra8p1_ep.elf"
```

### 4.3 逆アセンブルリスト生成

```bash
arm-none-eabi-objdump -S -x -C -l -w "quickstart_ek_ra8p1_ep.elf"
```

オプション説明：
- `-S`: ソースコード表示
- `-x`: 全ヘッダ表示
- `-C`: C++名前のデマングル
- `-l`: 行番号表示
- `-w`: 幅広出力

---

## 5. 特殊な設定

### 5.1 TrustZone設定

| 項目 | 状態 |
|------|------|
| TrustZone (`-mcmse`) | **無効** |
| セキュア/非セキュア分離 | なし |
| ビルド構成名 | `Debug_Flat`（Flat = TrustZone無効） |

### 5.2 フォルダ固有の警告抑制

以下のフォルダでは全警告が抑制されています（`-w`フラグ）：

| フォルダ | 理由 |
|---------|------|
| `ra/tes/` | Dave2Dグラフィックスライブラリ（サードパーティ） |
| `ra/aws/` | FreeRTOS（サードパーティ） |
| `ra/arm/` | CMSIS（サードパーティ） |

### 5.3 スタック・ヒープサイズ

リンカスクリプト（`fsp_gen.ld`）で定義：
- FSPコンフィグレータで設定可能
- `configuration.xml`で変更

---

## 6. ビルドツール一覧

| ツール | コマンド | 用途 |
|--------|---------|------|
| Cコンパイラ | `arm-none-eabi-gcc` | Cソースのコンパイル |
| C++コンパイラ | `arm-none-eabi-g++` | C++ソースのコンパイル |
| アセンブラ | `arm-none-eabi-as` | アセンブリソースのアセンブル |
| リンカ | `arm-none-eabi-ld` | オブジェクトファイルのリンク |
| アーカイバ | `arm-none-eabi-ar` | 静的ライブラリ作成 |
| HEX変換 | `arm-none-eabi-objcopy` | ELF→HEX変換 |
| 逆アセンブル | `arm-none-eabi-objdump` | 逆アセンブルリスト生成 |
| サイズ表示 | `arm-none-eabi-size` | セクションサイズ表示 |

---

## 7. 実際のビルドコマンド例

### 7.1 コンパイルコマンド（Debug構成）

```bash
arm-none-eabi-gcc \
  -mfloat-abi=hard \
  -mcpu=cortex-m85+nopacbti \
  -O0 \
  -fmessage-length=0 \
  -fsigned-char \
  -ffunction-sections \
  -fdata-sections \
  -fno-strict-aliasing \
  -Wunused -Wuninitialized -Wall -Wextra \
  -Wmissing-declarations -Wconversion -Wpointer-arith \
  -Wshadow -Wlogical-op -Waggregate-return -Wfloat-equal \
  -g \
  -D_RENESAS_RA_ -D_RA_ORDINAL=1 -D_RA_DEBUG_BUILD_ -D_RA_CORE=CPU0 \
  -I"./src" -I"./ra/fsp/inc" -I"./ra/fsp/inc/api" \
  -std=c99 \
  -Wno-stringop-overflow -Wno-format-truncation \
  -flax-vector-conversions --param=min-pagesize=0 \
  -MMD -MP \
  -c -o output.o input.c
```

### 7.2 リンクコマンド

```bash
arm-none-eabi-gcc \
  -mfloat-abi=hard \
  -mcpu=cortex-m85+nopacbti \
  -O0 -g \
  -T "fsp.ld" \
  -Xlinker --gc-sections \
  -L "script" \
  -Wl,-Map,"quickstart_ek_ra8p1_ep.map" \
  --specs=nano.specs \
  -o "quickstart_ek_ra8p1_ep.elf" \
  -Wl,--start-group *.o -lgcc -lc -lm -Wl,--end-group
```

---

## 8. 最適化に関する推奨事項

### 8.1 デバッグビルドでの注意点

- `-O0`により変数がレジスタに最適化されないため、デバッグが容易
- ただしコードサイズが大きくなり、実行速度も遅い
- 本番環境での動作確認には不十分な場合がある

### 8.2 リリースビルドでの推奨設定

| オプション | 推奨値 | 理由 |
|-----------|-------|------|
| 最適化 | `-O2` または `-Os` | 速度またはサイズ優先 |
| LTO | `-flto` | リンク時最適化（さらなるサイズ削減） |
| デバッグ情報 | `-g1` または なし | 最小限のデバッグ情報 |

### 8.3 コードサイズ削減のヒント

1. **`-Os`オプション**: サイズ最適化
2. **`--gc-sections`**: 未使用コード削除（既に有効）
3. **`nano.specs`**: 軽量Cライブラリ使用（既に有効）
4. **LTO**: `-flto`でリンク時最適化

---

## 発見事項

### 1. セキュリティ機能の無効化

- PAC/BTI（Pointer Authentication / Branch Target Identification）が無効化されている
- TrustZoneが無効（Flat構成）
- 本番環境ではセキュリティ要件に応じて有効化を検討

### 2. サードパーティコードの警告抑制

- `ra/tes/`, `ra/aws/`, `ra/arm/`で全警告を抑制
- これらのフォルダのコードを変更する場合は注意

### 3. 特殊なコンパイラオプション

- `-flax-vector-conversions`: SIMD最適化のために必要
- `--param=min-pagesize=0`: 組み込み環境向け最適化
- これらはFSP/Dave2Dとの互換性のために必要

---

## 関連ファイル

- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/.cproject` - ビルド設定（188KB）
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/script/fsp.ld` - リンカスクリプト
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/Debug/memory_regions.ld` - メモリ領域定義
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/Debug/fsp_gen.ld` - FSPセクション定義
- `_quickstart/quickstart_ek_ra8p1_ep/e2studio/Debug/makefile` - 生成Makefile

---

## 次のPhaseへの申し送り

### Phase 4（コード構造解析）への情報

- エントリポイント: `Reset_Handler` → `SystemInit` → `main`
- FreeRTOS使用のため、`main()`後はスケジューラが起動
- Cortex-M85向けのARM CMSIS v6を使用
- Dave2D（D/AVE 2D）グラフィックスエンジンを使用

### コンパイル環境の注意点

- GCC 13.2.1を使用（新しいバージョンでは警告が増える可能性）
- `-fno-strict-aliasing`が必要（FSPとの互換性）
- サードパーティライブラリは警告抑制されているため変更時は注意
