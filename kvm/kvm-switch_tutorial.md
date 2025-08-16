# Rancher Desktop + VirtualBox on Ubuntu 24.04

This tutorial goes through solving issues that a user will have if trying tools that requires kvm modules to be enabled or disabled for Rancher Desktop and VirtualBox on Ubuntu Linux. You’ll be able to switch cleanly between Rancher Desktop (KVM/QEMU) and VirtualBox, and this tool works on AMD or Intel; the script auto-detects `kvm_amd` vs `kvm_intel`.


## Table of Contents

* [Firmware (one-time)](#firmware-one-time)
* [Fix KVM persistence (blacklists, autoload, perms, group)](#1-fix-kvm-persistence-blacklists-autoload-perms-group)
* [Load now and verify](#2-load-now-and-verify)
* [Install the toggle script](#3-install-the-toggle-script)
  * [Option A (recommended): `~/kvm-switch.sh`](#option-a-recommended-kvm-switchsh)
  * [Option B: custom folder](#option-b-custom-folder)
* [Add the `kvms-*` wrappers](#4-add-the-kvms--wrappers)
  * [Wrappers for Option A (symlink into `~`)](#wrappers-for-option-a-symlink-into-)
  * [Wrappers for Option B (custom path)](#wrappers-for-option-b-custom-path)
* [Usage](#5-usage)
* [Troubleshooting & tips](#troubleshooting--tips)

---

## Firmware (one-time)

Enable virtualization in BIOS/UEFI and cold boot the machine:

* **AMD:** SVM Mode = **Enabled** (IOMMU optional)
* **Intel:** Intel VT-x = **Enabled** (VT-d optional)

---

## 1. Fix KVM persistence (blacklists, autoload, perms, group)

```bash
# Show every file that blacklists kvm/kvm_amd/kvm_intel
sudo grep -R --line-number -E '^\s*(blacklist|install)\s+(kvm(_amd|_intel)?)(\b|$)' \
  /etc/modprobe.d /lib/modprobe.d /usr/lib/modprobe.d || true

# Disable any offenders (renames to .disabled)
sudo bash -c '
for d in /etc/modprobe.d /lib/modprobe.d /usr/lib/modprobe.d; do
  for f in "$d"/*.conf; do
    [ -f "$f" ] || continue
    if grep -qE "^\s*(blacklist|install)\s+kvm(_amd|_intel)?(\b|$)" "$f"; then mv -v "$f" "$f.disabled"; fi
  done
done
'

# Make early-boot honor the change
sudo update-initramfs -u

# Autoload KVM at boot (AMD shown; if Intel, replace kvm_amd with kvm_intel)
printf "kvm\nkvm_amd\n" | sudo tee /etc/modules-load.d/kvm.conf

# Correct /dev/kvm perms and group
echo 'KERNEL=="kvm", GROUP="kvm", MODE="0660"' | sudo tee /etc/udev/rules.d/50-kvm.rules
sudo udevadm control --reload-rules
sudo udevadm trigger

# Add your user to the kvm group
sudo usermod -aG kvm "$USER"
newgrp kvm   # or log out/in once
```

---

## 2. Load now and verify

```bash
# Load modules now (AMD shown; use kvm_intel for Intel)
sudo modprobe -r kvm_amd kvm 2>/dev/null || true
sudo modprobe kvm
sudo modprobe kvm_amd

# Tools used by the script
sudo apt-get update && sudo apt-get install -y cpu-checker lsof psmisc

# Checks
lsmod | grep -E 'kvm(_amd|_intel)?'
ls -l /dev/kvm
kvm-ok   # expect: "KVM acceleration can be used"
```

---

## 3. Install the toggle script

> The script lets you switch between Rancher Desktop and VirtualBox without conflicts and prints a clean status. It auto-detects AMD/Intel.

### Option A (recommended): `~/kvm-switch.sh`

```bash
cat > ~/kvm-switch.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

has() { command -v "$1" >/dev/null 2>&1; }

# --- CPU Check ---
cpu_mod() {
  if lscpu 2>/dev/null | grep -q 'AuthenticAMD'; then echo kvm_amd
  elif lscpu 2>/dev/null | grep -q 'GenuineIntel'; then echo kvm_intel
  else echo kvm_amd; fi
}

# --- Quiet holder detection ---
kvm_pids() {
  { lsof -t -w -n /dev/kvm 2>/dev/null || fuser /dev/kvm 2>/dev/null || true; } \
    | tr ' ' '\n' | grep -E '^[0-9]+$' | sort -u || true
}

kvm_holders_table() {
  if [ ! -e /dev/kvm ]; then echo "(none)"; return 0; fi
  local pids; pids="$(kvm_pids)"
  if [ -z "${pids:-}" ]; then echo "(none)"; return 0; fi
  printf "%-7s %-12s %-18s %-s\n" "PID" "USER" "PROCESS" "SOURCE"
  printf "%-7s %-12s %-18s %-s\n" "-----" "------------" "------------------" "-------------------------"
  while read -r pid; do
    [ -n "$pid" ] || continue
    local user comm args source inst
    user="$(ps -o user= -p "$pid" 2>/dev/null || true)"
    comm="$(ps -o comm= -p "$pid" 2>/dev/null || true)"
    args="$(ps -o args= -p "$pid" 2>/dev/null || true)"
    if echo "$args" | grep -q 'rancher-desktop/lima/'; then
      inst="$(echo "$args" | sed -n 's#.*rancher-desktop/lima/\([^/]*\)/.*#\1#p' | head -n1)"
      source="Rancher Desktop (lima-${inst:-?})"
    elif echo "$comm" | grep -qi 'qemu'; then
      source="QEMU (non-Lima)"
    elif echo "$comm" | grep -qi 'VBox'; then
      source="VirtualBox"
    elif echo "$comm" | grep -qi 'qemu-kvm\|libvirtd\|virtqemud'; then
      source="libvirt"
    else
      source="Other"
    fi
    printf "%-7s %-12s %-18s %-s\n" "$pid" "${user:-?}" "${comm:-?}" "$source"
  done <<< "$pids"
}

wait_kvm_free() {
  local timeout="${1:-15}" i=0
  while [ $i -lt "$timeout" ]; do
    [ -z "$(kvm_pids)" ] && return 0
    sleep 1; i=$((i+1))
  done
  return 1
}

# --- Commands ---
kvm_on()  { sudo modprobe kvm || true; sudo modprobe "$(cpu_mod)" || true; }

# Patched: gracefully release holders, support --force, then unload
kvm_off() {
  local force="${1:-}"
  local MOD="$(cpu_mod)"

  echo "[kvm-off] Releasing holders (if any)…"
  if [ -n "$(kvm_pids)" ]; then
    if has rdctl; then
      echo "[kvm-off] Stopping Rancher Desktop…"
      rdctl shutdown >/dev/null 2>&1 || true
    fi
    if has limactl; then
      limactl list 2>/dev/null | awk 'NR>1 {print $1}' | while read -r inst; do
        [ -n "$inst" ] && limactl stop "$inst" >/dev/null 2>&1 || true
      done
    fi
    if ! wait_kvm_free 10; then
      echo "[kvm-off] Still in use by:"
      kvm_holders_table
      if [ "$force" = "--force" ]; then
        echo "[kvm-off] Killing holders of /dev/kvm…"
        kvm_pids | xargs -r -I{} sudo kill -TERM {} || true
        sleep 3
        kvm_pids | xargs -r -I{} sudo kill -KILL {} || true
      else
        echo "[kvm-off] Aborting. Re-run with: kvms-off --force   (or use: kvms-vbox)"
        return 1
      fi
    fi
  fi

  echo "[kvm-off] Unloading KVM modules…"
  sudo modprobe -r "$MOD" 2>/dev/null || true
  sudo modprobe -r kvm 2>/dev/null || true
  lsmod | grep -E 'kvm(_amd|_intel)?' >/dev/null || echo "[kvm-off] OK: modules unloaded"
}

use_vbox() {
  local force="${1:-}"
  echo "[use-vbox] Stopping Rancher Desktop…"
  has rdctl && rdctl shutdown >/dev/null 2>&1 || true
  if has limactl; then
    limactl list 2>/dev/null | awk 'NR>1 {print $1}' | while read -r inst; do
      [ -n "$inst" ] && limactl stop "$inst" >/dev/null 2>&1 || true
    done
  fi
  echo "[use-vbox] Waiting for /dev/kvm to be free…"
  if ! wait_kvm_free 15; then
    echo "[use-vbox] Still in use by:"; kvm_holders_table
    if [ "$force" = "--force" ]; then
      echo "[use-vbox] Killing holders of /dev/kvm…"
      kvm_pids | xargs -r -I{} sudo kill -TERM {} || true; sleep 3
      kvm_pids | xargs -r -I{} sudo kill -KILL {} || true
    else
      echo "[use-vbox] Re-run with: use-vbox --force"
    fi
  fi
  echo "[use-vbox] Unloading KVM modules for VirtualBox…"
  kvm_off || true
  echo "[use-vbox] KVM modules now:"; lsmod | grep -E 'kvm(_amd|_intel)?' || echo "(none)"
  echo "Ready for VirtualBox."
}

use_rd() {
  if has VBoxManage; then
    echo "[use-rd] Stopping VirtualBox VMs (ACPI → force)…"
    VBoxManage list runningvms 2>/dev/null | sed -n 's/.*\"\(.*\)\".*/\1/p' | \
      while read -r vm; do [ -n "$vm" ] && VBoxManage controlvm "$vm" acpipowerbutton || true; done || true
    sleep 5
    VBoxManage list runningvms 2>/dev/null | sed -n 's/.*\"\(.*\)\".*/\1/p' | \
      while read -r vm; do [ -n "$vm" ] && VBoxManage controlvm "$vm" poweroff || true; done || true
  fi
  echo "[use-rd] Loading KVM modules for Rancher Desktop…"
  kvm_on || true
  if has rdctl; then
    echo "[use-rd] Starting Rancher Desktop…"
    rdctl start >/dev/null 2>&1 || true
  fi
  echo "[use-rd] KVM modules now:"; lsmod | grep -E 'kvm(_amd|_intel)?' || echo "(none)"
  echo "You can start (or are starting) Rancher Desktop now."
}

fix_kvm() {
  local MOD="$(cpu_mod)"
  echo "[fix-kvm] Disabling any KVM blacklists…"
  sudo bash -c '
for d in /etc/modprobe.d /lib/modprobe.d /usr/lib/modprobe.d; do
  for f in "$d"/*.conf; do
    [ -f "$f" ] || continue
    if grep -qE "^\s*(blacklist|install)\s+kvm(_amd|_intel)?(\b|$)" "$f"; then mv -v "$f" "$f.disabled"; fi
  done
done
'
  echo "[fix-kvm] update-initramfs…"; sudo update-initramfs -u
  echo "[fix-kvm] enable auto-load @boot…"; printf "kvm\n%s\n" "$MOD" | sudo tee /etc/modules-load.d/kvm.conf >/dev/null
  echo "[fix-kvm] /dev/kvm perms…"; echo 'KERNEL=="kvm", GROUP="kvm", MODE="0660"' | sudo tee /etc/udev/rules.d/50-kvm.rules >/dev/null
  sudo udevadm control --reload-rules; sudo udevadm trigger
  echo "[fix-kvm] add user to kvm…"; sudo usermod -aG kvm "$USER" || true
  echo "[fix-kvm] load modules now…"; kvm_on
  status
}

status() {
  echo "=== KVM device ==="
  if [ -e /dev/kvm ]; then ls -l /dev/kvm; else echo "/dev/kvm: MISSING"; fi
  echo

  echo "=== Loaded modules ==="
  lsmod | grep -E 'kvm(_amd|_intel)?' || echo "(none)"
  echo

  echo "=== Acceleration check ==="
  if has kvm-ok; then kvm-ok || true; else echo "Install: sudo apt-get install -y cpu-checker"; fi
  echo

  echo "=== Holder(s) of /dev/kvm ==="
  kvm_holders_table
  echo

  echo "=== Rancher Desktop ==="
  if has rdctl; then
    rdctl version 2>/dev/null || true
  else
    echo "rdctl not found"
  fi
  echo

  echo "=== VirtualBox ==="
  if has VBoxManage; then
    VBoxManage -v | sed 's/^/Version: /'
    local running; running="$(VBoxManage list runningvms 2>/dev/null || true)"
    if [ -n "$running" ]; then
      echo "$running"
    else
      echo "(no VMs running)"
    fi
  else
    echo "VBoxManage not found"
  fi
}

case "${1:-status}" in
  status)   status ;;
  who)      kvm_holders_table ;;
  use-vbox) shift || true; use_vbox "${1:-}" ;;
  use-rd)   use_rd ;;
  kvm-on)   kvm_on ;;
  kvm-off)  shift || true; kvm_off "${1:-}" ;;
  fix-kvm)  fix_kvm ;;
  *) echo "Usage: $0 {status|who|use-vbox [--force]|use-rd|kvm-on|kvm-off [--force]|fix-kvm}"; exit 2 ;;
esac
EOF

chmod +x ~/kvm-switch.sh
```

---

### Option B: Custom folder

Use the instructions of this option if you creatted `kvm-switch.sh` in an different path. Change the `SCRIPT_PATH` parameter.

```bash
# Choose your folder (example)
SCRIPT_PATH="$HOME/Documents/scripts"
mkdir -p "$SCRIPT_PATH"

# Save the SAME script content to this path instead of ~/
nano "$SCRIPT_PATH/kvm-switch.sh"   # paste the full script above
chmod +x "$SCRIPT_PATH/kvm-switch.sh"

# (Optional but convenient) symlink into ~ so wrappers can use a stable path
ln -sf "$SCRIPT_PATH/kvm-switch.sh" "$HOME/kvm-switch.sh"
```

---

## 4. Add the `kvms` wrappers

These give you consistent, memorable commands.
Pick the block that matches how you installed the script.

### Wrappers for Option A (symlink into `~`)

```bash
cat >> ~/.bashrc <<'EOF'
# --- kvms-switch Configurations ---
for fn in kvms-status kvms-who kvms-vbox kvms-rd kvms-on kvms-off kvms-fix; do
  unset -f "$fn" 2>/dev/null || true
  unalias "$fn" 2>/dev/null || true
done
kvms-status() { ~/kvm-switch.sh status "$@"; }
kvms-who()    { ~/kvm-switch.sh who "$@"; }
kvms-vbox()   { ~/kvm-switch.sh use-vbox "$@"; }
kvms-rd()     { ~/kvm-switch.sh use-rd "$@"; }
kvms-on()     { ~/kvm-switch.sh kvm-on "$@"; }
kvms-off()    { ~/kvm-switch.sh kvm-off "$@"; }
kvms-fix()    { ~/kvm-switch.sh fix-kvm "$@"; }

EOF

source ~/.bashrc
hash -r
```

### Wrappers for Option B (custom path)

```bash
SCRIPT_PATH="$HOME/Documents/scripts"
cat >> ~/.bashrc <<EOF
# --- kvms-switch Configurations ---
KVMS_PATH="$SCRIPT_PATH/kvm-switch.sh"
for fn in kvms-status kvms-who kvms-vbox kvms-rd kvms-on kvms-off kvms-fix; do
  unset -f "\$fn" 2>/dev/null || true
  unalias "\$fn" 2>/dev/null || true
done
kvms-status() { "\$KVMS_PATH" status "\$@"; }
kvms-who()    { "\$KVMS_PATH" who "\$@"; }
kvms-vbox()   { "\$KVMS_PATH" use-vbox "\$@"; }
kvms-rd()     { "\$KVMS_PATH" use-rd "\$@"; }
kvms-on()     { "\$KVMS_PATH" kvm-on "\$@"; }
kvms-off()    { "\$KVMS_PATH" kvm-off "\$@"; }
kvms-fix()    { "\$KVMS_PATH" fix-kvm "\$@"; }

EOF

source ~/.bashrc
hash -r
```

---

## 5. Usage

| Source | Usage |
| --- | --- |
| kvms-status | Overall health & holders (/dev/kvm) |
| kvms-fix | Re-apply KVM fixes (if some package re-blacklists KVM) |
| kvms-vbox | Switch to VirtualBox (stop RD/Lima, release /dev/kvm, unload kvm modules) |
| kvms-rd | Switch back to Rancher Desktop (stop VBox VMs, load kvm modules, start RD) |
| kvms-who | See exactly who holds /dev/kvm |
| kvms-on | Manual module control to enabling kvm modules |
| kvms-off | Manual module control to disabling kvm modules |



