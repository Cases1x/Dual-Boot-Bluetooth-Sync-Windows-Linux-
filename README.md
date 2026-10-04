# 🔄 Dual Boot Bluetooth Sync (Windows & Linux)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Bash](https://img.shields.io/badge/Language-Bash-4EAA25.svg)](https://www.gnu.org/software/bash/)

A fully automated Bash script that extracts Bluetooth pairing keys from a Windows partition and injects them into a Linux installation. 

If you dual-boot Windows and Linux, you've likely noticed that Bluetooth devices (headphones, mice, keyboards) force you to re-pair them every time you switch operating systems. This script solves that problem by synchronizing the pairing keys so your devices connect seamlessly to both OS environments.

## ✨ Features
* **Universal Support:** Automatically detects and handles both **Classic Bluetooth** (headphones, speakers, controllers) and **Bluetooth Low Energy (BLE)** (modern wireless mice, keyboards).
* **Interactive Menus:** No need to manually copy-paste MAC addresses; the script lists available adapters and devices for you to choose from.
* **Smart Parsing:** Automatically handles 64-bit Hex-to-Decimal Little-Endian conversions required for BLE devices (EDIV, ERand).
* **Safe:** Automatically backs up your Linux Bluetooth configuration before making any changes.

---

## 📋 Prerequisites

Before running the script, ensure you have the required tools installed on your Linux system.

* **`chntpw`**: Required to read the Windows registry hive.
* **`python3`**: Required for accurately converting large 64-bit registry hex values into decimal format.

### Installation
**Arch Linux:**
```bash
sudo pacman -S chntpw python
```
**Debian/Ubuntu:**
```bash
sudo apt install chntpw python3
```
**Fedora:**
```bash
sudo dnf install chntpw python3
```

---

## 🚀 The Syncing Workflow

To ensure the devices are registered correctly in both operating systems, you **must** follow this specific order:

1. **Boot into Linux:** Pair your Bluetooth device normally.
2. **Boot into Windows:** Pair the exact same Bluetooth device. *(This generates a new key on the device, which breaks the Linux pairing).*
3. **Boot back into Linux:** Do **not** unpair the device in Linux. Proceed to the usage instructions below.

---

## 🛠️ Usage

1. **Mount your Windows Drive**  
   Mount your Windows `C:` partition in Linux so the script can read the registry.
   
```bash
   sudo mkdir -p /mnt/windows
   sudo mount /dev/nvme0n1p3 /mnt/windows
```
   *(Be sure to replace `/dev/nvme0n1p3` with your actual Windows partition identifier).*

2. **Download & Run the Script**  
   Clone this repository or download the `sync-bt.sh` script.
   
```bash
   chmod +x sync-bt.sh
   sudo ./sync-bt.sh
```

3. **Follow the Prompts**  
   The script will ask for your Windows mount path, which Bluetooth adapter to use, and which device you want to sync.

---

## 📜 The Script Code

If you prefer not to clone the repository, you can copy the script directly:

<details>
<summary><b>Click to expand and copy <code>sync-bt.sh</code></b></summary>

