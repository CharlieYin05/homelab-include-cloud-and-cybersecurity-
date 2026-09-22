# Add an Exit Node for Tailnet

***2026-09-22***

---

## Add an Exit Node in Tailnet that admin's device can use

1. Add exit node in `cy-server`:
```
sudo tee /etc/sysctl.d/99-tailscale.conf >/dev/null <<'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

sudo tailscale set --advertise-exit-node

sudo tailscale up
```

2. Approve it in admin console

3. Append a policy to allow `group:admin` to use the exit node
```
	"grants": [
		{
			"src": ["group:admin"],
			"dst": ["autogroup:internet"],
			"ip":  ["*"],
		},
```
