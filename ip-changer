#!/bin/bash
# change-proxmox-ip.sh
# Interactive helper to change the IP of a Proxmox cluster node safely.
# Run this from console access (not SSH), otherwise you may lock yourself out.

set -e

echo "=== Proxmox Cluster Node IP Changer ==="
echo "!! WARNING: Run from local console, not SSH !!"
echo

# 1. Ask for new IP
read -p "Enter the new IP address for this node: " NEW_IP
read -p "Enter the CIDR netmask (e.g. 24 for /24): " CIDR
read -p "Enter the gateway (leave blank if unchanged): " GATEWAY

# 2. Detect hostname
HOSTNAME=$(hostname)
echo "Detected hostname: $HOSTNAME"

# 3. Backup configs
BACKUP_DIR="/root/ip-change-backup-$(date +%F-%H%M%S)"
mkdir -p "$BACKUP_DIR"
cp /etc/network/interfaces "$BACKUP_DIR"/interfaces.bak
cp /etc/pve/corosync.conf "$BACKUP_DIR"/corosync.conf.bak
cp /etc/hosts "$BACKUP_DIR"/hosts.bak

echo "Backups saved to $BACKUP_DIR"
echo

# 4. Update /etc/network/interfaces
echo "Updating /etc/network/interfaces..."
IFACE_FILE="/etc/network/interfaces"
sed -i "s/address .*/address $NEW_IP\/$CIDR/" $IFACE_FILE
if [ -n "$GATEWAY" ]; then
    sed -i "s/gateway .*/gateway $GATEWAY/" $IFACE_FILE
fi

# 5. Update corosync config
echo "Updating /etc/pve/corosync.conf..."
sed -i "s/ring0_addr: .*/ring0_addr: $NEW_IP/" /etc/pve/corosync.conf

# 6. Update /etc/hosts
echo "Updating /etc/hosts..."
if grep -q "$HOSTNAME" /etc/hosts; then
    sed -i "s/^[0-9.]\+\s\+$HOSTNAME/$NEW_IP   $HOSTNAME/" /etc/hosts
else
    echo "$NEW_IP   $HOSTNAME" >> /etc/hosts
fi

# 7. Show diff to user
echo
echo "=== Changes preview ==="
echo "New IP: $NEW_IP/$CIDR"
if [ -n "$GATEWAY" ]; then
    echo "New Gateway: $GATEWAY"
fi
echo
read -p "Apply and restart networking & corosync? (y/n): " CONFIRM

if [[ "$CONFIRM" =~ ^[Yy]$ ]]; then
    echo "Restarting networking..."
    systemctl restart networking || true
    echo "Restarting corosync..."
    systemctl restart corosync || true
    echo
    echo "Done. Run 'pvecm status' to confirm cluster health."
else
    echo "Aborted. Restored backup files in $BACKUP_DIR."
    cp "$BACKUP_DIR"/interfaces.bak /etc/network/interfaces
    cp "$BACKUP_DIR"/corosync.conf.bak /etc/pve/corosync.conf
    cp "$BACKUP_DIR"/hosts.bak /etc/hosts
fi
