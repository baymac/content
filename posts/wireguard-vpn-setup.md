---
title: 'Setting Up WireGuard VPN with QR Code for Phone Connection'
date: '2025-01-27'
tags: vpn, wireguard, networking, security, server, mobile, qr-code
ai-gen: true
---

My gf wanted to use TikTok but its banned in India. I had a VPS in the US region, so setting up a VPN would help her access TikTok. I was tired to do any setup because I was calling it a night. I opened up Warp and logged into my VPS and asked it to setup the VPN using wireguard and give me a QR code which I can share with my gf to connect. It took me less than 4 minutes and the VPN was ready. :)

## Why WireGuard?

Before diving into the setup, let's understand why WireGuard is superior to traditional VPN solutions:

- **Performance**: Minimal overhead compared to OpenVPN or IPSec
- **Security**: Simpler codebase means fewer attack vectors
- **Ease of Use**: Simple configuration files and easy key management
- **Cross-Platform**: Works on Linux, macOS, Windows, iOS, and Android
- **Kernel Implementation**: Runs in the kernel for better performance

## Prerequisites

- A VPS or server with a public IP address
- Ubuntu 20.04+ or Debian 11+ (this guide uses Ubuntu)
- Root or sudo access
- Basic command line knowledge

## Step 1: Server Setup

First, let's update the system and install WireGuard:

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install WireGuard and required tools
sudo apt install wireguard qrencode -y

# Enable IP forwarding
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Step 2: Generate Server Keys

Create the WireGuard directory and generate the server's private and public keys:

```bash
# Create WireGuard directory
sudo mkdir -p /etc/wireguard
cd /etc/wireguard

# Generate server private key
wg genkey | sudo tee privatekey | wg pubkey | sudo tee publickey

# Set proper permissions
sudo chmod 600 privatekey
sudo chmod 644 publickey
```

## Step 3: Create Server Configuration

Create the main WireGuard configuration file:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Add the following configuration (replace `YOUR_SERVER_IP` with your actual server IP):

```ini
[Interface]
PrivateKey = <server_private_key>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
SaveConfig = true
```

**Important Notes:**

- Replace `<server_private_key>` with the content of your `privatekey` file
- Adjust the `Address` subnet if needed (10.0.0.1/24 gives you 254 usable IPs)
- Change `eth0` to your actual network interface (use `ip addr show` to find it)

## Step 4: Generate Client Keys

For each client (phone, laptop, etc.), you'll need to generate a key pair:

```bash
# Create client directory
sudo mkdir -p /etc/wireguard/clients
cd /etc/wireguard/clients

# Generate client keys (replace 'phone' with descriptive names)
wg genkey | sudo tee phone_privatekey | wg pubkey | sudo tee phone_publickey
```

## Step 5: Add Client to Server Configuration

Add the client configuration to your server's `wg0.conf`:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Add this section at the end:

```ini
[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
```

## Step 6: Create Client Configuration

Create a client configuration file:

```bash
sudo nano /etc/wireguard/clients/phone.conf
```

Add this configuration:

```ini
[Interface]
PrivateKey = <client_private_key>
Address = 10.0.0.2/24
DNS = 1.1.1.1, 8.8.8.8

[Peer]
PublicKey = <server_public_key>
Endpoint = <server_public_ip>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

**Replace the placeholders:**

- `<client_private_key>`: Content of `phone_privatekey`
- `<server_public_key>`: Content of server's `publickey`
- `<server_public_ip>`: Your server's public IP address

## Step 7: Generate QR Code

Now for the magic! Generate a QR code that your phone can scan:

```bash
# Generate QR code from client config
qrencode -t ansiutf8 < /etc/wireguard/clients/phone.conf
```

This will display a QR code in your terminal that you can scan with your phone.

## Step 8: Start WireGuard Service

Enable and start the WireGuard service:

```bash
# Enable WireGuard to start on boot
sudo systemctl enable wg-quick@wg0

# Start the service
sudo systemctl start wg-quick@wg0

# Check status
sudo systemctl status wg-quick@wg0
```

## Step 9: Phone Setup

### Android

1. Install WireGuard from Google Play Store
2. Tap the "+" button
3. Choose "Scan from QR code"
4. Scan the QR code displayed in your terminal
5. Tap "Create tunnel"
6. Enable the tunnel

### iOS

1. Install WireGuard from App Store
2. Tap "Add a tunnel"
3. Choose "Create from QR code"
4. Scan the QR code
5. Tap "Add tunnel"
6. Enable the tunnel

## Step 10: Testing the Connection

Test your VPN connection:

```bash
# Check WireGuard status
sudo wg show

# Test connectivity from client
ping 10.0.0.1
```

## Advanced Configuration

### Multiple Clients

For additional clients, repeat the key generation and configuration steps, incrementing the IP addresses:

```ini
# Client 2
[Peer]
PublicKey = <client2_public_key>
AllowedIPs = 10.0.0.3/32

# Client 3
[Peer]
PublicKey = <client3_public_key>
AllowedIPs = 10.0.0.4/32
```

### Split Tunneling

If you only want specific traffic to go through the VPN, modify the client's `AllowedIPs`:

```ini
# Only route specific subnets through VPN
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24

# Route all traffic through VPN (default)
AllowedIPs = 0.0.0.0/0, ::/0
```

### Custom DNS

You can specify custom DNS servers for VPN clients:

```ini
# In client config
DNS = 1.1.1.1, 8.8.8.8, 208.67.222.222
```

## Security Considerations

1. **Firewall Rules**: Ensure your server's firewall allows UDP port 51820
2. **Key Management**: Keep private keys secure and never share them
3. **Regular Updates**: Keep WireGuard and your system updated
4. **Monitoring**: Monitor logs for unusual activity

## Troubleshooting

### Common Issues

**Connection fails:**

- Check if WireGuard service is running: `sudo systemctl status wg-quick@wg0`
- Verify firewall rules: `sudo ufw status`
- Check server logs: `sudo journalctl -u wg-quick@wg0`

**QR code not scanning:**

- Ensure the terminal supports UTF-8
- Try copying the config file content and generating QR code online
- Check if the config file is properly formatted

**Client can't reach internet:**

- Verify IP forwarding is enabled: `cat /proc/sys/net/ipv4/ip_forward`
- Check iptables rules: `sudo iptables -L -n -v`

## Performance Optimization

WireGuard is already quite fast, but you can optimize further:

```bash
# Optimize network settings
echo 'net.core.rmem_max=134217728' | sudo tee -a /etc/sysctl.conf
echo 'net.core.wmem_max=134217728' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem=4096 87380 134217728' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem=4096 65536 134217728' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## Monitoring and Maintenance

### View Connected Clients

```bash
sudo wg show
```

### View Real-time Traffic

```bash
sudo watch -n 1 'wg show'
```

### Backup Configuration

```bash
sudo cp -r /etc/wireguard /backup/wireguard-$(date +%Y%m%d)
```

## Final Thoughts

WireGuard with QR code configuration provides a seamless VPN experience that's both secure and user-friendly. The setup process might seem complex initially, but once configured, it's incredibly reliable and fast.

The QR code feature is particularly useful for:

- Quickly adding new devices
- Sharing configurations with team members
- Avoiding manual configuration errors
- Streamlining the onboarding process

_This guide covers the basics of WireGuard setup. For production environments, consider additional security measures like certificate-based authentication and network segmentation._
