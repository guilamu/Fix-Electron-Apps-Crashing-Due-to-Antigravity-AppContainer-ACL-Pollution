# Fix: Electron Apps Crashing Due to Antigravity AppContainer ACL Pollution

Electron apps started crashing for me on June 14th 2026. It took me a while to figure it out. Here's the whole deal.

> **Affected apps:** Any Electron-based app installed under `%LOCALAPPDATA%\Programs` — Signal, GitHub Desktop, Discord, Notion, OpenCode, VS Code, Spotify, etc.
>
> **Symptom:** Apps crash immediately on launch, or a blank window flashes and closes. The flag `--no-sandbox` or `--disable-gpu-sandbox` makes them work again.
>
> **Root cause:** Antigravity 2.0 injected rogue AppContainer SIDs into the ACL of `%LOCALAPPDATA%`, which propagate by inheritance to every app installed underneath, breaking Chromium's GPU sandbox initialization with error `0x80000003`.

---

## Background

Electron apps use a Chromium-based GPU sandboxing mechanism (AppContainer on Windows). When Chromium tries to initialize its GPU service process, it reads the ACL of the directory the app lives in. If unexpected AppContainer SIDs are present, the sandbox setup fails and the process crashes.

Antigravity 2.0's installer added 6 AppContainer SIDs (`S-1-15-2-*`) directly on `%LOCALAPPDATA%`. Because Windows propagates permissions by inheritance, **every app installed under that folder** inherited these broken entries. This explains:

- Why `--no-sandbox` works (it bypasses the broken sandbox entirely)
- Why apps on non-standard paths sometimes worked (different inheritance chain)
- Why reinstalling temporarily helped (installer rebuilds folder permissions)
- Why disabling Defender, HVCI, or uninstalling antivirus had no effect

---

## Diagnosis

Open **PowerShell as Administrator** and run:

```powershell
icacls "$env:LOCALAPPDATA\Programs"
```

If you see lines starting with `S-1-15-2-` that include `(I)` (meaning "inherited"), the SIDs are set on the parent folder. Run:

```powershell
icacls "$env:LOCALAPPDATA"
```

If the same `S-1-15-2-*` entries appear **without** `(I)`, they are set directly on `%LOCALAPPDATA%` and that is your source.

**Example of a polluted output:**

```
C:\Users\<username>\AppData\Local
    S-1-15-2-659879304-689816817-...(F)
    S-1-15-2-659879304-689816817-...(OI)(CI)(IO)(F)
    S-1-15-2-496641071-1885145544-...(F)
    ... (6 SIDs total, each appearing twice)
    AUTORITE NT\Système:(OI)(CI)(F)
    BUILTIN\Administrateurs:(OI)(CI)(F)
    DESKTOP-<machine>\<username>:(OI)(CI)(F)
```

A healthy `%LOCALAPPDATA%` should only show `NT AUTHORITY\SYSTEM`, `BUILTIN\Administrators`, and your own user account — **no `S-1-15-2-*` entries**.

---

## Fix

### Step 1 — Back up your ACLs (do not skip)

```powershell
icacls "$env:LOCALAPPDATA" /save "$env:USERPROFILE\Desktop\acl-backup-localappdata.bin"
icacls "$env:LOCALAPPDATA\Programs" /save "$env:USERPROFILE\Desktop\acl-backup-programs.bin" /t
```

Two `.bin` files will be saved to your Desktop. Keep them until everything is confirmed working.

### Step 2 — Identify the exact SIDs on your machine

```powershell
icacls "$env:LOCALAPPDATA"
```

Copy every `S-1-15-2-*` SID you see. They will differ between machines. In the documented case there were 6 unique SIDs, each appearing twice (once for the folder, once with `(OI)(CI)(IO)` for children).

### Step 3 — Remove each Antigravity SID from `%LOCALAPPDATA%`

Run one command per SID (replace the values with those found in Step 2):

```powershell
icacls "$env:LOCALAPPDATA" /remove "*S-1-15-2-659879304-689816817-486076376-148918669-2289132224-4090998324-235152480"
icacls "$env:LOCALAPPDATA" /remove "*S-1-15-2-496641071-1885145544-693792816-488361937-1021200446-875339651-1251486819"
icacls "$env:LOCALAPPDATA" /remove "*S-1-15-2-729811823-2911358309-1093304830-668587673-1659552153-409309229-1111218266"
icacls "$env:LOCALAPPDATA" /remove "*S-1-15-2-3611496401-4014087101-3898385913-1957482850-289379856-1143870915-3498495650"
icacls "$env:LOCALAPPDATA" /remove "*S-1-15-2-1361304429-1054449971-3077055425-2476757490-2967482325-3120757492-845384970"
icacls "$env:LOCALAPPDATA" /remove "*S-1-15-2-3555064862-2733552438-2480326384-816773790-4245940572-2352748177-2830482964"
```

> The `*` prefix tells `icacls` to treat the value as a SID rather than a username, which is required here since these SIDs have no mapped display name.

### Step 4 — Reset inherited permissions on `Programs`

```powershell
icacls "$env:LOCALAPPDATA\Programs" /q /c /t /reset
```

This resets the ACL of `Programs` and all its subdirectories to inherit cleanly from the now-fixed parent. It touches **permissions only** — no files are moved, deleted, or modified. Expect it to take 1–2 minutes if many apps are installed (it may process 100k+ entries).

### Step 5 — Verify

```powershell
icacls "$env:LOCALAPPDATA"
```

The output should now only contain:

```
NT AUTHORITY\SYSTEM:(OI)(CI)(F)
BUILTIN\Administrators:(OI)(CI)(F)
<MACHINE>\<username>:(OI)(CI)(F)
```

No `S-1-15-2-*` lines should remain.

### Step 6 — Test your apps

Launch Signal, GitHub Desktop, OpenCode, or any other previously broken Electron app **without** any extra flags. They should start normally.

---

## Rollback (if something goes wrong)

```powershell
icacls "C:\Users\<your-username>\AppData" /restore "$env:USERPROFILE\Desktop\acl-backup-localappdata.bin"
icacls "$env:LOCALAPPDATA" /restore "$env:USERPROFILE\Desktop\acl-backup-programs.bin"
```

Replace `<your-username>` with your actual Windows username.

---

## References

- [Electron Process Sandboxing documentation](https://www.electronjs.org/docs/latest/tutorial/sandbox)
- [Similar reported case: stale AppContainer ACL on `%LOCALAPPDATA%\Programs` breaking Electron apps (Reddit r/codex)](https://reddit.com/r/codex)
- [icacls documentation — Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls)
