
# Wireless Home Survey 

## wlan survey daemon 

A lightweight, high-performance network automation daemon written in Go designed specifically for the **FreeBSD network stack**. 

This module continuously monitors ambient radio frequency (RF) metrics and dynamically adjusts wireless connection priorities within `wpa_supplicant` based on real-time Received Signal Strength Indicators (RSSI). It is built to solve routing priority conflicts when roaming near high-density public ESSIDs (like `#PUBLIC-WIFI`) without interrupting active socket connections.

## Features

- **FreeBSD Native Engine**: Utilizes kernel-space wireless scanning via native `ifconfig` abstractions.
- **Dynamic Hysteresis Matrix**: Adjusts network priority mapping automatically between 3 operational tiers (High, Moderate, Low/Absent).
- **Atomic File Subsystem**: Guarantees system configuration integrity against power loss by writing to a temporary file buffer before issuing a POSIX `rename()` system call.
- **Hot-Reload Syncing**: Triggers active configuration updates in-flight via `wpa_cli reconfigure`, eliminating interface teardown or connection drops.
- **Resource Optimized**: Single compiled, statically linked binary with minimal CPU footprint ($\approx$ 0.01%) and a light memory profile.

---

## Technical Architecture

[ Timer Loop (30s) ] ──> [ ifconfig wlan0 list scan ] ──> [ In-Memory RSSI Parser ]
│
[ wpa_cli reconfigure ] <── [ Atomic File Swap ] <── [ State Matrix Evaluation ]

---

## Installation & Compilation

Since this daemon targets the FreeBSD platform-specific text layouts of `ifconfig`, compile the binary directly on your target machine:

```sh
# Initialize the Go workspace module
go mod init wifi_survey

# Compile and optimize the production binary
go build -ldflags="-s -w" -o wifi_survey main.go

# Relocate the executable to system binaries
mv wifi_survey /usr/local/sbin/

```
## Command Line Arguments

The daemon exposes modular configurations directly via command flags. Default values fall back to standard FreeBSD wireless configurations.

| Flag | Parameter | Default Value | Description |
| :--- | :--- | :--- | :--- |
| **`-i`** | `string` | `wlan0` | Virtual wireless interface identifier |
| **`-s`** | `string` | `#Airport_Free_Wifi` | Target ESSID/SSID string pattern to evaluate |
| **`-c`** | `string` | `/etc/wpa_supplicant.conf` | Path to the active `wpa_supplicant` configuration |
| **`-t`** | `int` | `30` | Interval polling sequence frequency (in seconds) |
| **`-h`** | `N/A` | `N/A` | Display the help diagnostic submenu |

### Usage Examples

```sh
# 1. Run with standard system defaults
/usr/local/sbin/wifi_survey

# 2. Target a distinct interface with aggressive 10-second polling frequencies
/usr/local/sbin/wifi_survey -i wlan0 -t 10

# 3. Monitor a completely different open fallback network zone
/usr/local/sbin/wifi_survey -i wlan0 -s "Airport_Free_Wifi" -c /usr/local/etc/wpa_supplicant.conf
```

## Operational Mechanics

The internal state engine evaluates target infrastructure metrics against fixed mathematical boundaries:

* **$\ge$ -50 dBm (Excellent Signal):** Assigns `priority=10`. Forces the client to instantly favor this network over any local private configurations.
* **-51 to -74 dBm (Moderate Signal):** Assigns `priority=3`. Downgrades the posture to an alternate auxiliary state.
* **$\le$ -75 dBm (Attenuated/Failing):** Assigns `priority=1`. Safely demotes the interface to allow fallback onto home/office networks.

## Production Deployment

To run the survey application safely as a persistent background daemon, route stdout/stderr channels into system logging structures:

```sh
nohup /usr/local/sbin/wifi_survey -i wlan0 -s "Airport_Free_Wifi" > /var/log/wifi_survey.log 2>&1 &
```

To review operational transitions in real time:

``sh
tail -f /var/log/wifi_survey.log
``
