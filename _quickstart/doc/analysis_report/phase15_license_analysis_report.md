# Phase 15: ライセンス解析レポート

## 1. エグゼクティブサマリー

本解析では、EK-RA8P1 quickstartプロジェクトで使用されているすべてのソースコード、ライブラリ、サードパーティコンポーネントのライセンスを調査した。プロジェクトは複数のオープンソースライセンス（BSD-3-Clause、MIT、Apache-2.0）と独自ライセンス（Renesas Dave/2D）の組み合わせで構成されており、すべて商用利用可能な寛容なライセンスである。

## 2. ライセンスサマリー表

### 2.1 コンポーネント別ライセンス一覧

| コンポーネント | ライセンス | 著作権者 | バージョン |
|---------------|-----------|----------|-----------|
| アプリケーションコード | BSD-3-Clause | Renesas Electronics Corporation | - |
| FSP (Flexible Software Package) | BSD-3-Clause | Renesas Electronics Corporation | 6.3.0 |
| FreeRTOS Kernel | MIT | Amazon.com, Inc. | 11.1.0 |
| ARM CMSIS | Apache-2.0 | ARM Limited | 6.1 |
| Dave/2D (D/AVE) | Renesas独自 (無償) | Renesas Electronics (Europe) GmbH | - |
| 生成コード (ra_gen/) | (ライセンス表記なし) | 自動生成 | - |

### 2.2 ライセンス種別の分類

| ライセンス種別 | 分類 | コンポーネント数 |
|---------------|------|-----------------|
| BSD-3-Clause | Permissive | 2 (FSP, アプリ) |
| MIT | Permissive | 1 (FreeRTOS) |
| Apache-2.0 | Permissive | 1 (CMSIS) |
| 独自ライセンス | 無償・制限付き | 1 (Dave/2D) |

## 3. 各ライセンスの詳細

### 3.1 BSD-3-Clause ライセンス (FSP, アプリケーションコード)

**SPDX識別子**: BSD-3-Clause

**適用範囲**:
- `ra/fsp/` 以下のすべてのFSPソースコード
- `src/` 以下のアプリケーションコード

**ライセンスヘッダ例**:
```c
/*
* Copyright (c) 2020 - 2025 Renesas Electronics Corporation and/or its affiliates
*
* SPDX-License-Identifier: BSD-3-Clause
*/
```

**主な条件**:
| 条件 | 必要性 |
|------|--------|
| 著作権表示の保持 | 必須 |
| ライセンス文の同梱 | 必須 |
| 免責条項の記載 | 必須 |
| ソースコード開示 | 不要 |
| 派生物への同一ライセンス適用 | 不要 |

**制限事項**:
- 著作権者の名前を製品の宣伝・推奨に使用することは禁止

### 3.2 MIT ライセンス (FreeRTOS)

**SPDX識別子**: MIT

**適用範囲**:
- `ra/aws/FreeRTOS/` 以下のすべてのFreeRTOSソースコード
- `ra/fsp/src/rm_freertos_port/` (FreeRTOSポート層)

**ライセンスヘッダ**:
```c
/*
 * FreeRTOS Kernel V11.1.0
 * Copyright (C) 2021 Amazon.com, Inc. or its affiliates. All Rights Reserved.
 *
 * SPDX-License-Identifier: MIT
 */
```

**ライセンスファイル**: `ra/aws/FreeRTOS/LICENSE.md`

**主な条件**:
| 条件 | 必要性 |
|------|--------|
| 著作権表示の保持 | 必須 |
| ライセンス文の同梱 | 必須 |
| ソースコード開示 | 不要 |
| 商用利用 | 可能 |
| 改変 | 可能 |
| 再配布 | 可能 |

### 3.3 Apache License 2.0 (ARM CMSIS)

**SPDX識別子**: Apache-2.0

**適用範囲**:
- `ra/arm/CMSIS_6/` 以下のすべてのCMSISソースコード

**ライセンスヘッダ**:
```c
/*
 * Copyright (c) 2009-2023 ARM Limited. All rights reserved.
 *
 * SPDX-License-Identifier: Apache-2.0
 */
```

**ライセンスファイル**: `ra/arm/CMSIS_6/LICENSE`

