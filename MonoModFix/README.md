# MonoMod.Core 1.3.3 修正：技術說明

安裝方式與一般說明見 [根目錄 README](../README.md)。修正後的完整原始碼在 [`../MonoMod`](../MonoMod)，
這個資料夾放的是技術細節，以及相對於上游的 patch（[`monomod-core-1.3.3-fix.patch`](monomod-core-1.3.3-fix.patch)）。

## 環境

- Beat Saber 1.44.1（Unity 6000.0.40、Mono 6.13），Windows x64
- BSIPA 4.3.7，附帶 HarmonyX 2.16、MonoMod.Core 1.3.3、MonoMod.RuntimeDetour 25.3.3、MonoMod.Utils 25.0.11
- 上游基準：MonoMod commit `aa4a8474906ded4423d222cdf9ea5cf884c32b5f`

## 症狀

- 遊戲記錄大量 `NullReferenceException`，堆疊只停在 `MonoMod.Utils.DynamicMethodDefinition.Trampoline<...>`，
  沒有更深的呼叫框架。patch 本身的程式碼完全沒被執行到（加診斷記錄確認過）。
- 只在 patch 數量多時發生（Chroma 燈光事件模組 + 被它帶起的環境模組），有機率；掛上原生 debugger 時不會發生。
- 嚴重時會原生當機（Unity crash handler）。

## 現場

用 cdb 在第一次存取違規時中斷：

```
00000000`7aa8f040 ff25ba0fc8c2  jmp qword ptr [00000000`3d710000]  ds:00000000`3d710000=????????????????
```

trampoline 被改寫成 `Rel32Ind64`（`jmp [rip+disp32]`）。它讀取的跳轉格位址無效，Mono 把存取違規轉成 NRE。

## 根本原因

`x86_64Arch.ComputeDetourInfo` 在目標超過 ±2GB 時，會向配置器要一個位在 `from+6` 前後 ±2GB 內的 8 bytes 跳轉格。

1. trampoline 位在低位址（約 `0x7ab6f966`），下界 `target + int.MinValue` 溢位，被夾成 `0`。
2. 低位址區用完後，`QueryingPagedMemoryAllocator.TryAllocateNewPage` 往下找，最後 `lowPage` 到達 `0`
   （仍滿足 `lowPage >= lowPageBound`）。
3. `TryAllocNewPage` 用 `0` 呼叫 `VirtualAlloc(lpAddress = NULL)`，Windows 解讀為「任意位址」，回傳例如 `0x23b170000`。
4. 1.3.3 沒有檢查回傳的配置是否在範圍內，`Rel32Ind64Kind.GetBytes` 把超過 32 位元的位移截斷後寫入指令。

追蹤記錄（加在測試版 MonoMod.Core 裡）：

```
new-page alloc OUT OF BOUNDS 0x23b170000 target 0x7ab6f966 low 0x0 high 0xfab6f965 lowPage 0x0 highPage 0xfab60000
```

一次失敗的載入中，2266 個跳轉格裡有 15 個超出範圍，全部發生在出錯的那次載入，之後正常的載入一個都沒有。

## 修改內容

`src/MonoMod.Core/Platforms/` 底下 3 個檔案：

- `Architectures/x86_64Arch.cs`
  - `ComputeDetourInfo`：確認跳轉格在 rel32 範圍內，否則釋放它並改用 `Abs64`。
  - `Rel32Ind64Kind.GetBytes`：位移超出 32 位元時丟例外，不再寫出錯誤的跳轉。
- `Memory/QueryingPagedMemoryAllocator.cs`
  - `TryAllocNewPage`：不對位址 `0` 呼叫配置，並拒絕範圍外的配置（與上游 `main` 後來加入的檢查相同性質）。
  - 移植上游 `3fd91a1a4`、`a86ce7ae8`。
- `Memory/PagedMemoryAllocator.cs`
  - 移植上游 `0267d962a`、`d6bf3d8e8`（PR #335，同一頁被重複排入清理而誤釋放新頁）、`eb521dabc`（往低位址找頁時誤用 `lowIdx++`）。

組件版本維持 `1.3.3.0`，相依版本與原版相同，可以直接取代 `Libs/MonoMod.Core.dll`，不需要更動 HarmonyX 或 BSIPA。

截至 MonoMod.Core 1.3.6 正式版，上游都還沒有包含這個範圍檢查。

## English summary

On Beat Saber 1.44.1 (Unity 6 / Mono, Windows x64), MonoMod.Core 1.3.3's `QueryingPagedMemoryAllocator` can end its downward
page search at address 0 and call `VirtualAlloc(NULL, ...)`, which returns a page anywhere (observed ~7GB away). That page is
then used as a `Rel32Ind64` indirection cell, the rel32 displacement is truncated, and the trampoline jumps through
garbage/unmapped memory (NRE "in Trampoline<...>" or native crash). Chroma's large number of patches makes the low region
run out, so it shows up on Chroma maps. The fix adds range checks (falling back to Abs64), never allocates at address 0,
and backports upstream allocator fixes.
