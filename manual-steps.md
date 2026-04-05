# Manual steps after running playbook

## NordVPN setup

```bash
nordvpn login --token YOUR_TOKEN
nordvpn set lan-discovery enable
nordvpn set autoconnect on
nordvpn connect
```

## Verify NordVPN

```bash
nordvpn status
curl http://ipinfo.io
```