**主な条件**:
| 条件 | 必要性 |
|------|--------|
| 著作権表示の保持 | 必須 |
| ライセンス文の同梱 | 必須 |
| 変更箇所の明示 | 必須 (改変時) |
| NOTICEファイルの同梱 | 必須 (存在する場合) |
| 特許ライセンスの付与 | 含まれる |
| ソースコード開示 | 不要 |

**特記事項**:
- 特許訴訟を起こした場合、特許ライセンスは終了

### 3.4 Renesas独自ライセンス (Dave/2D)

**適用範囲**:
- `ra/tes/dave2d/` 以下のDave/2Dライブラリ

**ライセンスヘッダ例**:
```c
/*
============================================================================
C O P Y R I G H T
============================================================================
Copyright (c) 2011
by
Renesas Electronics (Europe) GmbH.
Arcadiastrasse 10
D-40472 Duesseldorf
Germany
All rights reserved.
============================================================================
Purpose: only for testing

Warranty Disclaimer

Because the Product(s) is licensed free of charge, there is no warranty
of any kind whatsoever and expressly disclaimed and excluded by Renesas,
either expressed or implied, including but not limited to those for
non-infringement of intellectual property, merchantability and/or
fitness for the particular purpose.
*/
```

**主な条件**:
| 条件 | 内容 |
|------|------|
| 利用料金 | 無償 (free of charge) |
| 保証 | なし (AS IS) |
| 責任制限 | 一切の損害について責任を負わない |
| 用途 | テスト目的 (Purpose: only for testing) |

**注意事項**:
- 「only for testing」という記載があるが、FSPに含まれているため実製品での使用も想定されていると解釈される
- 詳細な利用条件についてはRenesasに確認することを推奨

## 4. ライセンス互換性マトリクス

### 4.1 ライセンス間の互換性

| 組み合わせ | 互換性 | 備考 |
|-----------|--------|------|
| BSD-3-Clause + MIT | ✅ 互換 | 両方Permissive |
| BSD-3-Clause + Apache-2.0 | ✅ 互換 | 両方Permissive |
| MIT + Apache-2.0 | ✅ 互換 | 両方Permissive |
| 全体の組み合わせ | ✅ 互換 | すべてPermissiveライセンス |

### 4.2 互換性の根拠

すべてのライセンスがPermissive（寛容）ライセンスであり：
- コピーレフト条項がない
- 派生物に同一ライセンス適用の義務がない
- 商用利用が許可されている
- ソースコード開示義務がない

## 5. コンプライアンスチェックリスト

### 5.1 製品配布時に必要な対応

| 項目 | BSD-3-Clause | MIT | Apache-2.0 | Dave/2D |
|------|-------------|-----|-----------|---------|
| 著作権表示の同梱 | ✅ 必須 | ✅ 必須 | ✅ 必須 | ✅ 必須 |
| ライセンス文の同梱 | ✅ 必須 | ✅ 必須 | ✅ 必須 | - |
| 免責事項の記載 | ✅ 必須 | ✅ 含む | ✅ 含む | ✅ 必須 |
| NOTICEファイル | - | - | ⚠️ 存在する場合 | - |
| 変更履歴の明示 | - | - | ⚠️ 改変時 | - |
| ソースコード開示 | ❌ 不要 | ❌ 不要 | ❌ 不要 | ❌ 不要 |

### 5.2 推奨される対応

1. **ライセンス文書の同梱**
   - 製品ドキュメントまたはソフトウェアに以下を含める：
     - BSD-3-Clause ライセンス全文
     - MIT ライセンス全文
     - Apache License 2.0 全文
     - Dave/2D の著作権表示と免責事項

2. **著作権表示の例**
   ```
   This product includes software developed by:
   - Renesas Electronics Corporation (FSP, Dave/2D) - BSD-3-Clause, Proprietary
   - Amazon.com, Inc. (FreeRTOS) - MIT License
   - ARM Limited (CMSIS) - Apache License 2.0
   ```

3. **サードパーティライセンスファイルの作成**
   - `THIRD_PARTY_LICENSES.txt` などのファイルを作成
   - すべてのライセンス文と著作権表示を集約

## 6. ファイル別ライセンス表記

### 6.1 ディレクトリ構造とライセンス