```bash
#!/bin/bash

# Ensure script is run as root
if [ "$EUID" -ne 0 ]; then
  echo "Error: This script must be run as root."
  exit 1
fi

# Check for required tools
if ! command -v chntpw &> /dev/null; then
    echo "Error: chntpw is not installed. Please install it."
    exit 1
fi
if ! command -v python3 &> /dev/null; then
    echo "Error: python3 is required for 64-bit integer conversion."
    exit 1
fi

# 1. Prompt for Windows Mount Path
read -e -p "Enter the Windows C: drive mount path (e.g., /mnt/windows): " WIN_MOUNT
WINDOWS_SYSTEM_HIVE="$WIN_MOUNT/Windows/System32/config/SYSTEM"

if [ ! -f "$WINDOWS_SYSTEM_HIVE" ]; then
    echo "Error: SYSTEM hive not found at $WINDOWS_SYSTEM_HIVE"
    exit 1
fi

# 2. Select Bluetooth Adapter
echo -e "\nAvailable Bluetooth Adapters (Linux):"
mapfile -t ADAPTERS < <(find /var/lib/bluetooth -mindepth 1 -maxdepth 1 -type d -exec basename {} \;)
if [ ${#ADAPTERS[@]} -eq 0 ]; then 
    echo "No adapters found in /var/lib/bluetooth."
    exit 1
fi

select ADAPTER_MAC in "${ADAPTERS[@]}"; do
    if [ -n "$ADAPTER_MAC" ]; then break; else echo "Invalid selection"; fi
done

# 3. Select Target Bluetooth Device
echo -e "\nAvailable Bluetooth Devices for Adapter $ADAPTER_MAC:"
mapfile -t DEVICES < <(find "/var/lib/bluetooth/$ADAPTER_MAC" -mindepth 1 -maxdepth 1 -type d -exec basename {} \;)
if [ ${#DEVICES[@]} -eq 0 ]; then 
    echo "No paired devices found for this adapter."
    exit 1
fi

select DEVICE_MAC in "${DEVICES[@]}"; do
    if [ -n "$DEVICE_MAC" ]; then break; else echo "Invalid selection"; fi
done

LINUX_INFO_FILE="/var/lib/bluetooth/$ADAPTER_MAC/$DEVICE_MAC/info"
if [ ! -f "$LINUX_INFO_FILE" ]; then
    echo "Error: info file not found at $LINUX_INFO_FILE."
    exit 1
fi

# 4. Find the correct Windows Subkey & Detect Device Type
ADAPTER_MAC_WIN=$(echo "$ADAPTER_MAC" | tr -d ':' | tr '[:upper:]' '[:lower:]')
DEVICE_MAC_WIN=$(echo "$DEVICE_MAC" | tr -d ':' | tr '[:upper:]' '[:lower:]')

echo -e "\nSearching for device $DEVICE_MAC_WIN in Windows Registry..."

# Fetch the directory listing from Windows
CHNTPW_LS=$(chntpw -e "$WINDOWS_SYSTEM_HIVE" <<EOF 2>/dev/null
ls ControlSet001\Services\BTHPORT\Parameters\Keys\\${ADAPTER_MAC_WIN}
q
EOF
)

# Find our specific device in the list
DEVICE_LINE=$(echo "$CHNTPW_LS" | grep -i "$DEVICE_MAC_WIN")

if [ -z "$DEVICE_LINE" ]; then
    echo "Error: Device MAC not found in Windows registry under Adapter $ADAPTER_MAC_WIN."
    echo "DEBUG: Here is the raw output from the registry:"
    echo "$CHNTPW_LS"
    exit 1
fi

echo -e "\nBacking up Linux config to ${LINUX_INFO_FILE}.bak"
cp "$LINUX_INFO_FILE" "${LINUX_INFO_FILE}.bak"

# ------------------------------------------------------------------------------
# BRANCH A: CLASSIC BLUETOOTH
# ------------------------------------------------------------------------------
if echo "$DEVICE_LINE" | grep -qi "REG_BINARY"; then
    echo "-> Detected Classic Bluetooth Device."
    
    CLASSIC_RAW=$(chntpw -e "$WINDOWS_SYSTEM_HIVE" <<EOF 2>/dev/null | grep -E "^:00000"
cd ControlSet001\Services\BTHPORT\Parameters\Keys\\${ADAPTER_MAC_WIN}
hex $DEVICE_MAC_WIN
q
EOF
    )
    
    BT_KEY=$(echo "$CLASSIC_RAW" | cut -d' ' -f2-17 | tr -d ' ' | tr '[:lower:]' '[:upper:]')
    
    if [ -z "$BT_KEY" ]; then
        echo "Error: Failed to extract Classic Key."
        exit 1
    fi
    
    echo "Found LinkKey: $BT_KEY"
    
    # Inject Classic Key
    sed -i '/^\[LinkKey\]/,/^\[/{ s/^Key=.*/Key='"$BT_KEY"'/; }' "$LINUX_INFO_FILE"

# ------------------------------------------------------------------------------
# BRANCH B: BLUETOOTH LOW ENERGY (BLE)
# ------------------------------------------------------------------------------
elif echo "$DEVICE_LINE" | grep -qi "KEY"; then
    echo "-> Detected Bluetooth Low Energy (BLE) Device."
    
    # Get the last column (which is the name) and strip any < > formatting
    DEVICE_SUBKEY=$(echo "$DEVICE_LINE" | awk '{print $NF}' | tr -d '<>')
    REG_PATH="ControlSet001\Services\BTHPORT\Parameters\Keys\\${ADAPTER_MAC_WIN}\\${DEVICE_SUBKEY}"
    
    extract_hex() {
        chntpw -e "$WINDOWS_SYSTEM_HIVE" <<EOF 2>/dev/null | grep -E "^:00000"
cd $REG_PATH
hex $1
q
EOF
    }

    LTK_RAW=$(extract_hex "LTK")
    if [ -z "$LTK_RAW" ]; then
        echo "Error: LTK not found."
        exit 1
    fi

    # Clean up hex outputs
    LTK=$(echo "$LTK_RAW" | cut -d' ' -f2-17 | tr -d ' ' | tr '[:lower:]' '[:upper:]')
    IRK=$(extract_hex "IRK" | cut -d' ' -f2-17 | tr -d ' ' | tr '[:lower:]' '[:upper:]')
    CSRK=$(extract_hex "CSRK" | cut -d' ' -f2-17 | tr -d ' ' | tr '[:lower:]' '[:upper:]')

    # Convert Endianness for EDIV and ERAND
    EDIV_RAW=$(extract_hex "EDIV")
    EDIV_HEX=$(echo "$EDIV_RAW" | awk '{print $5$4$3$2}')
    EDIV_DEC=$(python3 -c "print(int('$EDIV_HEX', 16))")

    ERAND_RAW=$(extract_hex "ERand")
    ERAND_HEX=$(echo "$ERAND_RAW" | awk '{print $9$8$7$6$5$4$3$2}')
    ERAND_DEC=$(python3 -c "print(int('$ERAND_HEX', 16))")

    echo "LTK:   $LTK"
    echo "EDIV:  $EDIV_DEC"
    echo "ERand: $ERAND_DEC"

    # Inject BLE Keys
    sed -i '/^\[LongTermKey\]/,/^\[/{ s/^Key=.*/Key='"$LTK"'/; s/^EDiv=.*/EDiv='"$EDIV_DEC"'/; s/^Rand=.*/Rand='"$ERAND_DEC"'/; }' "$LINUX_INFO_FILE"
    sed -i '/^\[SlaveLongTermKey\]/,/^\[/{ s/^Key=.*/Key='"$LTK"'/; s/^EDiv=.*/EDiv='"$EDIV_DEC"'/; s/^Rand=.*/Rand='"$ERAND_DEC"'/; }' "$LINUX_INFO_FILE"
    
    if [ -n "$IRK" ]; then
        sed -i '/^\[IdentityResolvingKey\]/,/^\[/{ s/^Key=.*/Key='"$IRK"'/; }' "$LINUX_INFO_FILE"
    fi
    if [ -n "$CSRK" ]; then
        sed -i '/^\[LocalSignatureKey\]/,/^\[/{ s/^Key=.*/Key='"$CSRK"'/; }' "$LINUX_INFO_FILE"
    fi

else
    echo "Error: Could not determine if device is Classic or BLE."
    echo "DEBUG INFO: The registry line Windows returned was:"
    echo "$DEVICE_LINE"
    exit 1
fi

echo -e "\nKeys injected successfully. Restarting Bluetooth service..."
systemctl restart bluetooth

echo "Done! Turn your Bluetooth device on; it should now connect automatically."
```
</details>

---

## 🛟 Troubleshooting

* **"Device MAC not found in Windows registry"**: Make sure you have actually paired the device in Windows *after* pairing it in Linux. If your device supports pairing to multiple hosts (like some Logitech mice), ensure it was on the same "profile slot" for both OS pairings.
* **Missing `info` file in Linux**: You must pair the device in Linux first. If the file is missing, the folder structure hasn't been created by `bluetoothctl`.
* **Fast Startup Issues**: If Windows Fast Startup is enabled, the Windows partition might be locked in a hibernated state, making it unreadable/unmountable in Linux. Disable Fast Startup in Windows Power Settings.