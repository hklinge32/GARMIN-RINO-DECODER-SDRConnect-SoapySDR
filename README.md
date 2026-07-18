# Garmin RINO Burst Decoder

A self-calibrating decoder for Garmin RINO position broadcasts sent over FRS/GMRS
frequencies. It listens to live IQ from an SDR (or a saved recording) and
automatically detects and decodes position packets in real time — no manual
tuning required for normal use.

Decoded positions, callsigns, altitude, and any received text messages or poll
requests are available through a browser dashboard, exported live to Google
Earth, and logged to disk for later review.

## What it does

- Listens continuously across your configured FRS/GMRS channels
- Automatically detects transmissions and decodes position, callsign,
  altitude, and icon
- Tracks known callsigns, received messages, and poll requests
- Exports live positions to KML for Google Earth
- Serves a live web dashboard (map, channel status, users, messages, polls,
  and adjustable sensitivity) with no setup beyond running one command

The decoder supports two independent SDR backends — pick whichever matches your hardware:

- **SDRconnect** — SDRplay's own capture application. Only works with SDRplay hardware (RSP1A, RSPdx, RSP1B, etc.), but requires no compilation on your part.
- **SoapySDR** — a hardware-abstraction layer supporting a wide range of SDRs from different vendors (RTL-SDR, HackRF, Airspy, SDRplay, LimeSDR, PlutoSDR, USRP, and more). Requires building a small bridge library once, but isn't tied to one vendor.