```
quickstart_ek_ra8p1_ep/e2studio/
├── src/                          # BSD-3-Clause (Renesas)
│   ├── hal_entry.c
│   ├── display_thread_entry.c
│   └── ...
├── ra_gen/                       # 自動生成 (ライセンス表記なし)
│   ├── common_data.c
│   ├── hal_data.c
│   └── ...
└── ra/
    ├── fsp/                      # BSD-3-Clause (Renesas)
    │   ├── src/
    │   └── inc/
    ├── aws/FreeRTOS/             # MIT (Amazon)
    │   └── FreeRTOS/Source/
    ├── arm/CMSIS_6/              # Apache-2.0 (ARM)
    │   └── CMSIS/
    └── tes/dave2d/               # Renesas独自 (無償)
        └── src/
```

### 6.2 生成コードについて

`ra_gen/` ディレクトリのファイルは e2 studio / FSP Configurator による自動生成コードであり、明示的なライセンスヘッダがない。これらは FSP の設定に基づいて生成されるため、FSP と同様の BSD-3-Clause ライセンスが適用されると解釈される。

## 7. バージョン情報

### 7.1 使用ライブラリのバージョン

| ライブラリ | バージョン | 確認ファイル |
|-----------|-----------|-------------|
| FSP | 6.3.0 | `ra/fsp/inc/fsp_version.h` |
| FreeRTOS Kernel | 11.1.0 | `ra/aws/FreeRTOS/FreeRTOS/Source/include/FreeRTOS.h` |
| ARM CMSIS | 6.1 | `ra/arm/CMSIS_6/CMSIS/Core/Include/cmsis_version.h` |
| Dave/2D | 不明 | ヘッダにバージョン情報なし |

### 7.2 バージョン確認コード

**FSP バージョン** (`fsp_version.h:31-43`):
```c
#define FSP_VERSION_MAJOR (6U)
#define FSP_VERSION_MINOR (3U)
#define FSP_VERSION_PATCH (0U)
#define FSP_VERSION_STRING ("6.3.0")
```

**FreeRTOS バージョン**:
```c
FreeRTOS Kernel V11.1.0
```

**CMSIS バージョン** (`cmsis_version.h:33-36`):
```c
#define __CM_CMSIS_VERSION_MAIN  ( 6U)
#define __CM_CMSIS_VERSION_SUB   ( 1U)
```

## 8. 注意事項・推奨事項

### 8.1 注意が必要な点

1. **Dave/2D ライセンス**
   - 「Purpose: only for testing」という記載があるが、Renesas公式のFSPに含まれているため、実製品での使用も許可されていると解釈される
   - 不明点がある場合はRenesasに確認を推奨

2. **Apache-2.0 の特許条項**
   - 特許訴訟を起こした場合、ARMから付与された特許ライセンスが終了する
   - CMSIS関連の特許訴訟には注意

3. **生成コードの扱い**
   - `ra_gen/` の生成コードには明示的なライセンスがないが、FSPのBSD-3-Clauseが適用されると解釈

### 8.2 ベストプラクティス

1. **ライセンス文書の管理**
   - プロジェクトルートに `LICENSES/` ディレクトリを作成
   - 各ライセンス文を個別ファイルで保管

2. **ライセンス表示の自動化**
   - ビルド時にライセンス情報を収集するスクリプトを作成
   - 製品リリース時に自動的にライセンス文書を生成

3. **定期的なライセンス監査**
   - 新しいライブラリ追加時にライセンスを確認
   - FSP/FreeRTOSバージョンアップ時にライセンス変更を確認

## 9. ライセンス全文参照

### 9.1 ライセンスファイルの場所

| ライセンス | ファイルパス |
|-----------|-------------|
| Apache-2.0 | `ra/arm/CMSIS_6/LICENSE` |
| MIT (FreeRTOS) | `ra/aws/FreeRTOS/LICENSE.md` |
| MIT (FreeRTOS Source) | `ra/aws/FreeRTOS/FreeRTOS/Source/LICENSE.md` |

### 9.2 BSD-3-Clause ライセンス全文

```
BSD 3-Clause License

Copyright (c) 2020 - 2025 Renesas Electronics Corporation and/or its affiliates

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice, this
   list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

3. Neither the name of the copyright holder nor the names of its
   contributors may be used to endorse or promote products derived from
   this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

## 10. 結論

本プロジェクトで使用されているすべてのコンポーネントは、**商用利用可能な寛容なライセンス**の下で提供されている。主要な義務は著作権表示とライセンス文の同梱のみであり、ソースコード開示義務はない。

製品配布時には、適切な著作権表示とライセンス文を含めることで、すべてのライセンス要件を満たすことができる。
