**Download the latest release under the "releases" page or click here:** (https://github.com/hklinge32/GARMIN-RINO-DECODER-SDRConnect-SoapySDR/releases/tag/Rino_Decoder_V1.02)

# Garmin RINO Burst Decoder

A decoder for Garmin RINO position broadcasts transmitted over FRS/GMRS frequencies. Captures live radio signals and decodes position, callsign, altitude, and icon data in real time.

The decoder supports two independent SDR backends — pick whichever matches your hardware:

- **SDRconnect** — SDRplay's own capture application. Only works with SDRplay hardware (RSP1A, RSPdx, RSP1B, etc.), but requires no compilation on your part.
- **SoapySDR** — a hardware-abstraction layer supporting a wide range of SDRs from different vendors (RTL-SDR, HackRF, Airspy, SDRplay, LimeSDR, PlutoSDR, USRP, and more). Requires building a small bridge library once, but isn't tied to one vendor.

See [Choosing a backend](#choosing-a-backend-sdrconnect-vs-soapysdr) below for the tradeoffs, and [Supported SDRs via SoapySDR](#supported-sdrs-via-soapysdr) for hardware-specific notes.

## What it does

The RINO radio transmits a short data burst on the FRS/GMRS channel containing the sender's GPS position, callsign, altitude, and icon. This program listens continuously, detects those transmissions, and decodes the position data.

Detection and decoding is fully automatic — no manual tuning required for typical use.

## Choosing a backend: SDRconnect vs SoapySDR

The choice only affects how signal data gets from your hardware into the program. Set it in `settings.json`:

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

This program only ever tunes within the FRS/GMRS band (~462–467 MHz), so nearly any SoapySDR-supported device covers the tuning range you need. The real differences between devices show up in dynamic range and USB streaming stability. `SoapyDeviceArgs` in `settings.json` must match the driver string for your device (find it with `SoapySDRUtil --find`, see [Troubleshooting](#troubleshooting)).

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

- **You don't need a high sample rate.** Requesting more than a couple Msps from the hardware just spends CPU without improving decode quality — it only matters if you want to monitor more channels at once without switching between them.
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

The decoder receives data from SDRconnect over a WebSocket connection. This must be enabled before starting the decoder:

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

**Windows:** the easiest path is the [PothosSDR](https://github.com/pothosware/PothosSDR/wiki/Tutorial) bundle, which installs SoapySDR core plus modules for RTL-SDR, HackRF, Airspy, SDRplay, UHD, and others in one installer. Note the install path (typically `C:\Program Files\PothosSDR`) — its `bin` folder needs to be on your `PATH` (or its DLLs copied next to `RinoDecoder.exe`) so the decoder can find `SoapySDR.dll` and the driver modules at runtime.

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

#### 2. Configure settings.json

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
decoder. Run these in a **separate terminal window** while the decoder is
running.

**Web dashboard (recommended)** — a single-page browser dashboard combining
channel status and users/messages/polls in one modern, dark-themed view.
Opens automatically in your default browser:
```
RinoDecoder.exe --web
```
Optional flags:
```
RinoDecoder.exe --web --port 8080        # use a different port (default 5310)
RinoDecoder.exe --web --host 0.0.0.0     # listen on your LAN, not just this machine
```
This starts a small local web server with no login on it, so only pass
`--host 0.0.0.0` (or another non-localhost address) on a network you trust —
the default (`localhost`) only accepts connections from the same machine.
On Windows, binding to a non-localhost host may also require an
admin-elevated permission; the console will print the exact command if
binding fails.

You can also have the decoder start the dashboard automatically when the main
program launches. Add this to `settings.json`:

```json
"ProgramSettings": {
  "AutoLaunchWebDashboard": true
}
```

Set it to `false` if you prefer to start the dashboard only when you run
`RinoDecoder.exe --web` yourself.

**Channel status monitor (console)** — noise floor, signal level, and
burst count per channel:
```
RinoDecoder.exe --monitor
```

**Users monitor (console)** — every callsign seen so far with its last known
position and altitude, plus recent text messages and poll requests:
```
RinoDecoder.exe --users
```

The two console modes are useful over SSH or on a machine with no browser
handy.

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

The decoder also keeps a `tracking/users.json` file with the same information in a different shape — one entry per callsign with its last known position, plus running logs of received messages and poll requests. This is what the `--users` monitor reads (see above); you generally won't need to open it directly.

Both `live.kml`/`tracking/session.json` and `tracking/users.json` follow the same New/Continue choice at startup: choosing **New Live Session** starts both empty, and choosing **Continue Session** reloads whatever was saved from the last run.

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

## Troubleshooting

**"soapy_bridge failed to open device" / decoder exits immediately on the Soapy backend**
Run `SoapySDRUtil --find` — if your device isn't listed, SoapySDR itself can't see it yet (driver module not installed, device not plugged in, or on Linux, a udev rules/permissions issue — check that your device shows up under a non-root user). If it *is* listed, make sure `SoapyDeviceArgs` in `settings.json` matches the printed driver string exactly, including a serial/index if you have more than one SDR connected.

**`[BRIDGE] Overflow ×N` messages**
The device or host can't keep up with the requested data rate. Try, in order: raise `SoapyChunkSamples` in `settings.json`, lower `SampleRate`, reduce the number of enabled channels in `Channels[]`, or move to a USB3 port / powered hub if the device is USB-bus-powered.

**SDRconnect backend: decoder connects but no data ever arrives**
Confirm SDRconnect itself shows the SDR actively streaming (not just tuned) before you start the decoder. Double-check the WebSocket API is toggled on under **Settings → API / WebSocket**.

**`[WARN] 0 channels registered at ... MHz`**
Check `Channels[].Enabled` in `settings.json`, and that those channels' `FreqHz` values actually fall within range of whichever cluster centre is active (printed alongside the warning).

**Bursts are detected (`[BURST] ...` lines appear) but nothing decodes**
- If `UserLatitude`/`UserLongitude` are set, decoded positions outside a several-degree box around them are silently rejected as implausible — confirm they're set correctly, or set both to `0` to fall back to the built-in North America heuristic.
- Try a higher `DspDebugLevel` (2 or 3) for more detail in the console output about why a particular burst isn't decoding.

**High CPU usage / dropped chunks under load**
Disable channels/repeaters you don't need, lower `DspDebugLevel` (verbose logging itself has a cost at level 3), or reduce `SampleRate` if you're capturing far more bandwidth than your enabled channels actually span.

**`--web` dashboard won't bind / browser shows "can't connect"**
See the bind failure guidance printed to the console — usually either the port is already in use (`--port` to pick another) or, if you passed `--host` with something other than `localhost`, Windows may need an admin-elevated permission (the exact command is printed on failure).

**Missing runtime error on startup**
Install the [.NET 8 Runtime](https://dotnet.microsoft.com/download/dotnet/8.0) — the **Runtime**, not the SDK, unless you're building from source.

---

## settings.json Reference

### ProgramSettings

Controls the SDR front end and capture behavior.

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
| `DspDebugLevel` | `1` | Console verbosity. `0` = silent, `1` = decoded positions only, `2` = more detail, `3` = full debug output. |
| `Threshold` | `0.0010` | Amplitude gate for burst detection. Raise if false triggers occur on a noisy channel; lower if weak signals are being missed. |
| `ThresholdSigmas` | `4.0` | Per-channel squelch threshold as a multiple of that channel's measured noise floor. |
| `ChannelBandwidthHz` | `12000` | Per-channel filter bandwidth in Hz. |
| `SaveRawIq` | `false` | If true, saves the raw IQ of the last detected burst to `RawBurstFile` for diagnostic use. |
| `RawBurstFile` | `"raw_burst.cs16"` | Path for the saved raw burst file, when `SaveRawIq` is enabled. Each save also archives a timestamped copy under `recording/`. These archives accumulate with no automatic cleanup — worth an occasional manual prune on a long-running session with frequent traffic. |
| `MaxDecodes` | `8` | Maximum number of concurrent burst decode attempts before new ones are skipped. |
| `MonitorRepeaters` | `false` | Whether to also register repeater-input channels (labels ending in `R`, or `IsRepeater: true`). |
| `ShowStatusDisplay` | `true` | Whether the decoder writes status data for `--monitor`/`--web` to read. |

---

### RinoSettings

Controls position decoding and output.

| Setting | Default | Description |
|---|---|---|
| `UserLatitude` | `0` | Observer latitude in decimal degrees. Set to your location for distance calculations and improved decoding accuracy. Set to 0 to disable. |
| `UserLongitude` | `0` | Observer longitude in decimal degrees. Set to your location alongside `UserLatitude`. |
| `BitDrift` | `2` | Search width used when aligning an ambiguous frame. Rarely needs adjustment. |
| `RetryOnFail` | `true` | If the primary decode fails, retry with alternate alignment. |
