# C# Desktop Tools

[← Portfolio index](../README.md) · [繁體中文版](06-csharp-desktop-tools.zh-TW.md)

![C#](https://img.shields.io/badge/C%23-239120?logo=csharp&logoColor=white) ![Avalonia](https://img.shields.io/badge/Avalonia-8B44AC) ![.NET 10](https://img.shields.io/badge/.NET%2010-512BD4?logo=dotnet&logoColor=white) ![license: MIT](https://img.shields.io/badge/license:%20MIT-blue) ![public source](https://img.shields.io/badge/public%20source-2ea44f)

> Four public repositories spanning 2019 to 2026, from Windows Forms on .NET Framework to Avalonia with MVVM on .NET 10. **These are small and this document says so.** They are here because they are the only readable source in this portfolio, and because the one with users is the one with 81 lines in it.

## 1. At a glance

| Item | Detail |
|---|---|
| **Type** | Four standalone desktop utilities |
| **Role** | Sole author |
| **Period** | 2019-12 to 2026-05 · 50 commits total |
| **Size** | ~3,700 lines of C# across four repositories |
| **State** | **Public on GitHub**: the only readable code in this portfolio |
| **Result worth reporting** | The 81-line tool is the only one with users; the 2,450-line one has none |

| Tool | Size | Stack | Link |
|---|---|---|---|
| **CryptoWidget** | 23 files · ~2,450 lines · 30 commits · MIT · v1.0.8.0 | Avalonia · MVVM · AutoMapper · ccxt · resource i18n ×3 | [repo](https://github.com/hawiliu/CryptoWidget) |
| **aiueo** | 8 files · 818 lines · 3 commits in one day | Windows Forms · .NET Framework | [repo](https://github.com/hawiliu/aiueo) |
| **imageZoom** | 5 files · ~315 lines · 6 commits | Windows Forms · .NET 10 · `System.Drawing` | [repo](https://github.com/hawiliu/imageZoom) |
| **TWjpRunner** | 1 file · **81 lines** · 11 commits · **5 stars** | Console · `System.Management` (WMI) · `Microsoft.Extensions.Configuration` | [repo](https://github.com/hawiliu/TWjpRunner) |

## 2. Tech stack, by tool

| | Language / Framework | UI | Notable APIs |
|---|---|---|---|
| CryptoWidget | C# · Avalonia UI · .NET | MVVM: Views / ViewModels / Services / Dto / Converters | ccxt · AutoMapper · `Resources.Designer.cs` i18n · system tray · transparent borderless window |
| imageZoom | C# · .NET 10 | Windows Forms | `System.Drawing` batch resize |
| aiueo | C# · .NET Framework | Windows Forms | `App.config` only |
| TWjpRunner | C# · console | none | `ManagementObjectSearcher` WMI query · `ProcessStartInfo` with `Verb = "runas"` |

## 3. Architecture (CryptoWidget, the only one with any)

```text
CryptoWidget/
├── Services/
│   ├── ExchangeService.cs         ccxt access
│   ├── Util/ExchangeUtil.cs
│   ├── Dto/                       ExchangeDto · KLineDto · LanguageDto · SettingsDto
│   ├── AutoMapper/SettingsProfile.cs
│   └── Converter/BoolToGridLengthConverter.cs
├── ViewModels/                    Main · Setting · KLine · ExchangePositions · About · ViewModelBase
├── Views/                         MainWindow · SettingsWindow · KLineWindow · ExchangePositionsWindow · AboutWindow
├── i18n/Resources.Designer.cs     en · zh-TW · zh-CN
└── App.axaml.cs / Program.cs / ViewLocator.cs
```

**Avalonia chose the architecture.** Its binding model makes view-models the path of least resistance, whereas Windows Forms makes code-behind the path of least resistance. The other three tools have no layering because they had no reason to.

## 4. What was built

| Mechanism | Built with | Problem it solves |
|---|---|---|
| **Command-line recovery from a live process** | `SELECT CommandLine FROM Win32_Process WHERE ProcessId = ...` via `ManagementObjectSearcher` | The argument string exists only in memory, on a process you did not start, for as long as it runs |
| **Relaunch under elevation** | `ProcessStartInfo` with `UseShellExecute = true, Verb = "runas"` | The locale emulator requires elevation |
| **Fail-fast configuration** | `appsettings.json` path check that prints and exits | Proceeding without the emulator produces a confusing failure later |
| **Cross-platform desktop widget** | Avalonia instead of WPF/Windows Forms | The requirement was not-Windows-only; the framework then forced MVVM |
| **Binding-driven layout** | `BoolToGridLengthConverter` | The view binds a layout dimension to a boolean, so no code-behind reaches into the visual tree |
| **Exchange-agnostic data access** | ccxt | Adding a venue is configuration rather than code |
| **Three-language localisation** | Resource files, `Resources.Designer.cs` | For a public repository the README is the product, and a one-language widget is a one-person widget |
| **Six-year framework migration** | imageZoom, .NET Framework → .NET 10 | A 315-line tool is a cheap place to find out what a migration costs on something never designed for portability |

## 5. Key implementation details

<details>
<summary><b>The 81-line one is the only one anyone uses</b></summary>

`TWjpRunner` launches an application through a locale emulator. The problem is more specific than that sounds: the application's own launcher starts the target process itself, with arguments the launcher generates, so by the time the process exists it has already been started the wrong way.

The solution is to let it start, take its command line off the live process, kill it, and start it again correctly:

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
    Verb = "runas",                       // elevation required by the emulator
    Arguments = $"-run {TwCommandLine}",
    FileName = LEProcPath
});
```

The command line is not available from `Process` itself, so it comes from WMI:

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

**The code is trivial and the insight is not, and the insight is entirely about where the seam is.** Anyone can write a process launcher. The work was noticing that the argument string exists only in memory, on a process you did not start, for exactly as long as it is running, and that `System.Management` can read it while it does. Configuration is an `appsettings.json` with one path in it and a fail-fast check that prints and exits rather than proceeding without the emulator.

**Limitations:** it polls the process list once a second, it depends on a third-party emulator's install path, and it is Windows-only by construction because WMI is. It has not been touched since 2022 and does not need to be.

</details>

<details>
<summary><b>CryptoWidget: the framework chose the architecture</b></summary>

An always-on-top transparent desktop widget showing live prices across exchanges, with a mini candlestick window and a position monitor reporting unrealised profit and loss for spot and futures. MIT, version 1.0.8.0.

**Avalonia rather than WPF or Windows Forms**, because the requirement was a desktop widget that is not Windows-only. That single choice produced the MVVM structure: Avalonia's binding model makes view-models the path of least resistance, whereas Windows Forms makes code-behind the path of least resistance. The layout follows from it:

```text
Services/          ExchangeService, ExchangeUtil
Services/Dto/      ExchangeDto, KLineDto, LanguageDto, SettingsDto
Services/AutoMapper/  SettingsProfile
Services/Converter/   BoolToGridLengthConverter
ViewModels/        Main, Setting, KLine, ExchangePositions, About, ViewModelBase
Views/             MainWindow, SettingsWindow, KLineWindow, ExchangePositionsWindow, AboutWindow
i18n/              Resources.Designer.cs
```

A value converter like `BoolToGridLengthConverter` is a small thing that only exists in a binding-driven UI: the view binds a layout dimension to a boolean and the converter does the translation, so no code-behind reaches into the visual tree to resize a row.

**Exchange access is through ccxt**, so adding a venue is configuration rather than code. **Three-language localisation** (English, Traditional and Simplified Chinese) through resource files, in a personal tool, because a public repository's README is the product and a widget whose settings are in one language is a widget for one person.

**From its own README:** the cross-platform claim is **untested on macOS and Linux**. Avalonia makes it plausible, not verified, and the README says so rather than claiming three platforms. No tests.

</details>

<details>
<summary><b>imageZoom and aiueo: the endpoints of a seven-year line</b></summary>

`imageZoom` is a 315-line batch image resizer: enter a width and height, select images, resized copies land in a `resize\` subfolder. Started in 2020 on .NET Framework and **brought forward to .NET 10 in 2026**, which is the only reason it is worth a paragraph: a small Windows Forms tool is a cheap place to find out what a six-year framework migration costs on something never designed for portability. Its README is a **step-by-step setup guide written for someone who has never installed a .NET SDK**, down to which download link to click, because a public repository's first reader is often not a C# developer.

`aiueo` is a hiragana and katakana drilling program, 818 lines, **three commits on a single day in December 2019**, untouched since. It is the baseline: the oldest public code of mine, written in a day, with no separation of concerns, all logic in the form, state in fields, and no settings persistence beyond `App.config`. Left as it is deliberately, because rewriting it would delete the comparison with `CryptoWidget`.

</details>

## 6. What these four are not

They are not systems. No concurrency problem, no data pipeline, nothing distributed, and **zero tests between them**. Every substantive engineering claim in this portfolio comes from the other five documents.

What they are is the readable part. The five larger systems are private for employer or commercial reasons, so those documents ask to be taken on trust. These do not: the code is public, the commit history is public, and the 81-line one can be read in a minute.

**And the result worth reporting is the ranking.** `CryptoWidget` is thirty times the size of `TWjpRunner`, has MVVM layering, three-language localisation, a chart window and a position monitor, and has no users but me. Size, architecture and polish turned out to be uncorrelated with whether anyone wanted the thing. The tool that got used solves a problem its user could not solve another way; the one that did not solves a problem several other programs already solve.

---

[Portfolio index](../README.md)

---