See [Choosing a backend](#choosing-a-backend-sdrconnect-vs-soapysdr) below for the tradeoffs, and [Supported SDRs via SoapySDR](#supported-sdrs-via-soapysdr) for hardware-specific notes.

## Choosing a backend: SDRconnect vs SoapySDR

Both backends feed the same capture pipeline — the choice only affects how IQ samples get from your hardware into the program. Set it in `settings.json`:

```json
"SdrBackend": "sdrconnect"   // or "soapy"
```

| | **SDRconnect** | **SoapySDR** |
|---|---|---|
| Hardware supported | SDRplay only (RSP1A, RSPdx, RSP1B, RSP1, RSP2, RSPduo) | RTL-SDR, HackRF, Airspy, SDRplay, LimeSDR, PlutoSDR, USRP, and anything else with a SoapySDR driver module |
| Setup effort | Install SDRplay's app, flip on its WebSocket API | Install SoapySDR + the driver module for your device, compile the included bridge library once |
| Connection to decoder | WebSocket to a GUI app that must stay running | Direct connection — no other app needs to be open |
| Best for | SDRplay owners who want the least setup | Everyone else, or anyone who wants one process instead of two |

If you have an SDRplay device and don't want to compile anything, use SDRconnect. Otherwise, use SoapySDR.

## Supported SDRs via SoapySDR

This program only ever tunes within the FRS/GMRS band (~462–467 MHz) at a fairly modest sample rate, so nearly any SoapySDR-supported device covers the tuning range you need. The real differences between devices show up in dynamic range, USB streaming stability, and how many simultaneous FRS/GMRS channels you can comfortably monitor in one capture window. `SoapyDeviceArgs` in `settings.json` must match the driver string for your device (find it with `SoapySDRUtil --find`, see [Troubleshooting](#troubleshooting)).

| Device family | `driver=` string | ADC | Typical max stable rate for this app | Notes |
|---|---|---|---|---|
| SDRplay RSP1A / RSPdx / RSP1B | `sdrplay` | 12–14 bit | 2 Msps+ | Best dynamic range of this list; what the default `settings.json` is tuned for. Requires the SDRplay API service running in the background (same one SDRconnect uses) even when using the Soapy backend. |
| RTL-SDR (R820T2/R828D dongles) | `rtlsdr` | 8 bit | ~2.0–2.4 Msps | Cheapest option and fine for testing, but the 8-bit ADC gives noticeably less dynamic range — weak/distant RINOs are more likely to sit under the noise floor. No fine-grained AGC; gain is a fixed step table. |
| HackRF One | `hackrf` | 8 bit | ~2–8 Msps | Half-duplex (RX only here, which is fine). More overflow-prone than the others at low `SoapyChunkSamples` — raise it if you see `[BRIDGE] Overflow` messages. |
| Airspy Mini / R2 | `airspy` | 12 bit | 3 / 6 / 10 Msps (fixed set) | Good dynamic range for the price; only supports a handful of fixed sample rates, so `SampleRate` in `settings.json` should be set to one Airspy actually offers. |
| LimeSDR (Mini / USB) | `lime` | 12 bit | 2 Msps+ | Overkill for this program's needs but works fine; SoapySDR + `SoapyLMS7` module setup is more involved than the others. |
| PlutoSDR | `plutosdr` (often with `,uri=ip:192.168.2.1`) | 12 bit | 1–2 Msps | Stock firmware's usable range starts around 325 MHz, comfortably below the 462–467 MHz band this program needs. |
| USRP (B200/B210 etc.) | `uhd` | 12–16 bit | Well beyond what this app needs | Requires UHD installed separately from SoapySDR itself (`SoapyUHD` module bridges the two). Most cost-effective devices on this list will perform just as well for RINO decoding specifically — a USRP is not necessary for this use case. |

A few things that apply regardless of device:

- **You don't need a high sample rate.** The capture pipeline decimates everything down internally before per-channel processing. Requesting more than a couple Msps from the hardware just spends CPU without improving decode quality — it only matters if you want more of the surrounding spectrum (e.g. to monitor more repeater inputs in one capture window without switching between bands).
- **CS16 is requested from every device.** If a device's native format is actually complex float32 (most SDR chips are), SoapySDR converts on the host side transparently — you don't need to do anything, but it's worth knowing where a small amount of CPU overhead comes from on such devices.
- **Overflow messages** (`[BRIDGE] Overflow ×N`) mean the USB link or host processing isn't keeping up with the requested sample rate. Lower `SampleRate`, raise `SoapyChunkSamples`, or reduce the number of enabled channels — see [Troubleshooting](#troubleshooting).

## How to use

### Requirements

The decoder is a standalone self-contained executable — just run it directly, no installation needed for the program itself. If it doesn't start, install the [.NET 8 Runtime](https://dotnet.microsoft.com/download/dotnet/8.0).

Beyond that, what you need depends on your chosen backend:

- **SDRconnect backend:** [SDRconnect](https://www.sdrplay.com/sdrconnect/) running with a compatible SDRplay device, WebSocket API enabled (below).
- **SoapySDR backend:** SoapySDR core + the driver module for your device, and the bridge library built for your OS (below).

---

### Using the SDRconnect backend

#### Enabling the SDRconnect WebSocket

The decoder receives IQ samples from SDRconnect over a WebSocket connection. This must be enabled before starting the decoder:

1. Open SDRconnect
2. Go to **Settings → API / WebSocket**
3. Enable the WebSocket server
4. Set the centre frequency, mode to `NFM`, and sample rate to `250 ksps`
5. Start the SDR streaming

SDRconnect must be running and the WebSocket must be active before you launch the decoder. In `settings.json`, make sure:
```json
"SdrBackend": "sdrconnect"
```

---

### Using the SoapySDR backend

#### 1. Install SoapySDR and your device's driver module

**Windows:** the easiest path is the [PothosSDR](https://github.com/pothosware/PothosSDR/wiki/Tutorial) bundle, which installs SoapySDR core plus modules for RTL-SDR, HackRF, Airspy, SDRplay, UHD, and others in one installer. Note the install path (typically `C:\Program Files\PothosSDR`) — you'll need its `include`/`lib` folders to build the bridge DLL, and its `bin` folder needs to be on your `PATH` (or its DLLs copied next to `RinoDecoder.exe`) so the decoder can find `SoapySDR.dll` and the driver modules at runtime.

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get update
sudo apt-get install soapysdr-tools libsoapysdr-dev
# Then whichever driver module(s) match your hardware:
sudo apt-get install soapysdr-module-rtlsdr    # RTL-SDR
sudo apt-get install soapysdr-module-hackrf    # HackRF
sudo apt-get install soapysdr-module-airspy    # Airspy
sudo apt-get install soapysdr-module-uhd       # USRP
```
SDRplay is the one exception — Soapy's SDRplay module depends on SDRplay's own (proprietary, closed-source) API package, which isn't in most distro repos. Install the [SDRplay API](https://www.sdrplay.com/api/) for Linux first (this also starts the `sdrplay_apiService` background service both SDRconnect and Soapy need), then either install `soapysdr-module-sdrplay3` if your distro packages it, or build [SoapySDRPlay3](https://github.com/pothosware/SoapySDRPlay3) from source against that API.

Verify SoapySDR sees your device before going further:
```bash
SoapySDRUtil --find
```
This prints the exact `driver=...` string to put in `SoapyDeviceArgs`.

#### 2. Build the bridge library

A small bridge library sits between this program and SoapySDR's API and needs to be compiled once, on the platform you'll run the decoder on. The source is included with the project.

**Linux:**
```bash
gcc -O3 -shared -fPIC soapy_bridge.c -o libsoapy_bridge.so -lSoapySDR
```
Place `libsoapy_bridge.so` in the same folder as the `RinoDecoder` executable.

**Windows** (cross-compiled with MinGW, or built natively with MSYS2's `mingw-w64-x86_64-gcc`):
```bash
x86_64-w64-mingw32-gcc -O3 -shared soapy_bridge.c -o soapy_bridge.dll \
  -ISoapySDR-master/dist_win/include -LSoapySDR-master/dist_win/lib -lSoapySDR
```
Point `-I`/`-L` at wherever your SoapySDR headers/import library actually live — if you installed PothosSDR, that's its `include`/`lib` folders. Place the resulting `soapy_bridge.dll` next to `RinoDecoder.exe`.

#### 3. Configure settings.json

```json
"SdrBackend": "soapy",
"SoapyDeviceArgs": "driver=rtlsdr",
"SoapyGainDb": 35.0,
"SoapyAgc": false,
"SoapyChunkSamples": 16384,
"SoapyChannel": 0
```
`SoapyDeviceArgs` is whatever `SoapySDRUtil --find` printed for your device. If you have more than one SDR connected, add enough of the printed string to disambiguate (e.g. a serial number).

---

### Live status displays

The same executable can run in extra display-only modes alongside the main
decoder. These don't do any decoding themselves — they read files the
decoder writes and display a live view roughly once a second.

**Web dashboard (recommended — this is the primary interface for the program).**
A single-page browser dashboard covering everything you'd otherwise need the
console modes for: live channel status, a live map of decoded positions, live
squelch control (including an auto-sensitivity feature), users, messages,
poll requests, and burst audio playback, all in one view that updates roughly
once a second. Opens automatically in your default browser:
```
RinoDecoder.exe --web
```
Optional flags:
```
RinoDecoder.exe --web --port 8080        # use a different port (default 5310)
RinoDecoder.exe --web --host 0.0.0.0     # listen on your LAN, not just this machine
```
This starts a small local web server (no external dependencies — it works
with no internet access). There's no login on it, so only pass
`--host 0.0.0.0` (or another non-localhost address) on a network you trust —
the default (`localhost`) only accepts connections from the same machine. On
Windows, binding to a non-localhost host may also require an admin-elevated
URL ACL reservation; the console will print the exact command if binding
fails.

**Live squelch / sensitivity.** The dashboard's "Live Squelch Control" card
lets you adjust detection sensitivity live — no restart needed. Check
**Auto threshold** to have the dashboard continuously pick a sensible
sensitivity for you based on current band conditions, which is generally
more effective for weak/marginal signals than manually tuning a fixed value
in `settings.json` and leaving it. Use **Reset to settings.json** to drop any
live adjustment and go back to your configured values.

**`AutoLaunchWebDashboard` (on by default) automatically launches this
dashboard as the primary interface every time the main decoder starts** —
you get the live map, channels, squelch, users, messages, and polls in your
browser with no separate `--web` command needed. It's in `settings.json`:

```json
"ProgramSettings": {
  "AutoLaunchWebDashboard": true
}
```

Set it to `false` if you'd rather start the dashboard only when you
explicitly run `RinoDecoder.exe --web` yourself.

**Channel status monitor (console)** — noise floor, signal level, SNR, and
burst count per channel, plus which frequency band is currently active:
```
RinoDecoder.exe --monitor
```

**Users monitor (console)** — every callsign seen so far with its last known
position and altitude, plus recent text messages and poll requests:
```
RinoDecoder.exe --users
```

The two console modes read the same underlying data as the web dashboard
and are useful over SSH or on a machine with no browser handy.

---

### Windows

**1. Install .NET 8**
Install the [.NET 8 Runtime](https://dotnet.microsoft.com/download/dotnet/8.0) (choose the Runtime, not the SDK, unless you're building from source).

**2. Install your chosen backend**
- SDRconnect: install [SDRconnect](https://www.sdrplay.com/sdrconnect/) and enable its WebSocket API (above).
- SoapySDR: install [PothosSDR](https://github.com/pothosware/PothosSDR/wiki/Tutorial), build `soapy_bridge.dll` (above), and place it next to `RinoDecoder.exe`.

**3. Edit settings.json (first time setup)**
Open `settings.json` in any text editor and set your backend, centre frequency, and optionally your latitude/longitude for distance calculations:
```json
"SdrBackend": "sdrconnect",
"UserLatitude": 0,
"UserLongitude": 0
```

**4. Run the decoder**
The program is standalone — just double-click `RinoDecoder.exe` or run it from a Command Prompt:
```
RinoDecoder.exe
```

Decoded positions will print to the console. A KML file (`live.kml`) is written to the same folder and updated on each successful decode.

---

### Linux

**1. Install .NET 8**
```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install -y dotnet-runtime-8.0

# Fedora/RHEL
sudo dnf install dotnet-runtime-8.0
```

**2. Install your chosen backend**
- SDRconnect: install SDRplay's [SDRconnect](https://www.sdrplay.com/sdrconnect/) for Linux and enable its WebSocket API (above).
- SoapySDR: install SoapySDR + your device's driver module and build `libsoapy_bridge.so` (above).

**3. Edit settings.json (first time setup)**
```bash
nano settings.json
```
Set your backend, centre frequency, and coordinates as above.

**4. Run the decoder**
Open a terminal in the RinoDecoder folder and run:
```bash
./RinoDecoder
```

---

### Viewing decoded positions in Google Earth

Each successful decode appends the position to `live.kml` in the program folder. Google Earth connects via `googleearthloader.kml`, which acts as a Network Link and automatically refreshes the map as new positions come in.

The decoder also keeps a `tracking/users.json` file with the same information in a different shape — one entry per callsign with its last known position, plus running logs of received messages and poll requests. This is what the `--users` monitor and the web dashboard read (see above); you generally won't need to open it directly.

Both `live.kml`/session state and `tracking/users.json` follow the same New/Continue choice at startup: choosing **New Live Session** starts both empty, and choosing **Continue Session** reloads whatever was saved from the last run.

**Google Earth Pro (desktop):**
1. Open Google Earth Pro
2. Go to **File → Open** and select `googleearthloader.kml`
3. Decoded positions appear as placemarks on the map, labelled with the callsign
4. Google Earth will automatically poll `live.kml` and update the map as new positions are decoded — no manual refresh needed

**Google Earth web (earth.google.com):**
1. Click the menu (☰) → **Import KML file**
2. Select `googleearthloader.kml`
3. Positions appear on the map and update automatically as new decodes come in

Each placemark includes the callsign, latitude, longitude, altitude, and the time of decode.

---

## Testing decode robustness with a weaker signal

If you have a strong, cleanly-decoding capture (via `SaveRawIq`), you can generate a weaker test copy of it to see how well the decoder holds up at lower signal strength — useful for tuning settings before you actually encounter a marginal real-world signal.

```
RinoDecoder.exe --attenuate --in recording/raw_burst.cs16 --db 15
```

This produces a copy of the recording where the burst itself is genuinely weaker relative to the surrounding noise — the same effect as a real transmitter being farther away, not just a quieter recording of the same scenario.

Flags:

| Flag | Required | Description |
|---|---|---|
| `--in <path>` | No | Input capture file. Defaults to your configured raw-burst file. |
| `--db <N>` | Yes | How many dB weaker to make the signal. Must be ≥ 0. |
| `--out <path>` | No | Output path. Defaults next to the input file. |
| `--seed <N>` | No | Fixed random seed, for reproducible test cases. Omit for a different result each run. |
| `--steps` | No | Also export intermediate files for visual inspection in a spectrogram tool such as [inspectrum](https://github.com/miek/inspectrum). |

To test the result, rename/copy the output file (and its `.meta.json` sidecar) to your configured raw-burst file, and use the `[Y] Decode` prompt the decoder shows on startup when that file already exists (see [How to use](#how-to-use)), or point the file-decode mode at it directly.

**What this does and doesn't test:** replaying a saved file exercises the decoder itself — it does not test whether a live capture would have triggered on a signal this weak in the first place, since file-replay already knows where the burst is. Testing that requires actually running the live capture with a weak signal present, and adjusting live sensitivity (see the web dashboard's auto-threshold, above) if it isn't triggering.

---

## Recovering signals that are visible but not auto-detected

If a weak burst is visibly present in a spectrogram but the decoder isn't finding it, a few `settings.json` values are worth lowering — see the settings reference below for `DirectBurstStdThreshold`, the `Preamble*` thresholds, and `CoarseSegmentationLpfHz`. Use `--attenuate` (above) with a saved capture to try different values against the same weak signal repeatedly.

---

## Troubleshooting

**"soapy_bridge failed to open device" / decoder exits immediately on the Soapy backend**
Run `SoapySDRUtil --find` — if your device isn't listed, SoapySDR itself can't see it yet (driver module not installed, device not plugged in, or on Linux, a udev rules/permissions issue — check that your device shows up under a non-root user). If it *is* listed, make sure `SoapyDeviceArgs` in `settings.json` matches the printed driver string exactly, including a serial/index if you have more than one SDR connected.

**`[BRIDGE] Overflow ×N` messages**
The device or host can't keep up with the requested data rate. Try, in order: raise `SoapyChunkSamples` in `settings.json`, lower `SampleRate`, reduce the number of enabled channels in `Channels[]`, or move to a USB3 port / powered hub if the device is USB-bus-powered.

**SDRconnect backend: decoder connects but no IQ ever arrives**
Confirm SDRconnect itself shows the SDR actively streaming (not just tuned) before you start the decoder — the WebSocket only delivers samples while SDRconnect's own capture is running. Double-check the WebSocket API is toggled on under **Settings → API / WebSocket**.

**`[WARN] 0 channels registered at ... MHz`**
Check `Channels[].Enabled` in `settings.json`, and that those channels' `FreqHz` values actually fall within the capture window printed alongside the warning of whichever band is currently active.

**Bursts are detected (`[BURST] ...` lines appear) but nothing decodes**
- If `UserLatitude`/`UserLongitude` are set, decoded positions outside a several-degree box around them are silently rejected as implausible — confirm they're set correctly, or set both to `0` to fall back to a built-in North America default.
- Try a higher `DspDebugLevel` (2 or 3) for more diagnostic output.
- Enable `SaveRawIq` and inspect/replay the saved capture — see `RawBurstFile` in the settings reference below.

**High CPU usage / dropped chunks under load**
Disable channels/repeaters you don't need, lower `DspDebugLevel` (verbose logging has a cost), or reduce `SampleRate` if you're capturing far more bandwidth than your enabled channels actually span.

**`--web` dashboard won't bind / browser shows "can't connect"**
See the bind failure guidance printed to the console — usually either the port is already in use (`--port` to pick another) or, if you passed `--host` with something other than `localhost`, Windows may need an admin-elevated URL ACL reservation (the exact command is printed on failure).

**Missing runtime error on startup**
Install the [.NET 8 Runtime](https://dotnet.microsoft.com/download/dotnet/8.0) — the **Runtime**, not the SDK, unless you're building from source.

---

## settings.json Reference

### ProgramSettings

Controls the SDR front end and capture pipeline.

| Setting | Default | Description |
|---|---|---|
| `SdrBackend` | `"sdrconnect"` | Which backend to use: `"sdrconnect"` or `"soapy"`. See [Choosing a backend](#choosing-a-backend-sdrconnect-vs-soapysdr). |
| `SoapyDeviceArgs` | `"driver=sdrplay"` | SoapySDR device-selector string, only used when `SdrBackend` is `"soapy"`. Find yours with `SoapySDRUtil --find`. |
| `SoapyGainDb` | `35.0` | Manual RX gain in dB, only used when `SoapyAgc` is `false`. |
| `SoapyAgc` | `false` | Enable the device's hardware AGC instead of the fixed `SoapyGainDb` value. |
| `SoapyPpm` | `0.0` | Frequency correction in parts-per-million, if your device has a known crystal offset. |
| `SoapyChunkSamples` | `16384` | Samples read per hardware read call. Raise this if you see `[BRIDGE] Overflow` messages. |
| `SoapyChannel` | `0` | RX channel index for multi-channel devices (e.g. RSPduo). `0` for single-channel devices. |
| `SampleRate` | `250000` | IQ sample rate in samples/sec. For SDRconnect, set SDRconnect itself to match. |
| `Modulation` | `NFM` | SDRconnect modulation mode. NFM (narrowband FM) is correct for RINO. Not used on the Soapy backend. |
| `DspDebugLevel` | `1` | Verbosity. `0` = silent, `1` = decoded positions only, `2` = more detail, `3` = full debug output. |
| `Threshold` | `0.0010` | Absolute floor for burst detection sensitivity. Raise if false triggers occur on a noisy channel; lower if weak signals are being missed. |
| `ThresholdSigmas` | `4.0` | Per-channel detection sensitivity as a multiple of that channel's measured noise floor. Lower is more sensitive. |
| `SlotPreRollMs` | `100` | Milliseconds of audio prepended to a burst when it's first detected, so the very start isn't clipped. |
| `SlotHoldMs` | `3000` | Milliseconds detection stays "open" after signal drops before a burst is considered ended. |
| `ChannelBandwidthHz` | `12000` | Per-channel filter bandwidth in Hz. |
| `SaveRawIq` | `false` | If true, saves the raw IQ of the last detected burst to `RawBurstFile` for diagnostic use. Each detected burst overwrites the file — it always holds just the single most recent burst, not an accumulating log of the whole session. |
| `RawBurstFile` | `"raw_burst.cs16"` | Path for the saved raw burst IQ file. A sidecar file is written alongside it recording the sample rate it was actually captured at. The decoder's file-decode prompt reads this automatically. Every save also archives a timestamped, per-channel copy in the same `recording/` folder so earlier bursts aren't lost. These archives accumulate indefinitely with no automatic cleanup — worth an occasional manual prune on a long-running session with frequent traffic. Used when `SaveRawIq` is enabled. |
| `MaxDecodes` | `8` | Maximum number of concurrent burst decode attempts before new ones are skipped. |
| `MonitorRepeaters` | `false` | Whether to also register repeater-input channels (labels ending in `R`, or `IsRepeater: true`). |
| `ShowStatusDisplay` | `true` | Whether the decoder writes live status data for `--monitor`/`--web` to read. Must stay `true` for `--monitor` and the web dashboard's channel table/live squelch/auto-threshold to work — leave this on. |
| `AutoLaunchWebDashboard` | `true` | If true, automatically launches the web dashboard (the primary interface — live map, channels, squelch, users, messages, polls) every time the main decoder starts, opening it in your default browser with no separate command needed. See [Live status displays](#live-status-displays). Set to `false` to only start the dashboard when you explicitly run `RinoDecoder.exe --web`. |

---

### DecoderSettings

Controls the decoder's detection and demodulation behavior. These are auto-calibrated per capture — most values are thresholds that define what counts as a valid transmission. The settings most worth adjusting when tuning for weak signals are called out below the table.

| Setting | Default | Description |
|---|---|---|
| `PreRollMs` | `0.003` | Seconds of audio prepended to the detected burst before decoding, to compensate for detection latency. |
| `PostRollMs` | `0.003` | Seconds of audio appended after the burst end. |
| `MinBurstMs` | `265.0` | Minimum data burst duration in ms accepted as valid. |
| `MaxBurstMs` | `290.0` | Maximum data burst duration in ms accepted as valid. |
| `MinPreambleMs` | `80.0` | Minimum preamble duration in ms to be considered valid. Short blips below this are ignored. |
| `MaxPreambleMs` | `600.0` | Maximum preamble duration in ms. Longer regions are discarded (likely voice or another signal). |
| `WindowSamplesMs` | `0.005` | Analysis window width in seconds. Smaller = finer time resolution but noisier estimates — widening this slightly can stabilize classification at low signal strength. |
| `StrideSamplesMs` | `0.001` | Window stride in seconds. Smaller strides give smoother detection at higher CPU cost. |
| `BaudRatio` | `1.5625` | Internal timing ratio used to derive one measured rate from another. |
| `ManualLpfHz` | `1500` | A positive value is a true override for the demodulation filter cutoff (safety-clamped). Set to `0` to enable automatic mode based on measured signal timing. |
| `LpfBaudMultiplier` | `1.5` | Multiplier used by automatic filter-cutoff mode. Only takes effect when `ManualLpfHz` is `0`. Lower reduces noise bandwidth (helps weak signals) but risks distorting the signal if pushed too low. |
| `LpfMinHz` | `500` | Floor clamp on the automatic filter cutoff. Only used when `ManualLpfHz` is `0`. |
| `LpfMaxHz` | `10000` | Ceiling clamp on the automatic filter cutoff. Only used when `ManualLpfHz` is `0`. |
| `PreambleMaxStd` | `1500` | Upper bound on signal variability for a region to be classified as preamble. |
| `PreambleMinStd` | `75` | Lower bound on signal variability for a preamble region — excludes silence. |
| `PreambleLowFreqConc` | `0.5` | Minimum low-frequency concentration for a region to be classified as preamble; tends to drop as signal strength falls. |
| `BurstStdMultiplier` | `2.0` | Detection gate for the data portion of a transmission, relative to the measured preamble. Increase if preamble regions are being misidentified as data; decrease if weak data is being missed. |
| `DirectBurstStdThreshold` | `1000` | Detection gate used when no valid preamble is found at all — the primary path when detection is struggling on a weak signal. Lower it if weak transmissions without a clean preamble aren't being found. |
| `FallbackDataBaud` | `615` | Nominal data timing rate used when no preamble is available to measure it from directly. |
| `FallbackBaudRangePct` | `2` | ± percent search range around `FallbackDataBaud`. Set to `0` to skip refinement and use the nominal value outright. |
| `FmDemodBandwidth` | `12000` | Pre-demodulation channel filter bandwidth in Hz. There's headroom to narrow this for a genuine noise-bandwidth reduction. |
| `FmRfFilterTaps` | `151` | Filter sharpness for `FmDemodBandwidth`. More taps = sharper cutoff, at the cost of eating a few more samples from each end of the burst (see the coupling note below) and negligible extra CPU. |
| `FmBasebandFilterTaps` | `51` | Filter sharpness for the post-demodulation cleanup filter. **Under-provisioned by default for narrow cutoffs** — narrowing `LpfBaudMultiplier`/`ManualLpfHz` alone has much less effect than expected unless this is also raised. |
| `CoarseSegmentationLpfHz` | `0` | Filter cutoff used specifically for detecting whether a transmission is present at all, separate from the filter used for the final decode. `0` (default) keeps automatic behavior. A positive value is a true override (clamped 200–8000 Hz) — don't push it too low, or detection loses the information it needs. Narrowing this is a genuine noise-bandwidth reduction for detection specifically, without affecting the final decode quality. |
| `MmLoopBandwidth` | `0.035` | How quickly bit-timing tracking adapts. Lower values track more slowly but with less noise pulled in — usually worth reducing for weak signals, provided the initial timing estimate is still clean enough to start from. |
| `MmDamping` | `0.707` | Bit-timing tracking damping factor. Rarely needs adjustment on its own — usually tuned alongside `MmLoopBandwidth`. |
| `ExportDecodeSteps` | `false` | If true, writes intermediate processing stages of every decoded burst to disk, for visual before/after comparison in a spectrogram tool. Performs real disk I/O per burst — meant for deliberate tuning sessions, not left-on normal operation. |
| `DecodeStepsDir` | `"decode_steps"` | Folder (relative to the executable) that `ExportDecodeSteps` writes its per-burst subfolders into. |
| `EnableLiveBurstAudio` | `false` | If true, exports a listenable audio file of each detected burst, which the web dashboard's Burst Audio panel plays. |
| `SaveBurstAudioFiles` | `false` | Only relevant when `EnableLiveBurstAudio` is true. If true, keeps a separate file per burst (up to `MaxBurstAudioFiles`, oldest pruned first) instead of just keeping the latest. |
| `MaxBurstAudioFiles` | `10` | Maximum number of saved burst audio files kept when `SaveBurstAudioFiles` is true. |

**Most worth adjusting for weak signals**, roughly in order of impact: `LpfBaudMultiplier` together with `FmBasebandFilterTaps` (set `ManualLpfHz` to `0` first to enable automatic mode at all) and `FmDemodBandwidth`/`FmRfFilterTaps` for genuine noise-bandwidth reduction; `CoarseSegmentationLpfHz` for narrowing the *detection*-side noise bandwidth specifically; `DirectBurstStdThreshold` and the `Preamble*` thresholds since they degrade first as signal strength drops; `MmLoopBandwidth`/`MmDamping` for bit-timing robustness once demodulation itself is clean. Note that none of these affect whether a live capture triggers in the first place — that's `ProgramSettings.Threshold`/`ThresholdSigmas`, which gate detection upstream of everything in this table (see the web dashboard's auto-threshold control, above, for the most effective way to tune that).

**One coupling to watch:** raising `FmRfFilterTaps`/`FmBasebandFilterTaps` for sharper filtering isn't free — it eats a small number of samples from each end of the burst as a side effect. Push tap counts too far without also raising `PreRollMs`/`PostRollMs` and you'll start losing real content at the edges rather than just margin.

---

### RinoSettings

Controls position decoding and output.

| Setting | Default | Description |
|---|---|---|
| `UserLatitude` | `0` | Observer latitude in decimal degrees. Set to your location for distance calculations and improved decoding accuracy. Set to 0 to disable. |
| `UserLongitude` | `0` | Observer longitude in decimal degrees. Set to your location alongside `UserLatitude`. |
| `BitDrift` | `2` | How much alignment slack the decoder allows when a decoded frame doesn't line up cleanly. |
| `RetryOnFail` | `true` | If the primary decode attempt fails, retry with alternate alignments before giving up. |
