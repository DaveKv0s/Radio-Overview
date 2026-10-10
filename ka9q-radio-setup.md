**Title:** elaborate - Ask Brave

**Source:** [https://search.brave.com/ask?q=elaborate&source=chatllm-elaborate&conversation=09aaf887fe1430a76ef07373e84cf9d989be](https://search.brave.com/ask?q=elaborate&source=chatllm-elaborate&conversation=09aaf887fe1430a76ef07373e84cf9d989be)

---

# Page Structure Map
```text
elaborate - Ask Brave
├── 1\. Install Build Dependencies
├── 2\. Clone, Build, and Install
├── 3\. Set User Permissions
├── 4\. Create a Configuration File
├── 5\. Run radiod
│   ├── Via systemd (recommended)
│   └── Directly from the command line
├── 6\. Generate FFTW Wisdom (Performance)
└── 7\. Verify It's Working
```

---

1\. Install Build Dependencies 2\. Clone, Build, and Install 3\. Set User Permissions 4\. Create a Configuration File 5\. Run radiod 6\. Generate FFTW Wisdom (Performance) 7\. Verify It's Working

Encrypted & Private History

Your chat history is encrypted and auto-deleted after 24 hours of inactivity by default. The encryption key is stored locally on your device. Brave does not retain your IP address.

Learn more

once ka9q-radio is install from the github what step are needed to make radiod runnable from the commandline

After installing **ka9q-radio** from GitHub, the following steps are required to make `radiod` runnable and operational from the command line:

1. **Compile and Install**: Navigate to the source directory and run `make` followed by `sudo make install`. This creates the binary at `/usr/local/sbin/radiod` and installs the systemd service files. For specific hardware like SDRPlay, append `ENABLE_SDRPLAY=1` to the make commands.
2. **Create Configuration File**: Create a configuration file (e.g., `hf.conf`) in the `/etc/radio/` directory. The file name must follow the pattern `radiod@<instance_name>.conf` (e.g., `/etc/radio/radiod@hf.conf`), where `hf` is the descriptive instance name. Note that `.conf` is not part of the instance name.
3. **Set Permissions**: Ensure the user running `radiod` has read access to the configuration file in `/etc/radio/`. Add your user to the `radio` group using `sudo usermod -aG radio $(whoami)` to avoid needing root privileges for configuration edits.
4. **Verify with Direct Execution**: You can test the setup by running `radiod -v /etc/radio/radiod@hf.conf` directly from the command line. This verbose mode prints any errors and confirms if the demodulators start correctly.
5. **Optional: Optimize FFTW**: Run the FFTW calibration process described in `docs/FFTW3.md` to generate a "wisdom" file in `/var/lib/ka9q-radio/`. This step is critical for performance and data integrity, though not strictly required for the binary to execute.
```bash
# Example direct execution command
radiod -v /etc/radio/radiod@hf.conf

# Verify connectivity from another terminal
control
```

elaborate

Here's a detailed walkthrough of every step from a clean clone to a running `radiod` instance:

---

## 1\. Install Build Dependencies

Before compiling, install the required libraries. On Debian/Ubuntu:

```bash
sudo apt install git avahi-utils build-essential make gcc \
  libairspy-dev libairspyhf-dev libavahi-client-dev libbsd-dev \
  libfftw3-dev libhackrf-dev libiniparser-dev libncurses5-dev \
  libopus-dev librtlsdr-dev libusb-1.0-0-dev libusb-dev \
  portaudio19-dev libasound2-dev uuid-dev rsync libogg-dev \
  libsamplerate-dev libliquid-dev libncursesw5-dev libbladerf-dev
```

Hardware-specific extras:

- **SDRPlay**: Install the SDRPlay API **before** building, then pass `SDRPLAY=1` to `make`.
- **Fobos**: Build `libfobos` from source first, then pass `FOBOS=1` to `make`.
- **Raspberry Pi**: Add `libpigpio-dev`.

## 2\. Clone, Build, and Install

```bash
git clone https://github.com/ka9q/ka9q-radio.git
cd ka9q-radio
make -j
sudo make install
```

For SDRPlay: `make -j SDRPLAY=1` and `sudo make install SDRPLAY=1`.

`make install` places files in these locations:

| Path | Contents |
| --- | --- |
| `/usr/local/sbin/` | Daemon binaries (`radiod`) |
| `/usr/local/bin/` | User programs (`control`, `monitor`) |
| `/usr/local/share/ka9q-radio/` | Support files (e.g., `presets.conf`) |
| `/var/lib/ka9q-radio/` | State files (FFTW wisdom) |
| `/etc/systemd/system/` | systemd unit files (`radiod@.service`) |
| `/etc/sysctl.d/` | Socket buffer tuning (`98-sockbuf.conf`) |
| `/etc/udev/rules.d/` | Device rules (e.g., `52-airspy.rule`) |
| `/etc/radio/` | Config templates (won't overwrite existing) |

## 3\. Set User Permissions

Add your user to the `radio` group so you can edit config files without `sudo`:

```bash
sudo usermod -aG radio $(whoami)
```

**Reboot** (or log out/in) for the group change to take effect.

Also ensure the state directory is owned by the user who will run `radiod`:

```bash
sudo chown <username> /var/lib/ka9q-radio
```

## 4\. Create a Configuration File

`radiod` reads its config from `/etc/radio/radiod@<instance>.conf`, where `<instance>` is a descriptive name you choose (e.g., `hf`, `vhf`, `airspy-autorx`). The `.conf` extension is **not** part of the instance name.

A minimal config for an Airspy in FM mode looks like:

```ini
[global]
hardware = airspy
mode = fm
status = sonde.local
iface = lo
ttl = 0
data = sonde-pcm.local

[airspy]
device = airspy
description = "auto_rx"
frequency = 407m0
samprate = 20000000
```

Key `[global]` options:

- **`hardware`** – which SDR driver to use
- **`mode`** – default demodulation mode (`fm`, `usb`, `am`, `cw`, `iq`, etc.)
- **`status`** – mDNS hostname for status/control channel
- **`data`** – mDNS hostname for audio/IQ data output
- **`ttl`** – multicast TTL (0 = local only, 1+ = crosses routers)

You can also place additional `.conf` snippets in a subdirectory named `radiod@<instance>.conf.d/` (e.g., `01-extra.conf`), which are read after the main file, sorted by name.

## 5\. Run radiod

### Via systemd (recommended)

```bash
sudo systemctl enable radiod@<instance>
sudo systemctl start radiod@<instance>
```

Check status and logs:

```bash
sudo systemctl status radiod@<instance>
journalctl -u radiod@<instance> -f
```

### Directly from the command line

```bash
radiod -v /etc/radio/radiod@<instance>.conf
```

The `-v` flag enables verbose output so you can see demodulator startup and any errors. This runs in the foreground; press `Ctrl+C` to stop.

## 6\. Generate FFTW Wisdom (Performance)

FFTW (Fastest Fourier Transform in the West) is used for the digital downconverters. On first run it will be slow until "wisdom" files are generated. Run the calibration described in `docs/FFTW3.md` to populate `/var/lib/ka9q-radio/` with optimized plan files. On a Raspberry Pi this can take **hours**; on a desktop it's typically under an hour. Without this, `radiod` will still run but with significantly higher CPU usage and potential underruns.

## 7\. Verify It's Working

From another terminal, use the `control` utility to connect to the running instance:

```bash
control
```

This connects to the `status` mDNS name defined in your config and lets you query status, tune frequencies, and check demodulator levels. You can also verify multicast traffic with `tcpdump` or `wireshark` on the interface specified by `iface`.

---

**Common gotchas:**

- **Wrong instance name**: `systemctl start radiod@hf.conf` will fail — it should be `radiod@hf` (no `.conf`).
- **SDRPlay not detected**: You must install the SDRPlay API *before* building, and pass `SDRPLAY=1` to both `make` and `make install`.
- **Config overwritten on update**: Back up your `/etc/radio/radiod@*.conf` files before re-running `make install`, as templates may overwrite them.
- **Multicast not reaching other machines**: Ensure `ttl` is set appropriately and that mDNS/Avahi is running (`avahi-daemon`).