# Scrypted HomeKit Integration

Self-hosted HomeKit integration for IP cameras and smart home devices.

## Overview

- **Cameras**: Eufy Indoor, Amcrest, TP-Link Tapo
- **Network**: 192.168.1.x home network
- **HomeKit**: Full bridge mode with camera streaming
- **Processing**: CPU-only (GPU disabled)

## Network Configuration

### Key Points

- **Network Mode**: `host` - Required for proper HomeKit mDNS advertisement
- **IP**: 192.168.1.66 (cube server)
- **mDNS**: Uses host's avahi-daemon for discovery
- **Ports**:
  - 10443: HTTPS web interface (HomeKit accessory protocol)
  - 11080: HTTP web interface (debug/UI)
  - Dynamic HAP ports: ~41000-48000 (allocated per accessory)

### Why Host Networking?

HomeKit Accessory Protocol (HAP) requires:
1. mDNS advertisement on physical network (not Docker bridge)
2. HAP accessory ports accessible from home network (not private Docker IPs)
3. Container using host's network stack directly

Bridge networking (standard Docker) would advertise `10.0.1.x` private IP → HomeKit clients cannot connect.

## Configuration

All settings stored in `/opt/scrypted-data/scrypted.db/` (LevelDB):
- Camera credentials and RTSP URLs
- HomeKit pairings and device mappings
- Plugin settings and device list

**This persists automatically** - just mount the volume to migrate settings.

## Cameras Configured

| Camera | IP | Resolution | RTSP Stream |
|--------|----|-----------:|-------------|
| Eufy Indoor | 192.168.1.110 | 1920x1080 | `rtsp://Gm9alqGSRwki:Nh3gY9RDso2z@192.168.1.110/live0` |
| Amcrest Front Porch Left | 192.168.1.111 | 3840x2160 (4K) | `rtsp://admin:pepperFiber..@192.168.1.111/cam/realmonitor?channel=1&subtype=0` |
| TP-Link Tapo C120 | 192.168.1.151 | 2560x1440 (2K) | `rtsp://drose123:pepperFiber..@192.168.1.151:554/stream1` |

**Note**: Amcrest plugin has "Automatically Configure Settings" **disabled** (requires digest auth which plugin cannot handle).

## Deployment

### Via Coolify

1. Create new Coolify project
2. Select "Git Repository"
3. GitHub: `https://github.com/cipher982/scrypted`
4. Branch: `main`
5. Compose file: `docker-compose.yml`
6. Do NOT enable GPU support (running CPU-only)

### Via Docker Compose (Standalone)

```bash
ssh cube
cd /opt/scrypted && docker-compose up -d
```

### Via Docker CLI

```bash
docker run -d \
  --name scrypted-homekit \
  --network host \
  --restart unless-stopped \
  --cap-add SYS_ADMIN \
  --device=/dev/fuse \
  --security-opt apparmor:unconfined \
  --ulimit nofile=1024:1024 \
  -e SCRYPTED_DOCKER_AVAHI=true \
  -e SCRYPTED_ADVERTISE_ADDRESS=192.168.1.66 \
  -v /opt/scrypted-data:/server/volume \
  -v /run/dbus:/run/dbus:ro \
  -v /run/avahi-daemon:/run/avahi-daemon:ro \
  ghcr.io/koush/scrypted:latest
```

## HomeKit Pairing

### Initial Setup

1. Open Home app on iPhone/iPad
2. Tap "+" → "Add Accessory"
3. Select "Scrypted" from HomeKit accessories list
4. Scan QR code (or enter 8-digit PIN)
5. Assign to Home, Room
6. Configure automation/scenes as needed

### Troubleshooting

**Devices not appearing in HomeKit:**
- Verify mDNS advertisement: `avahi-browse -r _hap._tcp`
- Should show: `address = [192.168.1.66]` (not Docker IP)
- Check network connectivity: `ping 192.168.1.66` from iPhone

**Connection timeout after discovery:**
- Verify firewall allows port range 41000-48000 from 192.168.1.x
- Check UFW status: `sudo ufw status`
- Rule needed: `Anywhere ALLOW 192.168.1.0/24`

**HomeKit showing "No Response":**
- Device may need re-pairing (HomeKit limitation with IP changes)
- Remove accessory from Home app, re-add with new QR code

## Logs

View real-time logs:
```bash
docker logs -f scrypted-homekit
```

Check for HomeKit-specific messages:
```bash
docker logs scrypted-homekit | grep -i 'homekit\|published\|mdns'
```

## Backup & Recovery

Scrypted configuration is backed up daily via Kopia:
```bash
# Verify backup includes scrypted.db
sudo kopia snapshot show <latest-snapshot-id> | grep scrypted.db

# Restore if needed
sudo kopia snapshot restore <snapshot-id> \
  --target=/tmp/scrypted-restore \
  /opt/scrypted-data/scrypted.db/
```

## Performance Notes

- NVIDIA GPU enabled for video processing acceleration
- Max heap size: 4GB Node.js process
- Recommended: Dedicated Ethernet connection (WiFi unreliable for HomeKit)
- Keep cube server on stable 192.168.1.66 IP (DHCP reservation recommended)

## Related Documentation

- Scrypted Docs: https://docs.scrypted.app/
- HomeKit Accessory Protocol: https://developer.apple.com/homekit/
- Cube Infrastructure: ~/git/mytech/infrastructure/crestwood.md

## Changelog

### 2025-10-21

- **Migration from Coolify**
  - Changed from bridge networking to host networking
  - Enabled SCRYPTED_DOCKER_AVAHI for proper mDNS
  - Added explicit ADVERTISE_ADDRESS configuration
  - Added health checks
  - Added logging configuration

- **Network Fix**
  - Fixed Docker bridge IP routing conflict (removed 192.168.1.67 secondary IP)
  - Verified RTSP camera connectivity restored
  - Stop Sign Nanny feed operational

- **HomeKit Investigation**
  - Identified root cause: Coolify's bridge networking incompatible with HomeKit
  - Solution: Host networking with GitHub-backed compose
  - Configuration preserved via `/opt/scrypted-data` volume mount
