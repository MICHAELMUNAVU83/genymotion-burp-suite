# Genymotion — Installing Apps & Intercepting Traffic with Burp Suite

A practical guide for setting up a Genymotion Android virtual device for mobile app testing: installing apps (APK, ADB, Google Play) and routing HTTPS traffic through Burp Suite (proxy + CA certificate, including the system-level cert needed on Android 7+).

> Intended for security testing of apps you own or are authorized to test.

---

## Part 1 — Adding Apps to Genymotion

You have four reliable ways to get apps onto a running device.

### Method 1 — Drag & Drop an APK (fastest)

1. Start your virtual device in Genymotion Desktop.
2. Drag the `.apk` file from your file manager directly onto the device window.
3. Genymotion installs it automatically and shows a confirmation toast.

Works for single APKs. For **split APKs** (`base.apk` + `config.*.apk`), use ADB (Method 2).

### Method 2 — Install via ADB

Genymotion bundles its own `adb`, or you can use the one from Android SDK platform-tools.

```bash
# 1. Confirm the device is visible (it connects over TCP)
adb devices

# If it doesn't appear, connect manually using the device's IP
# (shown in the Genymotion device settings / title bar, e.g. 192.168.56.101)
adb connect 192.168.56.101:5555

# 2. Install a single APK
adb install app.apk

# 3. Install a split APK set
adb install-multiple base.apk split_config.arm64_v8a.apk split_config.en.apk

# 4. Reinstall / downgrade
adb install -r -d app.apk
```

> If `adb` is not on your PATH, use the one shipped with Genymotion, e.g.
> `~/.config/Genymobile/Genymotion/tools/adb` (Linux) or the `tools` folder in the install dir on Windows/macOS.

### Method 3 — Google Play Store (Open GApps)

To run apps that need Google Play Services (or to install from the Play Store directly):

1. With the device running, click the **Open GApps** widget in the Genymotion toolbar (right-hand side).
2. Accept the prompt; Genymotion downloads and flashes the matching GApps package.
3. **Reboot** the device when prompted.
4. Open the **Play Store**, sign in with a Google account, and install apps normally.

> Match GApps to the device's Android version — the Open GApps widget handles this automatically. On older Genymotion builds without the widget, you'd flash an Open GApps zip manually, but the widget is the supported path now.

### Method 4 — Pull an APK off another device

If you already have the app on a physical phone:

```bash
# Find the package's APK path
adb shell pm path com.example.app
# -> package:/data/app/.../base.apk

# Pull it to your machine, then install into Genymotion (Method 2)
adb pull /data/app/.../base.apk app.apk
```

---

## Part 2 — Connecting Genymotion to Burp Suite

Goal flow: **route device traffic through Burp → install Burp's CA cert → verify HTTPS decrypts.**

### Step 1 — Configure the Burp proxy listener

