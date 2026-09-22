# C# 桌面小工具

[← 作品集索引](../README.zh-TW.md) · [English version](06-csharp-desktop-tools.md)

![C#](https://img.shields.io/badge/C%23-239120?logo=csharp&logoColor=white) ![Avalonia](https://img.shields.io/badge/Avalonia-8B44AC) ![.NET 10](https://img.shields.io/badge/.NET%2010-512BD4?logo=dotnet&logoColor=white) ![license: MIT](https://img.shields.io/badge/license:%20MIT-blue) ![public source](https://img.shields.io/badge/public%20source-2ea44f)

> 四個公開 repository，橫跨 2019 到 2026，從 .NET Framework 上的 Windows Forms 到 .NET 10 上的 Avalonia 加 MVVM。**它們很小，而這份文件就這麼說。** 它們在這裡，是因為它們是這份作品集裡唯一能讀的原始碼，也因為有使用者的那一個只有 81 行。

## 1. 專案綱要

| 項目 | 內容 |
|---|---|
| **類型** | 四個獨立的桌面工具 |
| **角色** | 獨力完成 |
| **期間** | 2019-12 至 2026-05 · 合計 50 個 commit |
| **規模** | 四個 repository 合計約 3,700 行 C# |
| **狀態** | **GitHub 上公開**：這份作品集裡唯一能讀的程式碼 |
| **值得回報的結果** | 81 行那個是唯一有使用者的；2,450 行那個沒有 |

| 工具 | 規模 | 技術 | 連結 |
|---|---|---|---|
| **CryptoWidget** | 23 檔 · 約 2,450 行 · 30 commit · MIT · v1.0.8.0 | Avalonia · MVVM · AutoMapper · ccxt · 三語資源檔 i18n | [repo](https://github.com/hawiliu/CryptoWidget) |
| **aiueo** | 8 檔 · 818 行 · 一天內 3 個 commit | Windows Forms · .NET Framework | [repo](https://github.com/hawiliu/aiueo) |
| **imageZoom** | 5 檔 · 約 315 行 · 6 commit | Windows Forms · .NET 10 · `System.Drawing` | [repo](https://github.com/hawiliu/imageZoom) |
| **TWjpRunner** | 1 檔 · **81 行** · 11 commit · **5 stars** | Console · `System.Management`(WMI)· `Microsoft.Extensions.Configuration` | [repo](https://github.com/hawiliu/TWjpRunner) |

## 2. 逐工具技術棧

| | 語言 / 框架 | UI | 值得一提的 API |
|---|---|---|---|
| CryptoWidget | C# · Avalonia UI · .NET | MVVM:Views / ViewModels / Services / Dto / Converters | ccxt · AutoMapper · `Resources.Designer.cs` i18n · 系統匣 · 半透明無邊框視窗 |
| imageZoom | C# · .NET 10 | Windows Forms | `System.Drawing` 批次縮放 |
| aiueo | C# · .NET Framework | Windows Forms | 只有 `App.config` |
| TWjpRunner | C# · console | 無 | `ManagementObjectSearcher` WMI 查詢 · `ProcessStartInfo` 帶 `Verb = "runas"` |

## 3. 架構（只有 CryptoWidget 有）

```text
CryptoWidget/
├── Services/
│   ├── ExchangeService.cs         ccxt 存取
│   ├── Util/ExchangeUtil.cs
│   ├── Dto/                       ExchangeDto · KLineDto · LanguageDto · SettingsDto
│   ├── AutoMapper/SettingsProfile.cs
│   └── Converter/BoolToGridLengthConverter.cs
├── ViewModels/                    Main · Setting · KLine · ExchangePositions · About · ViewModelBase
├── Views/                         MainWindow · SettingsWindow · KLineWindow · ExchangePositionsWindow · AboutWindow
├── i18n/Resources.Designer.cs     en · zh-TW · zh-CN
└── App.axaml.cs / Program.cs / ViewLocator.cs
```

**Avalonia 選了架構。** 它的繫結模型讓 view-model 成為阻力最小的路，而 Windows Forms 讓 code-behind 成為阻力最小的路。另外三個工具沒有分層，因為它們沒有理由要有。

## 4. 做了哪些事

| 做法 | 用什麼做 | 解決什麼問題 |
|---|---|---|
| **從活著的行程取回命令列** | `SELECT CommandLine FROM Win32_Process WHERE ProcessId = ...`，經 `ManagementObjectSearcher` | 那個參數字串只存在記憶體、在一個不是你啟動的行程上、只在它還活著的期間 |
| **以提權重新啟動** | `ProcessStartInfo` 帶 `UseShellExecute = true, Verb = "runas"` | 語系模擬器需要提權 |
| **Fail-fast 設定檢查** | `appsettings.json` 路徑檢查，印訊息後結束 | 沒有模擬器還繼續跑，會在後面產生令人困惑的失敗 |
| **跨平台桌面小工具** | 選 Avalonia 而不是 WPF / Windows Forms | 需求是不限 Windows；框架接著逼出了 MVVM |
| **繫結驅動的佈局** | `BoolToGridLengthConverter` | view 把佈局尺寸綁到布林值，所以沒有 code-behind 需要伸進視覺樹 |
| **與交易所無關的資料存取** | ccxt | 加一個交易所是設定而不是程式碼 |
| **三語言本地化** | 資源檔、`Resources.Designer.cs` | 對公開 repository 來說 README 就是產品，而單語言的小工具是一個人的小工具 |
| **跨六年的框架遷移** | imageZoom,.NET Framework → .NET 10 | 315 行的工具是很便宜的地方，去弄清楚遷移在一個沒為可攜性設計的東西上要付什麼代價 |

## 5. 關鍵實作細節

<details>
<summary><b>那個 81 行的是唯一有人用的</b></summary>

`TWjpRunner` 透過語系模擬器啟動一個應用程式。問題比這句話更具體：那個應用程式自己的啟動器會去啟動目標行程，而且帶著啟動器自己生成的參數，所以等到那個行程存在時，它已經被用錯的方式啟動了。

解法是讓它先起來、從活著的行程上把命令列取下來、殺掉它、再用正確的方式重新啟動：

```csharp
while (true)
{
    var tw = Process.GetProcesses().FirstOrDefault(p => p.ProcessName.Equals("..."));
    if (tw != null)
    {
        TwCommandLine = GetCommandLine(tw);
        tw.Kill();
        break;
    }
    Thread.Sleep(TimeSpan.FromSeconds(1));
}

Process.Start(new ProcessStartInfo()
{
    UseShellExecute = true,
    Verb = "runas",                       // 模擬器需要提權
    Arguments = $"-run {TwCommandLine}",
    FileName = LEProcPath
});
```

命令列無法從 `Process` 本身取得，所以它來自 WMI:

```csharp
private static string GetCommandLine(this Process process)
{
    using (var searcher = new ManagementObjectSearcher(
        "SELECT CommandLine FROM Win32_Process WHERE ProcessId = " + process.Id))
    using (var objects = searcher.Get())
    {
        return objects.Cast<ManagementBaseObject>()
                      .SingleOrDefault()?["CommandLine"]?.ToString();
    }
}
```

**程式碼很平凡，洞見不平凡，而那個洞見完全是關於接縫在哪裡。** 任何人都會寫一個行程啟動器。工作量在於注意到：那個參數字串只存在於記憶體中、在一個不是你啟動的行程上、而且只在它還活著的那段時間，以及 `System.Management` 可以在那段時間內把它讀出來。設定是一個裡面放一條路徑的 `appsettings.json`，加上一個 fail-fast 檢查，印訊息後結束，而不是在沒有模擬器的情況下繼續跑。

**限制：** 它每秒輪詢一次行程列表、依賴一個第三方模擬器的安裝路徑，而且在構造上只能跑 Windows，因為 WMI 本來就只有 Windows 有。自 2022 年後沒有被動過，也不需要。

</details>

<details>
<summary><b>CryptoWidget：框架選了架構</b></summary>

一個永遠置頂的半透明桌面小工具，顯示跨交易所的即時價格，帶一個迷你 K 線視窗與一個部位監控，回報現貨與合約的未實現損益。MIT，版本 1.0.8.0。

**選 Avalonia 而不是 WPF 或 Windows Forms**，因為需求是一個不限 Windows 的桌面小工具。那一個選擇就產生了 MVVM 結構：Avalonia 的繫結模型讓 view-model 成為阻力最小的路，而 Windows Forms 讓 code-behind 成為阻力最小的路。佈局是它的結果：

```text
Services/          ExchangeService、ExchangeUtil
Services/Dto/      ExchangeDto、KLineDto、LanguageDto、SettingsDto
Services/AutoMapper/  SettingsProfile
Services/Converter/   BoolToGridLengthConverter
ViewModels/        Main、Setting、KLine、ExchangePositions、About、ViewModelBase
Views/             MainWindow、SettingsWindow、KLineWindow、ExchangePositionsWindow、AboutWindow
i18n/              Resources.Designer.cs
```

像 `BoolToGridLengthConverter` 這種 value converter 是只會存在於繫結驅動 UI 的小東西：view 把一個佈局尺寸綁到一個布林值，由 converter 做轉換，所以沒有任何 code-behind 需要伸進視覺樹去調整一列的高度。

**交易所存取走 ccxt**，所以加一個交易所是設定而不是程式碼。**三語言本地化**（英文、繁體與簡體中文）透過資源檔，在一個個人工具裡；之所以做，是因為對一個公開 repository 來說 README 就是產品，而一個設定只有一種語言的小工具是一個人的小工具。

**引自它自己的 README:** 跨平台這個宣稱**在 macOS 與 Linux 上未經測試**。Avalonia 讓這個宣稱說得過去，但沒有驗證過，而 README 就這麼寫，沒有宣稱支援三個平台。沒有測試。

</details>

<details>
<summary><b>imageZoom 與 aiueo：一條七年線的兩端</b></summary>

`imageZoom` 是 315 行的批次圖片縮放工具：輸入寬高、選圖片，縮放後的副本落在 `resize\` 子資料夾。2020 年在 .NET Framework 上開始，**2026 年被搬到 .NET 10**，這是它值得一段文字的唯一理由：一個小的 Windows Forms 工具，是很便宜的地方去弄清楚一次跨六年的框架遷移，在一個當初沒為可攜性設計的東西上要付什麼代價。它的 README 是一份**寫給從來沒安裝過 .NET SDK 的人的逐步設定指南**，細到該點哪個下載連結，因為一個公開 repository 的第一位讀者常常不是 C# 開發者。

`aiueo` 是平假名與片假名的練習程式，818 行，**2019 年 12 月某一天的三個 commit**，之後沒動過。它是基準線：我最舊的公開程式碼、在一天內寫成、沒有關注點分離、所有邏輯都在表單裡、狀態放在欄位上、除了 `App.config` 沒有設定持久化。刻意留成原樣，因為重寫它會刪掉與 `CryptoWidget` 的那個對比。

</details>

## 6. 這四個不是什麼

它們不是系統。沒有並行問題、沒有資料管線、沒有任何分散式的東西，而且四個加起來**零個測試**。這份作品集裡所有實質的工程宣稱都來自另外五份文件。

它們是可讀的那一部分。那五個比較大的系統因為雇主或商業理由維持私有，所以那些文件要求讀者信任它。這四個不要求：程式碼公開、commit 歷史公開，而那個 81 行的一分鐘就能讀完。

**而值得回報的結果是那個排序。** `CryptoWidget` 是 `TWjpRunner` 的三十倍大，有 MVVM 分層、三語言本地化、一個圖表視窗與一個部位監控，而除了我沒有使用者。規模、架構與完成度，最後證明跟有沒有人想要它完全不相關。被用的那個工具解決了它的使用者沒有別的辦法解決的問題；沒被用的那個，解決的問題已經有好幾個別的程式在解決了。

---

[作品集索引](../README.zh-TW.md)

---
