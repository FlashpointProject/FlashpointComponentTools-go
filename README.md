# Flashpoint Component Tools

Tools for installing and managing a copy of [Flashpoint Archive](https://flashpointarchive.org/) through its component system.

- `FlashpointInstaller.exe` installs Flashpoint into a new folder, with the components you pick, and creates shortcuts.
- `FlashpointManager.exe` adds, removes, updates and repairs components of an existing copy, or uninstalls it. The [Flashpoint Launcher](https://github.com/FlashpointProject/launcher) opens it.
- `fpm.exe` does the same from the command line (`fpm help`). The launcher reads `fpm list verbose` at startup.

Windows 10 or later, 64-bit. The manager and `fpm.exe` ship together in the `core-manager` component.

## Build

Go 1.27.3 or later.

```sh
scripts/build.sh            # dist/windows-amd64/: the three exes and core-manager.zip
go test ./...
```

The GUI uses [Gio](https://gioui.org) and builds for Windows without cgo; on macOS and Linux it needs cgo (development only).

## Layout

```
cmd/fpm, cmd/manager, cmd/installer   the three programs
internal/engine                       manifest -> plan -> download -> transactional apply
internal/gui                          the window (manager and installer mode)
internal/install                      installer and uninstall parts (destination, shortcuts, VC++ runtime)
testdata                              real component lists and records written by the C# tools
```

## Launcher contract

The launcher and existing installs depend on these; keep them byte-exact (golden tests in `cmd/fpm`).

- `Manager\fpm.exe list verbose`, run from `Manager\`: one line per component, `<*|!| > <id> (<title>)`, CRLF, stdout only, exit 0. `*` = installed and up to date, `!` = outdated. A failure hides the launcher's Manager button, so offline it answers from `Components\components.bak`.
- `FlashpointManager.exe /notemp /download "id1 id2"`: the IDs arrive as one argument. The launcher waits and treats exit 0 as "all installed". It finds support packs by component title, so titles matter.
- `FlashpointManager.exe /update /launcher`: opens Updates and starts `Launcher\Flashpoint.exe` on exit. The launcher quits ~2.5 s later and kills its job, so the manager relaunches itself outside the job first.
- `Components\<id>`: line 1 `<crc32> <install-size> [deps…]`, then Flashpoint-relative paths with backslashes, CRLF. The file existing means installed. Other tools read these.
- `Manager\fpm.cfg`: line 1 the Flashpoint folder (`..\` or absolute), line 2 the `components.xml` URL.
- `components.xml`: the C# tools crash on any new element or comment inside `<list>`; only new attributes are safe. `hash` is the CRC32 of the whole zip; the zip URL is `<list url>` + id + `.zip`.
- Rules: IDs starting with `core` are required, `extra*` are off by default in the installer, `core-database` never counts as outdated, install-size 0 means no zip.
- `core-manager` (path `Manager`) ships both `FlashpointManager.exe` and `fpm.exe`; the old C# manager installs it like any other update.
- The website's download button links to `releases/latest/download/FlashpointInstaller.exe`, so that asset name is fixed.