1. In Burp: **Proxy → Proxy settings** (older versions: **Proxy → Options**).
2. Edit the listener (default `127.0.0.1:8080`).
3. Set **Bind to address** to **All interfaces** (or your host's specific LAN/host-only IP) so the virtual device can reach it.
4. Keep the port as **8080** (or note whatever you choose).

### Step 2 — Find your host machine's IP

Genymotion devices sit on a host-only virtual network, typically `192.168.56.0/24`, with the host reachable at **`192.168.56.1`**. You can also use your normal LAN IP.

```bash
# Linux / macOS
ip addr | grep 192.168.56      # or: ifconfig
# Windows
ipconfig                        # look for the Genymotion/VirtualBox host-only adapter
```

**On macOS**, get your primary Wi-Fi address directly:

```bash
ipconfig getifaddr en0         # prints the IPv4 of en0 (usually Wi-Fi)
```

> If `en0` returns nothing, Wi-Fi may be on another interface (often `en1`) — try that, or list all addresses with `ifconfig | grep "inet "`.
> Use the `en0` LAN IP only when the device is **bridged** onto your normal network. If it's on the host-only network, use the host adapter IP (often `192.168.56.1`) instead. When unsure, bind Burp to **All interfaces** and confirm from the device at `http://burp`.

### Step 3 — Point the device at Burp

1. On the device: **Settings → Network & internet → Wi-Fi**.
2. Long-press the connected network (usually **AndroidWifi**) → **Modify network**.
3. Expand **Advanced options** → set **Proxy** to **Manual**.
4. **Proxy hostname** = your host IP (e.g. `192.168.56.1`), **Proxy port** = `8080`.
5. Save.

### Step 4 — Verify the proxy works (HTTP)

1. Open the device browser and go to **`http://burp`**.
2. You should see the **Burp Suite** landing page. If so, HTTP traffic is flowing through Burp.
3. Browse to any `http://` site and confirm requests appear in **Proxy → HTTP history**.

HTTPS sites will show certificate errors until the CA cert is installed — that's next.

### Alternative — Set a device-wide proxy via ADB

Instead of editing the Wi-Fi network, you can push a global HTTP proxy from the command line. Useful when the UI proxy fields are awkward, or for scripting the setup.

```bash
# Set the global proxy
adb shell settings put global http_proxy <host_ip>:8080
# e.g. adb shell settings put global http_proxy 192.168.56.1:8080

# Verify it took
adb shell settings get global http_proxy

# Clear it when done
adb shell settings put global http_proxy :0
# (or fully remove: adb shell settings delete global http_proxy)
```

You can also set it from the Genymotion UI: **Settings → Network → Proxy**.

> This routes app HTTP traffic to Burp **without** installing a system CA. For **HTTPS**, you still need a trusted cert (user or system, Part 3) or an SSL-pinning bypass (Part 4) — the proxy alone only moves the traffic; it doesn't let Burp decrypt TLS.

---

## Part 3 — Installing the Burp CA Certificate

### Step 1 — Export Burp's CA certificate

In Burp: **Proxy → Proxy settings → Import / Export CA certificate → Certificate in DER format** → save as **`cacert.der`**.

(You can also download it from **`http://burp`** on the device.)

### Step 2 — Install as a *user* certificate (quick, limited)

Convert and install:

```bash
# Android expects .crt/.cer for user install; DER content is fine
cp cacert.der cacert.crt
```

On the device: **Settings → Security → Encryption & credentials → Install a certificate → CA certificate** → pick the file.

> **Limitation:** From **Android 7 (API 24)** on, apps ignore user-added CAs unless they explicitly opt in via `network_security_config`. So a user cert only intercepts the browser and a few cooperative apps. For most apps you need a **system** cert (below).

### Step 3 — Install as a *system* certificate (Android 7+, recommended)

Genymotion gives you root `adb`, so you can write to the system trust store.

**1. Produce the Android-format filename** (hash of the subject + `.0`):

```bash
openssl x509 -inform DER -in cacert.der -out cacert.pem
hash=$(openssl x509 -inform PEM -subject_hash_old -in cacert.pem | head -1)
cp cacert.pem "$hash.0"
echo "Certificate file: $hash.0"
```

**2. Push it into the system store:**

```bash
adb root                 # Genymotion permits root adb
adb remount              # make /system writable

adb push "$hash.0" /system/etc/security/cacerts/
adb shell chmod 644 /system/etc/security/cacerts/"$hash.0"

adb reboot
```

**3. Verify after reboot:** **Settings → Security → Trusted credentials → System** — your Burp CA (PortSwigger) should be listed.

> **Android 10+ note:** on some newer images the cert store is served from a tmpfs overlay and a plain `adb remount` won't persist. If `adb remount` fails or the cert vanishes after reboot, mount the cacerts directory as tmpfs, copy the full existing store back in plus your cert, then fix contexts — or simply use a Genymotion image at **Android 9** or below for the simplest system-cert install.

### Step 4 — Confirm HTTPS interception

1. Open an app or browse an `https://` site on the device.
2. Decrypted HTTPS requests should now appear in Burp's **HTTP history** with no TLS errors.

---

## Part 4 — Certificate Pinning (when apps still won't decrypt)

If traffic still won't intercept after a valid system cert, the app likely **pins** its certificate. Options:

- **Frida + objection** — runtime unpinning:
  ```bash
  pip install frida-tools objection
  # push & run frida-server (matching the device arch) on the device, then:
  objection -g com.example.app explore
  # inside objection:
  android sslpinning disable
  ```
- **Patch the APK** with a modified `network_security_config.xml` that trusts user CAs, then repackage/resign.
- Use a Frida script targeting the specific pinning library (OkHttp, TrustManager, etc.).

Pinning bypass is app-specific and beyond basic proxy setup — reach for it only after the proxy + system cert are confirmed working on an unpinned test.

---

## Quick Reference

| Task | Command / Action |
|------|------------------|
| Install APK | drag-drop, or `adb install app.apk` |
| Install split APK | `adb install-multiple *.apk` |
| Connect ADB | `adb connect <device_ip>:5555` |
| Add Play Store | **Open GApps** widget → reboot |
| Burp listener | bind **All interfaces**, port **8080** |
| Device proxy (UI) | Wi-Fi → Modify → Manual → `<host_ip>:8080` |
| Device proxy (ADB) | `adb shell settings put global http_proxy <host_ip>:8080` |
| Clear ADB proxy | `adb shell settings put global http_proxy :0` |
| Test proxy | browse to `http://burp` |
| Export CA | Burp → Proxy settings → Export → DER |
| System cert | `adb root && adb remount`, push `<hash>.0` to `/system/etc/security/cacerts/` |
| Bypass pinning | `objection ... android sslpinning disable` |
