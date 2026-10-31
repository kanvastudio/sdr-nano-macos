
# MSI SDR Nano on macOS — SDR++ Troubleshooting

**Tested fixes for SDRplay API libraries, background services, USB detection, and startup problems.**

The MSI SDR Nano worked with CubicSDR on my Intel MacBook Pro, but SDR++ initially couldn't detect it. After investigating shared libraries, the SDRplay API service, USB errors, and macOS background permissions, I got it working reliably.

This repository documents the diagnostic commands and fixes that worked on my system.

**[Read the complete investigation on BestQuest](https://bestquest.net/msi-nano-sdr-sdrplusplus-macos-fix/)**

## Tested Configuration

| Component | Configuration |
|---|---|
| Receiver | Chinese MSI Nano, RSP1-compatible |
| Device identification | SDRplay RSP1, B0020P0003 |
| Computer | Intel MacBook Pro |
| Operating system | macOS Ventura 13 |
| SDR++ | v1.3.0, July 5, 2026 build |
| SDRplay API | 3.15.1 |
| SoapySDR interface | Mirics/SDRplay API 2.13 |
| Connection | Direct USB, original cable |

**Important:** This documents one tested configuration. Other SDR clones and software versions may behave differently.

## 1. Check Receiver Detection

Close SDR++ and CubicSDR.

Run:

```bash
SoapySDRUtil --find
```

My receiver appeared as:

```text
Found device 0
driver = sdrplay
label = SDRplay Dev0 RSP1 B0020P0003
```

Then probe the device:

```bash
SoapySDRUtil --probe="driver=sdrplay"
```

Relevant output:

```text
driver=SDRplay
hardware=B0020P0003
mir_sdr_api_version=2.130000
mir_sdr_hw_version=1
```

This established that SoapySDR could detect and communicate with the receiver.

## 2. Check the SDR++ Source Module

The SDRplay source was missing from SDR++ even though the module existed.

Locate SDR++:

```bash
find /Applications "$HOME/Applications" -name sdrpp -type f 2>/dev/null
```

My installation was named `SDR++ 2.app`.

Check the module:

```bash
ls -l "/Applications/SDR++ 2.app/Contents/Plugins/sdrplay_source.dylib"
```

Inspect its library dependencies:

```bash
otool -L "/Applications/SDR++ 2.app/Contents/Plugins/sdrplay_source.dylib"
```

The module required:

```text
@rpath/libsdrplay_api.so.3
```

Locate the installed API library:

```bash
find /usr/local /opt/homebrew /Library -name 'libsdrplay_api.so.3' 2>/dev/null
```

On my Mac, it was located at:

```text
/usr/local/lib/libsdrplay_api.so.3
```

## 3. Fix the Library Search Path

My SDR++ module expected the library inside its application bundle.

After confirming that the destination did not already exist, I created a symbolic link:

```bash
ln -s /usr/local/lib/libsdrplay_api.so.3 "/Applications/SDR++ 2.app/Contents/Frameworks/libsdrplay_api.so.3"
```

**Caution:** This modifies the SDR++ application bundle and may affect code signing. An application update may remove the link. Do not overwrite an existing library.

This resolved the missing-library problem, but the SDRplay API service still needed attention.

## 4. Check the SDRplay API Service

Launch SDR++ from Terminal:

```bash
"/Applications/SDR++ 2.app/Contents/MacOS/sdrpp" 2>&1 | tee "$HOME/Desktop/sdrpp-log.txt"
```

My log reported:

```text
[ERROR] Could not initialized the SDRplay API.
Make sure that the service is running.
```

Check the service:

```bash
ps aux | grep '[s]drplay_apiService'
```

Check its registration:

```bash
sudo launchctl print system/com.sdrplay.service
```

When the service was not registered, I verified the installed LaunchDaemon:

```bash
ls -l /Library/LaunchDaemons/com.sdrplay.service.plist
```

Then loaded it:

```bash
sudo launchctl bootstrap system /Library/LaunchDaemons/com.sdrplay.service.plist
```

After the service started, SDR++ recognized the SDRplay source and receiver.

Do not repeatedly bootstrap an already registered service.

## 5. Recover from Intermittent Detection Failures

The receiver occasionally disappeared or returned:

```text
No device available
```

Other errors included:

```text
[ERROR] Could not select RSP device: sdrplay_api_Fail
```

And:

```text
libusb: error [darwin_claim_interface] interface not found
```

Restarting the SDRplay API service restored operation on my system.

Close all SDR applications first.

Stop the service:

```bash
sudo launchctl bootout system/com.sdrplay.service
```

Wait approximately five seconds.

Restart it:

```bash
sudo launchctl bootstrap system /Library/LaunchDaemons/com.sdrplay.service.plist
```

Verify:

```bash
ps aux | grep '[s]drplay_apiService'
```

Reopen SDR++, select SDRplay, refresh the device list, and start reception.

This procedure restored FM reception without unplugging the USB receiver.

## 6. Fix Automatic Startup After Reboot

Although the service worked manually, it initially failed to start automatically after reboot.

Check the background-item database:

```bash
sudo sfltool dumpbtm
```

The SDRplay service initially reported:

```text
Disposition: [enabled, disallowed, visible, notified]
```

On macOS Ventura:

1. Open **System Settings**.
2. Select **General → Login Items**.
3. Find **SDRplay Limited** under **Allow in the Background**.
4. Enable it.

Afterward, the service reported:

```text
Disposition: [enabled, allowed, visible, notified]
```

I also replaced a symbolic-link LaunchDaemon configuration with a regular file during troubleshooting. Because both changes preceded the successful reboot, I cannot establish which was decisive.

That advanced LaunchDaemon modification is documented in the full article, but it is **not recommended as a routine fix**.

## 7. Final Verification

After restarting macOS, I checked the service:

```bash
sudo launchctl print system/com.sdrplay.service
```

The output included:

```text
state = running
pid = 298
last exit code = (never exited)
properties = keepalive | runatload
```

SDR++ recognized the receiver.

The spectrum and waterfall displayed correctly, FM reception worked with audio, and the receiver continued working after USB disconnection and reconnection.

## Troubleshooting Summary

| Symptom | First thing to check |
|---|---|
| Receiver missing from SoapySDR | USB connection and device recognition |
| SDRplay missing from SDR++ Source menu | Source module and library dependencies |
| SDRplay API initialization error | Background service registration |
| Receiver disappears intermittently | API state and USB errors |
| Works until reboot | Login Items permissions and LaunchDaemon |
| CubicSDR works but SDR++ fails | Different API paths and SDR++ dependencies |

## Full Technical Article

The complete investigation includes additional diagnostics, Terminal outputs, explanations, recovery steps, and limitations.

**[MSI Nano Worked in CubicSDR but Not SDR++ — How I Fixed It on macOS](https://bestquest.net/msi-nano-sdr-sdrplusplus-macos-fix/)**

Related hardware review:

[MSI SDR Nano — Receiver Review](https://bestquest.net/msi-sdr-nano-review/)

## Contributions

Corrections and reproducible findings from other macOS versions, receiver revisions, and SDR++ builds are welcome.

When opening an issue, include the macOS version, receiver model, SDR++ version, and relevant diagnostic output. Remove personal information before publishing logs.

## About BestQuest

[BestQuest](https://bestquest.net/) publishes independent research, reviews, comparisons, and practical guides across technology, software, electronics, and other subjects.

**Know what matters before you choose.**

This repository documents practical troubleshooting experience and is provided without warranty.
