# SAN Network Raspberry Pi Feeder and VHF Radio Guide

Build a ground station that receives ADS-B broadcasts from aircraft and AIS position reports from vessels, then feeds the data to SAN Network. This guide also covers receiving VHF voice communications with software-defined radio (SDR) and streaming audio to SAN Radio.

## Table of Contents

- [ADS-B and AIS Feeder Setup](#ads-b-and-ais-feeder-setup)
- [VHF Radio Reception and Streaming](#vhf-radio-reception-and-streaming)

## ADS-B and AIS Feeder Setup

### Hardware Requirements

| Component | Requirements |
| --- | --- |
| Computer | Raspberry Pi 3B+, 4, or 5 running 64-bit Raspberry Pi OS Lite |
| SDR receiver | RTL2832U-based USB dongle, such as an RTL-SDR Blog V3/V4 or FlightAware Pro Stick |
| ADS-B antenna | 1090 MHz vertical antenna |
| AIS antenna | 162 MHz antenna suitable for the VHF marine band |
| Connections | SMA adapters and quality 50-ohm coaxial cable |
| Power supply | Official Raspberry Pi power supply rated for your model, with a minimum of 5 V / 3 A as specified in the source guide |

### 1. Prepare the Raspberry Pi

Connect to the Raspberry Pi over SSH, update the system, and install the required packages.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl build-essential rtl-sdr librtlsdr-dev
```

Blacklist the default DVB-T driver to prevent it from claiming the RTL-SDR dongle, then reboot.

```bash
echo "blacklist dvb_usb_rtl2832u" | sudo tee /etc/modprobe.d/nortl.conf
sudo reboot
```

### 2. Install the ADS-B Decoder

After the Raspberry Pi restarts, reconnect over SSH and install `readsb`.

```bash
bash -c "$(curl -sL https://raw.githubusercontent.com/wiedehopf/adsb-scripts/master/readsb-install.sh)"
```

Set your station coordinates and antenna altitude for multilateration (MLAT).

```bash
sudo readsb-set-location 38.32 26.30 120
```

Replace `38.32` with your latitude, `26.30` with your longitude, and `120` with your antenna altitude in meters.

### 3. Install the AIS Decoder

Build `rtl-ais` to decode marine AIS signals on 161.975 MHz and 162.025 MHz.

```bash
git clone https://github.com/dgiardini/rtl-ais.git
cd rtl-ais
make
sudo cp rtl_ais /usr/local/bin/
```

Create a systemd unit to run the AIS decoder in the background.

```bash
sudo nano /etc/systemd/system/ais-feeder.service
```

Add the following unit configuration.

```ini
[Unit]
Description=SAN Network AIS Feeder Service
After=network.target

[Service]
ExecStart=/usr/local/bin/rtl_ais -h 127.0.0.1 -p 10110 -n
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable the service at boot and start it now.

```bash
sudo systemctl enable --now ais-feeder
```

### 4. Connect the SAN Network Feeder Client

1. Sign in to [SAN Network](https://sannetwork.net).
2. Open **Dashboard → Claim Station** and generate a Feed Ingestion Key. The key follows the format `san_feed_live_xxx`.
3. Install the SAN Network Feeder client.

   ```bash
   bash -c "$(curl -sL https://sannetwork.net/install-feeder.sh)"
   ```

4. When prompted, enter your feed API key, station latitude and longitude, and station name.

### 5. Verify Your Station

1. Open the [SAN Network Dashboard](https://sannetwork.net/dashboard).
2. Under **Station Status**, confirm that your station shows **Online**.
3. Open your personal live map layer to view aircraft and vessels received by your station.

## VHF Radio Reception and Streaming

Use an SDR receiver to pick up aviation or marine VHF voice communications and stream the audio to SAN Radio. The example below configures an aviation audio feed.

### Hardware and Software Requirements

| Component | Requirements |
| --- | --- |
| SDR receiver | RTL-SDR Blog V3/V4, Airspy R2, or HackRF One |
| Antenna | Dipole or ground-plane antenna designed for the VHF aviation band |
| Computer | Linux computer running Ubuntu or Debian, or a Raspberry Pi |
| Software | `rtl_fm`, RTLSDR-Airband, Icecast, or SAN Radio Audio Streamer, depending on your setup |

### 1. Select a VHF Frequency

| Band | Frequency range | Modulation | Examples |
| --- | --- | --- | --- |
| Aviation | 118.000–136.975 MHz | AM | Tower, approach, and radar communications |
| Marine | 156.000–174.000 MHz | Narrowband FM (NFM) | Channel 16 at 156.800 MHz for distress and calling |

### 2. Build RTLSDR-Airband

Install the build dependencies, clone the repository, and compile the software.

```bash
sudo apt update
sudo apt install -y build-essential cmake libmp3lame-dev libshout3-dev \
  libconfig++-dev libfftw3-dev libpulse-dev
git clone https://github.com/szpajder/RTLSDR-Airband.git
cd RTLSDR-Airband
mkdir build && cd build
cmake .. -DPLATFORM=generic
make
sudo make install
```

### 3. Configure the Audio Stream

Open the RTLSDR-Airband configuration file.

```bash
sudo nano /usr/local/etc/rtl_airband.conf
```

Use the following example to scan a list of aviation frequencies and send the received audio to SAN Radio through Icecast. Replace the example frequencies, station code, stream name, and stream key with your own values.

```text
devices: (
  {
    type = "rtlsdr";
    index = 0;
    gain = 25;
    correction = 0;
    mode = "scan";

    channels: (
      {
        # Frequencies to monitor, in MHz.
        freqs = ( 118.100, 121.500, 128.300 );
        labels = ( "TOWER", "EMERGENCY", "APPROACH" );

        outputs: (
          {
            type = "icecast";
            server = "radio.sannetwork.net";
            port = 8000;
            mount = "vhf_your_station_code.mp3";
            password = "YOUR_SAN_RADIO_STREAM_KEY";
            name = "Ankara Esenboğa TWR / SAN Radio Feed";
            genre = "Aviation Live Audio";
          }
        );
      }
    );
  }
);
```

### 4. Start the Live Stream

Run RTLSDR-Airband in the foreground to test the configuration.

```bash
rtl_airband -c /usr/local/etc/rtl_airband.conf -F
```

When a received signal exceeds the squelch threshold, the software sends audio to the configured SAN Radio server.

Stop the foreground test with `Ctrl+C`, then enable and start the background service.

```bash
sudo systemctl enable --now rtl_airband
```

### 5. Monitor the Stream

1. Sign in to the [SAN Radio dashboard](https://radio.sannetwork.net).
2. Open **VHF Audio Streams** and select the stream that matches your station code.
3. Use the live console to monitor audio latency and current listener statistics.
