# tidewin: tide for Linux and tide for Chrome on Windows

**Status: exploration.** Nothing is built yet. This spec takes what tide for
Linux ([mikelward/tide](https://github.com/mikelward/tide)) does on Linux and
tide for Chrome ([mikelward/tideco](https://github.com/mikelward/tideco)) does
on chromeOS, and says, for each one, how Windows can do it, how well, and at
what cost. It follows tide for macOS
([mikelward/tidemo](https://github.com/mikelward/tidemo)), whose SPEC.md asks
the same questions of a Mac.

Claims about Microsoft or third-party behavior carry a source. Anything not
checked against a source or a Windows machine is marked *(unverified)*.

## 1. Goals

tide for macOS's goals hold here, with "macOS" read as "Windows":

- **Everything tide for Linux and tide for Chrome do,** where Windows allows
  it: layouts, named workspaces, dimming inactive windows, one key map, one
  switcher, the dashboard with clocks, attention, idle and theme.
- **The same key map and layouts,** so a hand moving between Linux,
  chromeOS, a Mac and a PC doesn't notice.
- **No admin rights to run,** and nothing a managed work PC would refuse.
  Installing may ask once; running never does.
- **Nothing changes until it's asked for.** tide decides window management
  and its own keys. Windows' own settings stay as they were until turned on.
- **Native, or a widely used free tool.**
- **Say what can't be done** (§7), and **never degrade silently**: a
  missing helper or permission shows as a notification and in `tide doctor`.

## 2. What Windows allows

### 2.1 Moving other apps' windows

- **What it does:** `SetWindowPos` moves, resizes and restacks any top-level
  window of the same or lower integrity level
  ([SetWindowPos](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowpos)).
  No permission prompt, unlike macOS's Accessibility grant.
- **Hearing about windows:** `SetWinEventHook` reports windows being shown,
  hidden, focused, moved, minimized and destroyed, from every process,
  without injecting a DLL when called with `WINEVENT_OUTOFCONTEXT`
  ([SetWinEventHook](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwineventhook)).
- **Window IDs:** every window has an `HWND`, public and stable for its
  life. Nothing like macOS's private `_AXUIElementGetWindow` is needed.
- **Limits:**
  - **Elevated windows.** User Interface Privilege Isolation stops a normal
    process moving, or sending keys to, an app running as administrator.
    tide would have to run elevated itself, or as a signed `uiAccess` app
    installed under Program Files *(unverified as the only ways)*.
  - **Invisible borders.** On Windows 10 and 11 a window's rectangle includes
    an invisible resize border. The visible frame comes from
    `DWMWA_EXTENDED_FRAME_BOUNDS`, and tiles have to correct for the
    difference
    ([DWMWINDOWATTRIBUTE](https://learn.microsoft.com/en-us/windows/win32/api/dwmapi/ne-dwmapi-dwmwindowattribute)).
  - Apps can refuse a size, as on a Mac: a minimum size, or a terminal's
    grid.
  - A hung app's window answers late, but `SetWindowPos` with
    `SWP_ASYNCWINDOWPOS` doesn't wait for it, so one app needn't stall a
    retile.

### 2.2 Global keys

- **`RegisterHotKey`** binds a key and modifiers with no permission. It
  fails if another app holds the combination, and Microsoft reserves
  combinations with the Windows key for the system
  ([RegisterHotKey](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-registerhotkey)).
- **A low-level keyboard hook** (`WH_KEYBOARD_LL`) sees every key before apps
  do, with no permission, and can swallow it
  ([LowLevelKeyboardProc](https://learn.microsoft.com/en-us/windows/win32/winmsg/lowlevelkeyboardproc)).
  That allows tide for Linux's whole Super map, the lone Super tap, and
  held-modifier tricks. AutoHotkey works this way.
  - Windows removes a hook that answers too slowly, so the hook's thread
    must do nothing else.
  - Ctrl+Alt+Del and Win+L stay the system's.
  - A lone Win press opens Start. A hook that takes the tap has to send a
    harmless key between the press and release, as AutoHotkey's
    `A_MenuMaskKey` does
    ([AutoHotkey](https://www.autohotkey.com/docs/v2/lib/A_MenuMaskKey.htm)).
  - Keys typed into an elevated window aren't seen (§2.1) *(unverified)*.
- **So Super is the Windows key,** as on Linux, with no Karabiner-style
  helper. tide for macOS needs one because a Mac keyboard has no spare
  modifier.

### 2.3 Virtual desktops: a public API that can't switch

- **Public:** `IVirtualDesktopManager` can say whether a window is on the
  current desktop and which desktop it's on, and move a window, but only one
  of the calling process's own
  ([IVirtualDesktopManager](https://learn.microsoft.com/en-us/windows/win32/api/shobjidl_core/nn-shobjidl_core-ivirtualdesktopmanager);
  [Stack Overflow](https://kiwix.lema.org/stackoverflow.com_en_all_2023-11/questions/38633392/move-window-of-other-process-to-another-virtual-desktop)
  reports `E_ACCESSDENIED` for another's).
- **Switching** and moving another app's window need
  `IVirtualDesktopManagerInternal`, which is undocumented, and whose
  interface IDs change between Windows builds.
- **So tide emulates workspaces,** as the Windows tiling managers do. Their
  two ways:
  - **Cloaking:** the window stays where it is but isn't drawn. komorebi and
    GlazeWM both default to it, through the undocumented `SetCloak`
    ([komorebi](https://lgug2z.github.io/komorebi/cli/window-hiding-behaviour.html),
    [GlazeWM 3.6](https://newreleases.io/project/github/glzr-io/glazewm/release/v3.6.0)).
    Switching is instant, but every workspace's windows stay on the taskbar.
  - **Hiding or minimizing,** both public. Hiding takes a window off the
    taskbar too, but some apps misbehave while hidden; minimizing animates.
  - Moving windows off screen, as tide for macOS parks them, also works,
    since Windows doesn't keep a window on screen *(unverified on Windows 11)*.
- tide's choice is §6's first question.

### 2.4 Drawing over the desktop

- **Own windows at any level:** a layered, click-through, topmost window
  draws a bar, a dashboard or an overlay.
- **A bar:** an appbar reserves an edge of a display, so maximized windows
  stop short of it
  ([Application desktop toolbars](https://learn.microsoft.com/en-us/windows/win32/shell/application-desktop-toolbars)).
  The taskbar can hide itself automatically, but not be removed.
- **Dimming exactly the inactive windows:** unlike on a Mac, tide can place
  its own window directly below another app's in the stacking order, with
  `SetWindowPos`'s `hWndInsertAfter`. A dark layer just under the focused
  window dims everything else, desktop included. Dimming only the other
  windows, not the desktop, needs a layer per window *(unverified cost)*.
- **Borders and corners:** Windows 11 has `DWMWA_BORDER_COLOR` and
  `DWMWA_WINDOW_CORNER_PREFERENCE`. Whether they apply to another
  process's windows is *(unverified)*.
- **Title bars:** clearing `WS_CAPTION` removes a classic app's title bar.
  Apps that draw their own, such as Chrome, VS Code and Windows Terminal,
  keep theirs *(unverified per app)*.
- **WebView2,** part of Windows 11, shows web pages in tide's own windows
  ([WebView2](https://learn.microsoft.com/en-us/microsoft-edge/webview2/)).
  It can run tide for Chrome's dashboard and launcher page, as tide for
  macOS runs them in `hs.webview`.

### 2.5 Reading what other apps are doing

- **Windows and titles:** `EnumWindows` and `GetWindowText`, with no
  permission.
- **Attention:** an app calling `FlashWindowEx` flashes its taskbar button,
  and a shell hook window hears `HSHELL_FLASH` for it
  ([RegisterShellHookWindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-registershellhookwindow)).
  So, unlike a Mac, tide hears each window's attention request.
- **Notifications:** `UserNotificationListener` reads other apps'
  notifications, once the user allows it, but only from an app with
  package identity
  ([Notification listener](https://learn.microsoft.com/en-us/windows/apps/design/shell/tiles-and-notifications/notification-listener)).
- **Browser tabs:** no AppleScript. UI Automation can read the titles in
  Chrome's tab strip, but not each tab's address *(unverified)*.
  tide for Chrome's extension could report them instead.

### 2.6 Focus

- **Focus follows mouse is built in:** `SPI_SETACTIVEWINDOWTRACKING`,
  with `SPI_SETACTIVEWNDTRKZORDER` deciding whether focusing raises
  ([SystemParametersInfo](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfow)).
  Focus without raising, which a Mac can't do, is a setting here.
- **Taking focus is restricted:** `SetForegroundWindow` works only for a
  process that received the last input, or a few other cases
  ([SetForegroundWindow](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setforegroundwindow)).
  tide, which owns the keys, receives the input, so its own switches work.
  The same rule stops other apps stealing focus, tide for Linux's §14.

### 2.7 What Windows does natively

- **Snap layouts** arrange windows by hand
  ([Snap your windows](https://support.microsoft.com/en-us/windows/snap-your-windows-885a9b1e-a983-a3b1-16cd-c531795e6241)).
  No API sets a dwm-style layout through them *(unverified)*.
- **Win+V** is clipboard history
  ([clipboard](https://support.microsoft.com/en-us/windows/clipboard-in-windows-c436501e-985d-1c8d-97ea-fe46ddf338c6)).
  **PowerToys Run** and its successor, **Command Palette**, are launchers
  ([PowerToys](https://learn.microsoft.com/en-us/windows/powertoys/)).
- **Do not disturb**, **night light** and **power plans** for idle are
  Windows settings *(unverified for what each exposes to other apps)*.
- **Light and dark mode** are the `AppsUseLightTheme` and
  `SystemUsesLightTheme` values under
  `HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize`
  *(unverified that writing them switches running apps without a sign-out)*.

## 3. Ways to build it

The goal is tide's user experience: the same keys, layouts, workspaces,
switcher, bar and dashboard. The architecture can differ from tide for Linux
and tide for macOS.

| | A. Configure existing tools | B. AutoHotkey script | C. A native tide app, in Go | D. A fork of GlazeWM | E. A native tide app, in C# |
|---|---|---|---|---|---|
| What | GlazeWM, Zebar, PowerToys Command Palette and an AutoHotkey key map, set up by an installer | One AutoHotkey v2 script, as tide for macOS is one Hammerspoon config | Window manager, keys, bar and dim in Win32; switcher and dashboard as web pages in WebView2 | GlazeWM's window manager (Rust), with tide's layouts, workspaces and keys added, as tidewm forks dwl; bar and dashboard as [Zebar](https://github.com/glzr-io/zebar) widgets, which are web pages | Window manager in Win32 through .NET; bar, switcher and dashboard in WinUI 3 or WPF |
| tide's layouts | **No**: GlazeWM tiles as i3 does, by hand | Rewritten in AutoHotkey | tide for Linux's own `geometry.lua`, run by Lua 5.4 built in | Added to GlazeWM's tree model, the fork's biggest change | Ported, or Lua 5.4 through a binding |
| Named workspaces, "in" first | Partial: GlazeWM's are named, not made and removed as tide's are | tide's own | tide's own | tide's own, on GlazeWM's | tide's own |
| Switcher and dashboard | Command Palette: its own look and tabs | AutoHotkey's GUI, or WebView2 through a community library | **tide for Chrome's pages**, as tide for macOS uses them | **tide for Chrome's pages**, in Zebar | Native, redrawn to match |
| Fit to tide's experience | Low | Medium | High | High | High |
| Cost to build | Days | Days to a first version | Weeks | Weeks, then merging upstream | Weeks |
| Cost to run | Free *(GPL-3.0, unverified)*; komorebi in GlazeWM's place does dwm layouts but **needs a paid license at work** ([licensing](https://komorebi.lgug2z.com/about/licensing/)) | Free (GPL-2.0) | Free; signing about $10 a month, or none (§5.1) | Free; the fork stays GPL-3.0 | Free; signing optional (§5.1) |
| Shares code with tide | None | None | Lua layouts, clocks, launcher page; Go like tide's `cmd/` | Launcher page and clocks | None |
| Tests here | None | Little: AutoHotkey runs only on Windows | Pure parts on Linux; the rest on CI's Windows runners | Rust's on Linux; the rest on Windows runners | Mostly on Windows runners |
| Breakage risk | Theirs | AutoHotkey's | Ours | Ours plus upstream's churn | Ours |

**Recommendation: C, a native tide app in Go.**

- It gives the whole experience, and shares the most with the other tides:
  `geometry.lua` and tide for Chrome's pages, as tide for macOS does, so
  `make check-tide` keeps all four desktops' layouts, clocks and launcher in
  step.
- tide for Linux already has Go (`cmd/`), and Go builds Windows programs
  from Linux, with Win32 through `golang.org/x/sys/windows`. Lua 5.4 needs
  cgo and a C compiler, MinGW-w64.
- D is the close second: GlazeWM already solves cloaking, displays and
  focus, but tide's layouts and workspaces would replace its core, and
  every upstream release would need merging.
- B is the fastest start, but can't share the Lua, and would end where C
  starts. A is a weekend, but isn't tide.
- Unlike tide for macOS, it can be tested on real Windows in CI: GitHub's
  Windows runners have a desktop session *(unverified for window tests)*.

## 4. Feature map

**yes** = Windows meets the tide for Linux or tide for Chrome behavior;
**partial** = close, with the gap named; **no** = not reachable.

| tide for Linux / tide for Chrome | On Windows | Status |
|---|---|---|
| Tile, three-column, two columns + stack, monocle | `geometry.lua` over `SetWindowPos` (§2.1) | **yes**, with apps' own size limits and invisible borders corrected |
| Named workspaces, "in" first, spanning every display | emulated (§2.3) | **partial**: cloaked windows stay on the taskbar; hidden ones don't but some apps object |
| New window on an unasked workspace asks for a name | tide's prompt | **yes** |
| Dimming inactive windows | a layer just under the focused window (§2.4) | **yes**, dimming the desktop too |
| No borders, no gaps | gaps 0; classic title bars removed | **partial**: apps that draw their own keep them |
| Maximize, full screen, restore | tide's own | **yes** |
| Dialogs float, centered | owner, style and fixed size | **yes** *(unverified per app)* |
| Super+drag moves, Super+right-drag resizes | a low-level mouse hook | **yes** |
| Focus follows mouse, without raising | Windows' own setting (§2.6) | **yes** |
| Nothing steals focus | Windows' foreground rules (§2.6) | **yes** *(unverified against every app)* |
| Every tide for Linux key, and the Super tap | a low-level keyboard hook (§2.2) | **yes**, but not over elevated windows |
| One switcher with apps, workspaces, windows, attention and actions | tide for Chrome's launcher page in WebView2 | **yes** |
| Browser tabs in the Windows tab | UI Automation, or tide for Chrome's extension | **partial** |
| Attention, newest first | `HSHELL_FLASH`, plus notifications (§2.5) | **yes**; notifications need package identity |
| Full-width bar with workspaces, clocks and status | an appbar (§2.4) | **partial**: the taskbar can hide but stays |
| Clipboard history and characters on Super+V | `AddClipboardFormatListener` | **yes** |
| Notifications, history, Do not disturb | Windows' own | **yes**, native |
| Idle: dim, lock, off, suspend | tide's timeline, with power settings beside it | **partial**: Windows' own timers still run |
| Lock = login, hostname shown | Windows' lock screen | **no** for a custom one |
| Light and dark by time | tide writes the theme registry values | **yes** |
| US Dvorak, key repeat, mouse and touchpad | Windows' settings | **yes** |
| Settings pages and `tide doctor` | tide's own, reading `~/.config/tide` | **yes** |

## 5. Signing, elevated windows, work PCs and distribution

This section compares B (AutoHotkey) and C (a native Go app), the two
options still open.

### 5.1 Signing

- **What an unsigned program meets:**
  - **SmartScreen** warns on a downloaded file until it has built a
    reputation. Reputation is per file hash, so every release starts again
    ([SmartScreen reputation](https://learn.microsoft.com/windows/apps/package-and-deploy/smartscreen-reputation)).
    The warning has a "Run anyway" link; it doesn't block.
  - **Smart App Control,** on new Windows 11 installs, blocks unsigned
    files without a good reputation outright, downloaded or not
    ([same page](https://learn.microsoft.com/windows/apps/package-and-deploy/smartscreen-reputation)).
    It can be turned off, but not back on without reinstalling Windows
    *(unverified on current builds)*.
- **What signing buys:** signed files build reputation under the
  certificate, not each hash, so a new release inherits it. Since 2024 an
  EV certificate no longer skips the wait
  ([AppRep best practices](https://textslashplain.com/2024/11/15/best-practices-for-smartscreen-apprep/)).
- **What it costs:**
  - **Azure Artifact Signing** (formerly Trusted Signing): about $9.99 a
    month for 5,000 signatures. Individuals qualify only in the US and
    Canada, on a paid Azure subscription
    ([quickstart](https://learn.microsoft.com/en-us/azure/artifact-signing/quickstart),
    [DevClass](https://www.devclass.com/security/2026/01/14/code-signing-windows-apps-may-be-easier-and-more-secure-with-new-azure-artifact-service/4079554)).
    It signs from CI with no key to keep.
  - **An OV certificate** from a certificate authority: about $150 to $300
    a year, on a hardware key
    ([Microsoft](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options)).
  - **SignPath Foundation, free for open source:** it signs releases built
    by CI from a public repository under an OSI-approved license, such as
    tide's Apache 2.0. The publisher shown is "SignPath Foundation", not
    tide, and the project publishes a signing policy crediting it
    ([SignPath Foundation](https://signpath.org/),
    [terms](https://signpath.org/terms)). Whether a new project qualifies
    straight away is *(unverified)*.
    - **One manual step per release:** CI builds and submits each release
      unattended, with SignPath checking it came from the repository's
      workflow, but the terms say "every release needs manual approval
      for signing": a maintainer approves it in SignPath before the signed
      files come back
      ([terms](https://signpath.org/terms),
      [origin verification](https://docs.signpath.io/origin-verification)).
  - **The Microsoft Store** signs for you, and Store apps never see
    SmartScreen's warning. It needs an MSIX package (§5.4). Registering as
    an individual developer has been free since 2025
    ([Windows Developer Blog](https://blogs.windows.com/windowsdeveloper/2025/09/10/free-developer-registration-for-individual-developers-on-microsoft-store/)).
- **B:** AutoHotkey itself is signed. A `.ahk` script isn't an executable,
  so SmartScreen and Smart App Control judge AutoHotkey, not tide
  *(unverified for Smart App Control's script checks)*. A script compiled
  to an `.exe` is unsigned again, and antivirus often flags compiled
  AutoHotkey with a keyboard hook *(unverified rate)*.
- **C:** `tide.exe` and its MSI need signing to pass Smart App Control.
  Without it, it's "Run anyway" once per release where Smart App Control
  is off.

### 5.2 Elevated windows

- **The problem:** User Interface Privilege Isolation stops a normal
  program moving, focusing, or sending keys to the window of one running
  as administrator: an installer, Task Manager, an admin terminal (§2.1).
  tide's keys also go dead while such a window has focus *(unverified for
  low-level hooks)*.
- **The ways round it:**
  - **Leave them alone.** tide floats them where they open, and says so in
    `tide doctor`. Free; most days nothing is elevated.
  - **Run tide elevated.** Everything works, but every app tide launches
    is elevated too unless it de-elevates them, and Windows asks at every
    sign-in unless started by a scheduled task. Not recommended.
  - **`uiAccess`:** the accessibility exemption. The program's manifest
    says `uiAccess="true"`, it's signed by a certificate the machine
    trusts, and it's installed under Program Files or System32, which
    only admins can write
    ([UIAccess secure locations](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/security-policy-settings/user-account-control-only-elevate-uiaccess-applications-that-are-installed-in-secure-locations),
    [UI Automation security](https://learn.microsoft.com/windows/win32/winauto/uiauto-securityoverview)).
    Microsoft says only assistive technology should use it.
- **B:** AutoHotkey's installer offers "Run with UI Access": it installs
  under Program Files and signs its `UIA.exe` with a self-signed
  certificate it adds to the machine's trusted roots
  ([AutoHotkey's setup script](https://cdn.jsdelivr.net/npm/w-mousekey@1.0.16/AutoHotkey_2.0.19/UX/ui-setup.ahk)).
  So B gets elevated windows free, with an admin install.
- **C:** the same trick works for `tide.exe`: a per-machine MSI, admin once,
  installs it under Program Files and either carries a real signature
  (§5.1) or makes a local certificate as AutoHotkey does. A work PC's
  policy may forbid adding a root certificate *(unverified)*.

### 5.3 Work PCs

- **What IT controls:**
  - **AppLocker:** its default rules let everyone run programs from
    Program Files and Windows, and only admins anything else
    ([AppLocker default rules](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/understanding-applocker-default-rules)).
    A per-user install in `%LOCALAPPDATA%` is then blocked.
  - **App Control for Business** (formerly WDAC) allows by signer or
    hash; an unsigned program needs IT to add its hash, every release.
  - **Intune and Group Policy** install MSIs per machine, silently.
- **So the work path is a signed, per-machine MSI that IT deploys or
  allows.** It lands in Program Files, passes AppLocker's defaults, gets
  `uiAccess`, and IT can allow it by its signer once.
- **B on a work PC:** needs AutoHotkey allowed. Many companies block it by
  name, since it's a common malware host *(unverified how common)*. Where
  it's allowed, tide is a script in the user's folder, which AppLocker's
  script rules don't cover *(unverified)*.
- **C on a work PC:** needs the MSI deployed or allowed. Asking IT to
  approve one signed app is the usual request; unsigned is a hash to
  re-approve each release.
- **Keyboard hooks** are what keyloggers use, so endpoint security may
  flag either B or C *(unverified)*. Signing helps C here; B relies on
  AutoHotkey's standing.

### 5.4 Distribution

| | B: AutoHotkey | C: native Go |
|---|---|---|
| Release artifact | A zip of scripts; or an MSI that installs AutoHotkey and the scripts | An x64 MSI, and an arm64 one when wanted |
| Install without admin | Yes, if AutoHotkey is installed or portable | Yes, per-user MSI; no `uiAccess` |
| Install with admin, once | Yes, with UI Access | Yes, per-machine MSI with `uiAccess` |
| Package managers | winget and Scoop already carry AutoHotkey; tide's scripts are a `git clone` or zip | winget, Scoop, and the Store as MSIX *(unverified that MSIX allows `uiAccess`)* |
| Updates | `git pull`, or tide updates its own scripts | A new MSI; winget upgrades it |
| Package identity, for reading notifications (§2.5) | No | Only as MSIX, or an MSI with a signed sparse package |
| Build | Nothing to build | `go build` with MinGW-w64, and WiX on a Windows runner, or msitools' `wixl` on Linux *(unverified)* |

- **Releases:** a release workflow builds and attaches the artifacts on
  each push to main, as tidewm's tags each push.
- **Failure modes:** undocumented calls, if tide uses `SetCloak`, can break
  with a Windows update. Nothing touches the network: nothing to go down
  and no running cost beyond signing.

### 5.5 Building and testing on GitHub

Both options build, test and release on GitHub Actions. Hosted runners are
free for a public repository; a private one gets a monthly allowance, with
Windows minutes counted double *(unverified current terms)*. Releases and
their artifacts are free.

| | B: AutoHotkey | C: native Go |
|---|---|---|
| On Linux runners | Nothing runs: AutoHotkey is Windows only. Packaging the zip | `gofmt`, `go vet`, the pure Go and Lua tests, `make check-tide`, a cross-compiled `tide.exe` with MinGW-w64, and an MSI with `wixl` *(unverified)* |
| On Windows runners | Install AutoHotkey, check every script loads *(unverified switch)*, and drive real windows | `go test` against real Win32 windows, WiX for the MSI |
| Signing | Not needed for scripts | SignPath, free: CI submits, a maintainer approves each release (§5.1) |
| Release | A zip on each GitHub release | x64 and arm64 MSIs on each GitHub release, and a winget manifest |

- **Windows runners are Windows Server,** not Windows 11: no Smart App
  Control, and a different shell. Whether tests there can open, move and
  focus windows on a desktop is *(unverified)*; the first workflow
  proves it, before any window code depends on it.
- **What CI can't cover:** a real Windows 11 desktop, Smart App Control,
  and a managed PC's policies. Those need a hand check on a real PC.

## 6. Open questions

1. **How to build it (§3).** Recommended: C, a native tide app in Go.
2. **How to hide a workspace (§2.3):** cloak, hide, minimize or park off
   screen. Suggested: hide, falling back to cloak per app, so the taskbar
   shows only the current workspace.
3. **The taskbar:** hide it automatically under tide's bar, or keep it and
   skip tide's bar.
4. **Elevated windows (§5.2):** leave them unmanaged, or install
   per-machine with `uiAccess`.
5. **Signing (§5.1).** **Decided (2026-11-18):** no paid signing; the
   Microsoft Store, as MSIX, stays an option for later. SignPath Foundation
   was chosen if CI could sign unattended, but its terms need a
   maintainer's approval for every release. Open: accept one approval per
   release, or ship unsigned. tidewin is public under Apache 2.0, as
   SignPath requires.
6. **First target (§5.3):** a personal PC, or a managed work PC.

## 7. What tide can't have

| tide for Linux has | Why Windows can't | Workaround |
|---|---|---|
| Real workspaces | Switching virtual desktops is undocumented (§2.3). | Emulate them. |
| No title bars | Apps that draw their own keep them (§2.4). | None for those. |
| Managing admin windows | User Interface Privilege Isolation (§2.1). | Run elevated, or a signed `uiAccess` build. |
| Its own lock screen and login | They're the system's. | None. |
| Win+L and Ctrl+Alt+Del | They're the system's (§2.2). | None. |
| Its own bar alone | The taskbar can hide, not go. | Hide it automatically. |
