# MonoMod.Core 1.3.3 修正（Beat Saber 1.44.1 / Chroma）

在 Beat Saber 1.44.1（Unity 6、Mono）上，Chroma 的燈光事件譜面會出現「第一次載入沒東西、黑屏或原生當機」，
而且有機率發生。原因不在 Heck／Chroma／NoodleExtensions，而是 BSIPA 附帶的 **MonoMod.Core 1.3.3** 的記憶體配置器有 bug。
這個資料夾放的是修改版 MonoMod.Core 的原始碼與 patch。

這是非官方的個人版本，Heck 作者或 BSIPA 釋出正式更新後，請改用官方版本。

## 直接安裝（給一般玩家）

到本 repo 的 Releases 下載 zip，解壓縮後的結構與遊戲資料夾相同：

```
Libs/MonoMod.Core.dll
Plugins/Heck.dll
Plugins/Chroma.dll
Plugins/NoodleExtensions.dll
```

1. 關閉 Beat Saber。
2. 備份遊戲資料夾裡原本的 `Libs\MonoMod.Core.dll`。
3. 把 zip 裡的 `Libs`、`Plugins` 直接覆蓋到遊戲資料夾（也就是有 `Beat Saber.exe` 的那一層）。

還需要的相依模組（zip 不包含）：BSIPA、SiraUtil、BSML、CustomJSONData（1.44.1 版）、LookupID。

用 BSManager 重新安裝或修復 BSIPA 時，`Libs\MonoMod.Core.dll` 可能被換回原版，請再覆蓋一次。

## 原始碼

- `src/MonoMod.Core/...`：修改過的三個檔案（完整內容），對應 MonoMod repo 裡的同路徑檔案。
- `monomod-core-1.3.3-fix.patch`：同樣的修改，以 diff 形式呈現。
- `MonoMod-LICENSE.txt`：MonoMod 的授權（MIT）。
- Heck、Chroma、NoodleExtensions 的原始碼就是本 repo 本身（上游 Aeroluna/Heck `dev` 分支的 1.44.1 支援，未修改）。

## 症狀

- 遊戲記錄出現大量 `NullReferenceException`，堆疊只停在
  `MonoMod.Utils.DynamicMethodDefinition.Trampoline<...>`，例如 `PlayerTransforms`、`SaberMovementData`、`ObstacleController`。
- patch 本身的程式碼完全沒被執行到。
- 只有 patch 數量多的時候才會發生（Chroma 燈光事件 + 環境模組），而且有機率；掛上原生 debugger 時不會發生。

## 根本原因

x64 上，當 trampoline 與目標距離超過 ±2GB，MonoMod 會改用 `jmp [rip+disp32]`，並另外配置一個 8 bytes 的跳轉格
（`Rel32Ind64`）。這個跳轉格**必須**位在 trampoline 前後 ±2GB 以內。

1. trampoline 位在約 `0x7ab6f966` 這種低位址時，±2GB 的下界會被夾成 `0`。
2. 低位址區的空間用完後，`QueryingPagedMemoryAllocator` 一路往下找，最後找到位址 `0`。
3. 它呼叫 `VirtualAlloc(0, ...)` 想在位址 0 配置，但 Windows 把 `0` 解讀成「任意位址」，於是回傳約 7GB 外的頁，例如 `0x23b170000`。
4. MonoMod 1.3.3 沒有檢查回傳位址，直接把它當成跳轉格。`disp32` 被截斷，`jmp` 讀到錯誤或未對應的記憶體，
   Mono 把存取違規轉成 NRE；讀到垃圾資料時則直接原生當機。

用 cdb 抓到的現場：

```
00000000`7aa8f040 ff25ba0fc8c2  jmp qword ptr [00000000`3d710000]  ds:00000000`3d710000=????????????????
```

一次失敗的載入中，2266 個跳轉格裡有 15 個超出範圍，全部發生在出錯的那一次載入。

MonoMod 上游 `main` 已經在 `TryAllocNewPage` 加入範圍檢查，但截至 MonoMod.Core 1.3.6 正式版都還沒包含這個修正。

## 修正內容（`monomod-core-1.3.3-fix.patch`）

以 MonoMod commit `aa4a8474906ded4423d222cdf9ea5cf884c32b5f`（MonoMod.Core 1.3.3，BSIPA 4.3.7 附帶的版本）為基礎：

- `x86_64Arch.ComputeDetourInfo`：確認跳轉格真的在 rel32 範圍內，否則改用 `Abs64`（任何距離都能跳）。
- `Rel32Ind64Kind.GetBytes`：位移放不進 32 位元時直接丟例外，不會再寫出錯誤的跳轉。
- `QueryingPagedMemoryAllocator.TryAllocNewPage`：不再對位址 0 呼叫配置（避免變成「任意位址」），並拒絕範圍外的配置。
- 從上游移植的配置器修正：`0267d962a` / `d6bf3d8e8`（PR #335，重複釋放頁）、`eb521dabc`、`3fd91a1a4`、`a86ce7ae8`。

## 編譯與安裝

```bash
git clone https://github.com/MonoMod/MonoMod.git && cd MonoMod
git checkout aa4a8474906ded4423d222cdf9ea5cf884c32b5f
git submodule update --init --depth 1
git apply /path/to/monomod-core-1.3.3-fix.patch   # 或把 MonoModFix/src 底下的三個檔案直接覆蓋過去
# global.json 鎖定 SDK 10.0.100；若只有較新的 SDK，把 rollForward 改成 latestMajor
dotnet build src/MonoMod.Core/MonoMod.Core.csproj -c Release -f net452
```

輸出在 `artifacts/bin/MonoMod.Core/release_net452/MonoMod.Core.dll`，組件版本仍是 `1.3.3.0`，相依版本與原版一致，不需要更動 HarmonyX 或 BSIPA。

1. 關閉 Beat Saber。
2. 備份 `<Beat Saber>\Libs\MonoMod.Core.dll`。
3. 用編譯出的檔案取代它。

注意：用 BSManager 重新安裝或修復 BSIPA 時，`Libs` 可能被換回原版，需要重新取代。

## English summary

On Beat Saber 1.44.1 (Unity 6 / Mono, Windows x64), MonoMod.Core 1.3.3's `QueryingPagedMemoryAllocator` can end its downward
page search at address 0 and call `VirtualAlloc(NULL, ...)`, which returns a page anywhere (observed ~7GB away). That page is
then used as a `Rel32Ind64` indirection cell, the rel32 displacement is truncated, and the trampoline jumps through
garbage/unmapped memory (NRE "in Trampoline<...>" or native crash). Chroma's large number of patches makes the low region
run out, so it hits Chroma maps. The patch adds range checks (falling back to Abs64) plus backported allocator fixes.
