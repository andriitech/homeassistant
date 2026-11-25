# Home Assistant

Self-hosted Home Assistant stack (Container install) for an old laptop running Debian 64-bit.
The Samsung X120 has a legacy BIOS, so Home Assistant OS cannot boot; this runs each part in Docker instead.

| Service | What it does | Address |
|---|---|---|
| homeassistant | Main app | http://<server-ip>:8123 |
| mosquitto | MQTT broker | localhost:1883 |
| zigbee2mqtt | Zigbee radio (ZBT-2 #1) | http://<server-ip>:8080 |
| matter-server | Matter controller | ws://localhost:5580/ws |
| otbr | Thread border router (ZBT-2 #2) | http://localhost:8081 |

## Setup

1. Check the OS is 64-bit: `uname -m` must print `x86_64`.
2. Install Docker:
   ```bash
   curl -fsSL https://get.docker.com | sudo sh && sudo usermod -aG docker $USER
   ```
   Log out and back in.
3. Flash one ZBT-2 with Zigbee firmware and the other with Thread (OpenThread RCP) firmware using https://toolbox.openhome.foundation from Chrome.
4. Plug both in, then:
   ```bash
   git clone https://github.com/andriitech/homeassistant.git ~/homeassistant
   cd ~/homeassistant
   cp .env.example .env
   ls -l /dev/serial/by-id/   # put the two paths into .env
   ip -br link                # put the wired interface name into .env
   docker compose up -d
   ```
5. Zigbee2MQTT: open http://<server-ip>:8080, pick the adapter `/dev/ttyACM0`, MQTT server `mqtt://mosquitto:1883`.
6. Home Assistant at http://<server-ip>:8123, then add integrations:
   - **MQTT**: broker `localhost`, port `1883`
   - **Matter**: `ws://localhost:5580/ws`
   - **OpenThread Border Router**: `http://localhost:8081`
   - Make sure IPv6 is enabled on the router/LAN (needed for Matter over Thread).

The OTBR image's environment variables have changed between versions; if it fails to start, check `docker compose logs otbr` against the current openthread/otbr docs.

## Laptop housekeeping

- Keep running with the lid closed: in `/etc/systemd/logind.conf` set
  `HandleLidSwitch=ignore` and `HandleLidSwitchExternalPower=ignore`, then `sudo systemctl restart systemd-logind`.
- Remove the old battery and run it from a small UPS.
- Use wired Ethernet.

## Updates

```bash
docker compose pull && docker compose up -d && docker image prune -f
```

## Backups

Nightly cron (`crontab -e`), to a USB drive mounted at `/mnt/usb`:

```cron
30 3 * * * cd ~/homeassistant && docker compose stop && tar czf /mnt/usb/ha-$(date +\%F).tgz ha z2m matter otbr mosquitto .env; docker compose start
```
