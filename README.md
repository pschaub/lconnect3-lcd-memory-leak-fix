# L-Connect 3 memory leak with LCD fans, and how to fix it

If you use Lian Li fans with built-in LCD screens together with L-Connect 3, the program's background service
("L-Connect Service") slowly uses more and more memory (RAM) for as long as the fan screens play videos. With four
LCD fans that is about 126 MB per day, and it keeps growing until you restart the service or the PC. On the test PC it
had reached more than 1 GB.

This repository explains why that happens and offers a small script that fixes it on your own PC. The script changes
one spot in one L-Connect file, keeps a backup of the original, and can undo the change with a single command.

- **Affected version:** L-Connect 3 **2.1.29**, the latest version in the
  [official changelog](https://lian-li.com/l-connect3/l3-changelog/) when this was tested (October 2026).
- **Download the fix:** [latest release](../../releases/latest). Instructions are under [Install](#install).
- **Not affected:** fan speed, pump, RGB lighting and sensors work normally and are not changed by the fix.

## Background

L-Connect 3 installs a Windows service, `LConnectService`, that controls fans, pump, lighting and sensors, and sends
pictures and videos to the fan and cooler screens over USB. The changelog mentions a memory fix in version 2.1.13, but
the leak described here is still present in 2.1.29.

The test PC had four SL-INF FLEX LCD fans playing video themes, an AIO cooler with an OLED screen and the wireless fan
controller.

## Evidence

The measurements come from snapshots of the running service's memory, taken with
[`tools/Measure-LConnect.ps1`](tools/Measure-LConnect.ps1), plus the memory figures Windows reports for the process.

**After a long uptime**, the service's memory held more than **19 million queued entries, using 228 MiB**, in four
internal lists, one per LCD fan. Every entry pointed to the same small data buffer. This memory was still in use by
the program, not just waiting to be cleaned up.

**Controlled before/after comparison.** The same PC was measured twice with the same method: once with the original
file and once with the fixed file. Each run restarted the service, waited one minute, and then took 15 samples one minute
apart.

| time since start | original: entries | original: memory used by them | fixed: entries | fixed: memory used by them |
|---:|---:|---:|---:|---:|
| 2 min | 9,502 | 123 KB | 4,093 | 55 KB |
| 6 min | 39,062 | 492 KB | 4,093 | 54 KB |
| 10 min | 68,620 | 862 KB | 4,093 | 55 KB |
| 14 min | 98,189 | 1,231 KB | 4,093 | 55 KB |
| 16 min | 112,982 | 1,416 KB | 4,093 | 55 KB |

- **Original:** about 30 new entries per second per fan, which adds up to **about 5 MiB per hour, or 126 MiB per day**
  with four fans. The growth is steady and never levels off.
- **Fixed:** the number stays flat at about 1,000 per fan (≈55 KB in total) from the start.
- Over just 15 minutes, the service's total memory moves up and down by tens of MB for normal reasons, which hides the
  leak. The leak shows up over hours and days, as in the long-uptime numbers above.

Raw samples: [`data/before.csv`](data/before.csv), [`data/after.csv`](data/after.csv).

## Cause

Whenever L-Connect sends a video frame to an LCD fan, the fan answers "frame received". L-Connect puts every one of
these answers into a list, but during video playback nothing ever reads or removes them. At 30 frames per second this
list grows by about 30 entries per second per fan, forever.

Technical details:

- The code is in `lianli.slv3.dll`, class `slv3.WinUsb`, method `OnRxEndPointData`. It is the handler that receives
  data from the fan over USB. Every incoming packet whose first byte is not `0x0D`, `0x0F` or `0x28` is appended to
  `private ConcurrentQueue<byte[]> rxList` (decompiled excerpt):

  ```csharp
  if (e.Buffer != null && e.Buffer[0] != 13 && e.Buffer[0] != 15 && e.Buffer[0] != 40
      && (!needSyncL3 || (e.Buffer[0] != 103 && e.Buffer[0] != 101)))
      rxList.Enqueue(e.Buffer);
  ```

- The queue has no size limit. Only `Read()` and `Read(CmdType)` take entries out, and those run only when the
  software sends a command and waits for the reply.
- `0x65` (101) is `CmdType.PushJpg`, the screen's acknowledgement of a pushed image frame. Video frames are sent with
  `isRead: false`, so their acknowledgements are never consumed. Every queued packet on the test PC started with
  `0x65`, and the measured ~30.6 entries per second per fan matches the 30 fps videos.
- `0x65` and `0x67` are filtered out only while `needSyncL3` is true. The service sets that flag only for screen themes
  that contain a GIF or PNG animation. Turning the flag on is not a fix, because it also changes how frames are sent.
- LibUsbDotNet reuses `e.Buffer`, so all entries point to the same array. The memory goes into the queue's own
  structure, about 12.5 bytes per entry.

## Who is affected

The leak only happens while L-Connect keeps sending pictures to an LCD fan screen.

| Situation | Leaks? | Basis |
|---|---|---|
| LCD fan screen playing a **video** (built-in or your own) | **Yes**, about 30 entries per second per fan | Measured |
| Other screen content that is sent over and over (e.g. live sensor layouts) | Probably, at the rate it is sent | From the code, not measured |
| LCD fan screen showing a **still image** | No lasting growth, only a few entries when the theme changes | From the code |
| LCD fan screen theme with a **GIF or PNG animation** | No, these answers are filtered out | From the code |
| Fan speed, pump, RGB lighting, sensors, wireless control | No | Measured |

| Device (USB ID) | Status |
|---|---|
| SL-INF FLEX LCD fans (`1CBE:A019`) | **Affected**, measured |
| TL FLEX LCD fans (`1CBE:A018`) | **Very likely affected**: they use exactly the same code. Not measured. |
| Older Lian Li LCD devices (`1CBE:0005`, `1CBE:0006`) | **Possibly affected**: same receiving code, but not tested |
| Wireless fan controller/receiver (`0416:8040`, `0416:8041`, PID `8051`) | Not affected (their lists stayed empty) |
| HydroShift II OLED/LCD cooler screens (`1CBE:A068`) | Not affected: they use a different part of L-Connect (`lcd207.dll`). The OLED was playing video during the measurement. |

The memory grows with *number of LCD fans × frame rate × uptime*. One fan playing 30 fps video adds about 1.3 MiB per
hour (about 32 MiB per day). Only a restart of the service clears it. The still-image and GIF rows are based on the
code; those themes were not switched on live during testing.

**Am I affected?** If you use L-Connect 3 2.1.29 with LCD fans playing videos, and the memory of "L-Connect Service" in
Task Manager keeps rising over days, you very likely are.

## The fix

The script adds a limit right before an entry is added: if a fan's list already has 1,024 entries, the oldest are
removed first.

```csharp
byte[] result;
while (rxList.Count >= 1024 && rxList.TryDequeue(out result)) { }
rxList.Enqueue(e.Buffer);
```

Nothing else in the file is changed. Because all entries point to the same reused buffer, removing old ones loses no
information. `Read(CmdType)` already skips entries that don't match, and file uploads clear the list themselves.
A proper fix from Lian Li would copy each received packet, match answers to requests, and ignore unrequested ones.

**Checked on the test PC after patching:** all USB devices connected, the screen themes loaded, the OLED video started,
and pump, fans and sensors reported normally. No new errors or crashes appeared, and the fixed service ran for more
than 3 hours without a restart.

## Install

1. Download `Patch-LConnectRxQueue.ps1` from the [latest release](../../releases/latest).
2. Open PowerShell **as administrator**: right-click the Start button, choose *Terminal (Admin)*, then go to your
   download folder, e.g. `cd $HOME\Downloads`.
3. Check that your L-Connect version is supported. It should print `Original`:
   ```powershell
   powershell -ExecutionPolicy Bypass -File .\Patch-LConnectRxQueue.ps1 -Action Check
   ```
4. Apply the fix:
   ```powershell
   powershell -ExecutionPolicy Bypass -File .\Patch-LConnectRxQueue.ps1 -Action Patch
   ```
5. Open L-Connect 3 again.

`-Action Patch` does the following:
1. Checks that the installed file is exactly the supported original.
2. Saves the original file to `%ProgramData%\LConnectRxQueueFix\lianli.slv3.dll.original` (never overwritten) and
   copies your L-Connect settings (`settings`, `device`, `profile`, `slv3\config`) next to it.
3. Builds the fixed file from the backup and checks it before installing anything.
4. Stops the L-Connect services and the L-Connect window, replaces the file and starts the services again. If anything
   fails along the way, it puts the original back automatically.

The script runs in Windows PowerShell 5.1, which ships with Windows. It downloads one helper library (Mono.Cecil 0.11.6)
from nuget.org and checks it against a fixed SHA256.

## Undo

```powershell
powershell -ExecutionPolicy Bypass -File .\Patch-LConnectRxQueue.ps1 -Action Restore
```

To undo by hand, stop both L-Connect services, copy `lianli.slv3.dll.original` over `lianli.slv3.dll`, then start the
services again. An L-Connect update replaces the file anyway, so run `-Action Check` after every update.

## Compatibility

| | |
|---|---|
| L-Connect 3 | 2.1.29 |
| File | `C:\Program Files\Lian-Li\L-Connect 3\lianli.slv3.dll` |
| Assembly | `lianli.slv3, Version=1.5.0.146` |
| SHA256 original | `1A1711717520DDFF9B789FCD2379C0636062CF199BB0AE38FBBBA2FE040C0317` |
| SHA256 fixed | `0C57AE4C55C94F8966D1F0C6BE81E8ECC303CAC536551590A13EA28B60DFDACB` |

The script refuses any other file version. It always produces the same fixed file, so that file's checksum is pinned
too. No Lian Li files are included here; the script changes your own installed copy.

## Repeating the measurement

You need PowerShell 7 (`pwsh`), run as administrator. The script downloads ClrMD 3.1.512801 from nuget.org and checks
its SHA256. It reads a snapshot of the service's memory, so the service keeps running.

```powershell
Restart-Service LConnectServiceWatcher, LConnectService; Start-Sleep 60
.\tools\Measure-LConnect.ps1 -OutFile before.csv -Samples 15 -IntervalSeconds 60
# apply the fix, wait 60 s, then:
.\tools\Measure-LConnect.ps1 -OutFile after.csv  -Samples 15 -IntervalSeconds 60
```

## Limitations

- Lian Li digitally signs the original file, and the fix removes that signature. Windows and L-Connect still load the
  file normally, but security software that only trusts signed files may flag it.
- This was tested on one PC. Other L-Connect versions are refused on purpose.
- The rest of the service's memory use (about 650 MB after a fresh start) is not caused by this bug. Other leaks are
  neither shown nor ruled out here.
- This is an unofficial fix, not affiliated with or endorsed by Lian Li. Use it at your own risk.

## Credits and license

The investigation, fix, measurements and this write-up were created with Claude Opus 5.5 (Anthropic).

Scripts and documentation are released under the [MIT License](LICENSE). L-Connect, `lianli.slv3.dll` and all related
names belong to Lian Li Industrial Co., Ltd.
