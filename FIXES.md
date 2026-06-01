# Termrig integration fixes

## Summary

This integration branch combines the pending XTerm.NET fixes that Termrig needs while the upstream pull requests are under review. The branch is intentionally limited to terminal emulator behavior. It does not include Termrig host-rendering changes, Avalonia integration changes, PTY policy, or ConPTY-specific workarounds.

Included upstream fixes:

- Text-presentation emoji width for Docker-style progress rows.
- DEC origin-mode cursor positioning with scroll regions.

## Text-presentation emoji width

The confirmed defect was that `InputHandler.GetStringCellWidth` treated any code point classified by `NeoSmart.Unicode.Emoji.IsEmoji` as width 2. That makes U+2714 HEAVY CHECK MARK (`\u2714`) consume two terminal cells even when emitted in text presentation.

Docker Compose emits U+2714 without U+FE0F emoji presentation, and Windows `cmd.exe` renders it as a single-cell icon. XTerm.NET therefore shifted the rest of those progress rows by one cell.

The fix is to use `Wcwidth.UnicodeCalculator.GetWidth` for the base width and keep the existing variation-selector handling:

- `\u2714` remains width 1.
- `\u2714\uFE0F` becomes width 2 because U+FE0F explicitly requests emoji presentation.
- Existing wide emoji and CJK width behavior continues to come from `UnicodeCalculator`.

### Minimal checkmark-width reproduction

```csharp
var terminal = new Terminal(new TerminalOptions { Cols = 20, Rows = 3 });
terminal.Write("\u2714X");

Assert.Equal(2, terminal.Buffer.X);
Assert.Equal("\u2714", terminal.Buffer.Lines[0]![0].Content);
Assert.Equal(1, terminal.Buffer.Lines[0]![0].Width);
Assert.Equal("X", terminal.Buffer.Lines[0]![1].Content);
Assert.Equal(1, terminal.Buffer.Lines[0]![1].Width);
```

### Emoji-presentation checkmark

```csharp
var terminal = new Terminal(new TerminalOptions { Cols = 20, Rows = 3 });
terminal.Write("\u2714\uFE0FX");

Assert.Equal(3, terminal.Buffer.X);
Assert.Equal(2, terminal.Buffer.Lines[0]![0].Width);
Assert.Equal(0, terminal.Buffer.Lines[0]![1].Width);
Assert.Equal(1, terminal.Buffer.Lines[0]![2].Width);
```

## Origin mode and scroll regions

The fixed behavior is:

- `DECSTBM` / `CSI t;b r` moves the cursor to home after setting the scroll region.
- `CUP` / `CSI row;col H` and `HVP` / `CSI row;col f` treat row coordinates as relative to the scroll region when `DECOM` / origin mode is enabled.
- `VPA` / `CSI row d` applies the same origin-mode row translation.
- Enabling origin mode moves the cursor to the top margin of the scroll region; disabling origin mode moves the cursor to absolute home.

Full-screen and prompt-oriented terminal applications often reserve a bottom input or status row by setting a scroll region for the output area. They then use origin-mode cursor addressing inside that region. If the emulator treats those row coordinates as absolute screen rows, application output can be written outside the intended scroll region.

### Scroll region homes the cursor

```csharp
var terminal = new Terminal(new TerminalOptions { Cols = 20, Rows = 5 });
var handler = new InputHandler(terminal);
terminal.Buffer.SetCursor(10, 10);

var parameters = new Params();
parameters.AddParam(2);
parameters.AddParam(4);
handler.HandleCsi("r", parameters);

Assert.Equal(0, terminal.Buffer.X);
Assert.Equal(0, terminal.Buffer.Y);
```

### Origin-mode `CUP` is relative to the scroll region

```csharp
var terminal = new Terminal(new TerminalOptions { Cols = 20, Rows = 5 });
var handler = new InputHandler(terminal);
terminal.Buffer.SetScrollRegion(1, 3);
terminal.OriginMode = true;

var parameters = new Params();
parameters.AddParam(3);
parameters.AddParam(20);
handler.HandleCsi("H", parameters);

Assert.Equal(19, terminal.Buffer.X);
Assert.Equal(3, terminal.Buffer.Y);
```

### Origin-mode `VPA` is relative to the scroll region

```csharp
var terminal = new Terminal(new TerminalOptions { Cols = 20, Rows = 5 });
var handler = new InputHandler(terminal);
terminal.Buffer.SetScrollRegion(1, 3);
terminal.OriginMode = true;
terminal.Buffer.SetCursor(10, 1);

var parameters = new Params();
parameters.AddParam(3);
handler.HandleCsi("d", parameters);

Assert.Equal(10, terminal.Buffer.X);
Assert.Equal(3, terminal.Buffer.Y);
```

## Files changed

- `src/XTerm.NET/InputHandler.cs`
  - Replaced broad emoji classification width override with Unicode cell-width calculation.
  - Added shared row translation for origin-mode cursor addressing.
  - Applied that translation to `CUP` / `HVP` and `VPA`.
  - Homed the cursor after `DECSTBM`.
  - Homed to the top margin when origin mode is enabled.
- `src/XTerm.NET.Tests/InputHandlerTests.cs`
  - Added regression tests for text-presentation checkmark width.
  - Added regression tests for emoji-presentation checkmark width.
  - Added a Docker-style progress alignment test.
  - Added explicit coverage for `CSI Ps C` cursor-forward clamping.
  - Added explicit coverage for `CSI Ps X` erase-character preserving cursor position.
  - Added regression coverage for scroll-region cursor homing.
  - Added regression coverage for origin-relative `CUP` / `HVP`.
  - Added regression coverage for origin-relative `VPA`.
- `src/XTerm.NET.Tests/ModeHandlingTests.cs`
  - Added regression coverage for enabling origin mode with a non-zero top margin.

## Termrig compatibility note

Termrig should consume this branch through a direct project reference while upstream PRs are pending. Local Termrig compatibility shims should be treated as temporary quarantine for behavior that is not yet handled by XTerm.NET. Once Termrig consumes an XTerm.NET build containing the relevant emulator fixes, those shims should be retested and removed where possible.

## Validation

Run from this repository root:

```powershell
dotnet test src/XTerm.NET.slnx --no-restore
```

Result on this branch:

```text
Passed: 589
Failed: 0
Skipped: 0
```
