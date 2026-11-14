# Heck / Chroma / NoodleExtensions for Beat Saber 1.44.1（非官方）

這是 [Aeroluna/Heck](https://github.com/Aeroluna/Heck) 的個人 fork，目的是讓 **Beat Saber 1.44.1** 先能正常玩
Noodle Extensions 和 Chroma 的譜面，給自己和朋友使用。

- Heck、Chroma、NoodleExtensions 的程式碼來自上游 `dev` 分支的 1.44.1 支援，**沒有修改**，只是先編譯出來。
- 另外附上一份**修正過的 MonoMod.Core**，修掉 1.44.1 上 Chroma 譜面隨機「沒東西／黑屏／當機」的問題（說明見下方）。

> 這不是官方版本。Aeroluna 或 BSIPA 釋出正式更新後，請改用官方版本。
> 使用這個版本遇到的問題，**請不要回報給 Heck 原作者**。

## 下載與安裝

到 [Releases](https://github.com/Akiraoo/Heck-1.44.1/releases) 下載最新的 zip。解壓後的結構和遊戲資料夾相同：

```
Libs/MonoMod.Core.dll            ← 修正版（取代 BSIPA 附帶的版本）
Plugins/Heck.dll
Plugins/Chroma.dll
Plugins/NoodleExtensions.dll
Plugins/CustomJSONData.dll       ← 為 1.44.1 編譯的版本
Plugins/SiraUtil.dll             ← 3.4.0
LICENSE-*.txt
```

1. 關閉 Beat Saber。
2. 備份遊戲資料夾裡原本的 `Libs\MonoMod.Core.dll`。
3. 把 zip 裡的 `Libs`、`Plugins` 覆蓋到遊戲資料夾（有 `Beat Saber.exe` 的那一層）。

### 還需要從 BSManager／BeatMods 安裝的模組

- BSIPA
- BeatSaberMarkupLanguage（BSML）
- LookupID（Chroma 需要；授權不明，所以不包含在 zip 裡）

注意：

- **CustomJSONData** 請用 zip 裡的版本。BSManager 上的 `2.6.8+1.40.0` 是給 1.40 編譯的。
  zip 裡的版本是從上游 `dev` 分支為 1.44.1 編譯的，但版本號字串一樣顯示 `2.6.8+1.40.0`，這是上游還沒改版號，屬於正常現象。
- **SiraUtil** 請用 zip 裡的 3.4.0。BSManager 上的 3.3.1 在 1.44.1 會讓 Noodle 譜面出錯。
- 用 BSManager 重新安裝或修復 BSIPA 時，`Libs\MonoMod.Core.dll` 可能被換回原版，請再覆蓋一次。

## 為什麼要換 MonoMod.Core？

模組改變遊戲行為時，最底層是靠 **MonoMod** 直接改寫記憶體裡的機器碼。它會在遊戲方法前面插入「改走這邊」的跳轉（detour）。
如果跳轉目標太遠，MonoMod 會在附近另外放一個小格子，記錄完整位址；這個格子**必須**放在 ±2GB 以內。

BSIPA 附帶的 MonoMod.Core 1.3.3 有 bug：附近的空間用完時（Chroma 的 patch 特別多，最容易遇到），它會向 Windows 要
「位址 0」，而 Windows 把 0 解讀成「任意位址」，回傳遠在約 7GB 外的記憶體。MonoMod 沒有檢查就拿來用，結果跳轉位址被截斷，
程式跳到錯誤的地方：

- 輕微時：遊戲記錄大量 `NullReferenceException`（停在 `Trampoline<...>`），譜面沒東西。
- 嚴重時：黑屏或整個遊戲當掉。

因為要「剛好用完附近的空間」才會發生，所以是**隨機**的，而且掛上 debugger 時不會出現。

修正版加上了範圍檢查：不再向系統要位址 0、拒絕範圍外的記憶體；萬一還是太遠，就改用不需要小格子、多遠都能跳的寫法。
技術細節（含 debugger 抓到的現場）見 [`MonoModFix/README.md`](MonoModFix/README.md)。

## 分支

| 分支 | 用途 |
|---|---|
| `monomod-core-fix`（預設） | 這個 fork 的主要版本：上游 1.44.1 支援 + 修正版 MonoMod 原始碼 + 本說明。Releases 都從這裡建立 |
| `master` | 上游 [Aeroluna/Heck](https://github.com/Aeroluna/Heck) 的鏡像，不放任何自己的修改，只用來同步上游 |

上游有更新時：先把上游同步到 `master`，再把 `master` 合進 `monomod-core-fix`。
通常只有根目錄 `README.md` 會衝突，保留這邊的版本即可。

## 原始碼結構

| 路徑 | 內容 |
|---|---|
| `Heck/`、`Chroma/`、`NoodleExtensions/` | 上游 Aeroluna/Heck 的原始碼（1.44.1 支援，未修改） |
| `MonoMod/` | 修正版 MonoMod 的完整原始碼。以 MonoMod commit [`aa4a847`](https://github.com/MonoMod/MonoMod/commit/aa4a8474906ded4423d222cdf9ea5cf884c32b5f)（MonoMod.Core 1.3.3）為基礎，只修改 `src/MonoMod.Core/Platforms/` 底下的 3 個檔案 |
| `MonoMod/external/iced` | git submodule（iced 1.21.0，MonoMod 原本就依賴它） |
| `MonoModFix/` | 技術說明，以及相對於上游 MonoMod 的 patch |

## 自己編譯

需要 .NET SDK 10，以及一個 Beat Saber 1.44.1 的安裝（編譯時要參照遊戲的 DLL）。

```bash
git clone --recurse-submodules https://github.com/Akiraoo/Heck-1.44.1.git
cd Heck-1.44.1
```

**MonoMod.Core**

```bash
cd MonoMod
dotnet build src/MonoMod.Core/MonoMod.Core.csproj -c Release -f net452
# 輸出：MonoMod/artifacts/bin/MonoMod.Core/release_net452/MonoMod.Core.dll
```

**Heck / NoodleExtensions / Chroma**

這三個專案的建置工具放在 GitHub Packages 上，需要一個有 `read:packages` 權限的 GitHub token 才能還原 NuGet 套件。

```bash
dotnet build Heck/Heck.csproj -c Release-1.44.1 "-p:BeatSaberDir=<Beat Saber 1.44.1 的路徑>"
dotnet build NoodleExtensions/NoodleExtensions.csproj -c Release-1.44.1 "-p:BeatSaberDir=<Beat Saber 1.44.1 的路徑>"
dotnet build Chroma/Chroma.csproj -c Release-1.44.1 "-p:BeatSaberDir=<Beat Saber 1.44.1 的路徑>"
```

CustomJSONData 請從 [Aeroluna/CustomJSONData](https://github.com/Aeroluna/CustomJSONData) 的 `dev` 分支用同樣的 `Release-1.44.1` 設定編譯。

## 致謝與授權

- Heck、Chroma、NoodleExtensions、CustomJSONData：[Aeroluna](https://github.com/Aeroluna)，MIT（見 [`LICENSE`](LICENSE)）。
  喜歡這些模組的話，請到 [Ko-Fi](https://ko-fi.com/aeroluna) 支持原作者。
- MonoMod：[MonoMod contributors](https://github.com/MonoMod/MonoMod)，MIT（見 [`MonoMod/LICENSE`](MonoMod/LICENSE)）。
- SiraUtil：[Auros](https://github.com/Auros/SiraUtil)，MIT。
- Heck 各模組的功能說明請看官方 wiki：[heck.aeroluna.dev](https://heck.aeroluna.dev/)
