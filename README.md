# 🎬 Frozen-Pi Media Vault: The Ultimate Offline Static Streaming Appliance

An architectural blueprint for turning a Raspberry Pi and an external 1TB USB hard drive into an immortal, 100% offline, read-only media node. By freezing the operating systems and filesystems into a static state, this appliance is entirely invulnerable to sudden power cuts (plug yanking) and provides a fast, poster-rich, Netflix-style interface on a Fire TV stick without any outside internet dependencies.

---

## 📌 Features & Architecture Overview
* **100% Offline Autonomy:** No cloud authentication barriers (unlike Stremio). Native metadata parsing runs entirely inside the local network.
* **Immortal Hardware Life:** Both the Pi SD card (via `OverlayFS`) and the 1TB hard drive are locked in a **Read-Only (RO)** state. Zero write wear, zero fragmentation, and absolute immunity from database or filesystem corruption during sudden power drops.
* **Ultra-Lightweight Client UI:** Powered by Nova Video Player on Fire TV. App size stays under ~150MB because artwork is streamed dynamically from the Pi rather than filling up the TV stick's internal storage.
* **Separated Media Libraries:** Clean network classification splits movies and television series into dedicated, distinct native dashboards.


```

```
                  [ TOTAL INTERNET OUTAGE / POWER DROP ]
                                     │

```

┌───────────────────────────────────────┼───────────────────────────────────────┐
│ YOUR LOCAL OFFLINE LAN                ▼                                       │
│                                                                               │
│  ┌──────────────────────┐   SMB Protocol (Port 445)   ┌────────────────────┐  │
│  │   Raspberry Pi 3     │ ──────────────────────────> │   Fire TV 4K       │  │
│  │  • OS: OverlayFS (RO)│                             │   • Nova Player    │  │
│  │  • 1TB Drive (RO)    │ <──[Reads Pre-Baked Assets]─ │   • Static Grid    │  │
│  └──────────────────────┘                             └────────────────────┘  │
│              │                                                                │
│              ▼ (Direct Local Disk Directory Mapping)                          │
│       /Movies/Inception (2010)/                                               │
│         ├── Inception (2010).mkv  (Raw Video Container)                       │
│         ├── Inception (2010).nfo  (Local XML Movie Details/Actors)            │
│         └── Inception (2010)-poster.jpg  (Local High-Res Cover Art)           │
└───────────────────────────────────────────────────────────────────────────────┘

```

---

## 🛠️ Step 1: Raspberry Pi Provisioning & Setup

### 1. Minimal Base OS Installation
* **OS Version:** Raspberry Pi OS Lite (64-bit), Debian Bookworm base.
* Flash using Raspberry Pi Imager, enable SSH, log into the terminal, and execute system updates:
  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo apt install samba samba-common-bin python3 python3-pip python3-requests ntfs-3g sg3-utils -y

```

### 2. Static Permanent Hard Drive Mount

To map your 1TB USB drive consistently across power cycles:

1. Identify the partition's unique hardware token (`UUID`):
```bash
lsblk -o NAME,FSTYPE,UUID,MOUNTPOINTS

```


2. Create the target attachment node:
```bash
sudo mkdir -p /mnt/movies

```


3. Append the static mounting rules into the system table:
```bash
sudo nano /etc/fstab

```


Add the following line at the bottom (configured dynamically as **Read-Write `rw**` initially for database staging):
```text
UUID=YOUR-TARGET-UUID-HERE /mnt/movies auto rw,user,nofail,noatime 0 0

```


4. Bind the changes immediately:
```bash
sudo mount -a

```



### 3. Anonymous Samba Service Setup

Configure Samba to broadcast two distinct, clean shares for your media library without forcing password entry screens on the TV client.

1. Open the network management configuration:
```bash
sudo nano /etc/samba/smb.conf

```


2. Ensure this line is present inside the `[global]` block to allow guest access:
```text
map to guest = bad user

```


3. Scroll to the absolute bottom of the file and paste the layout shares (adjust path strings if your actual hard drive directories contain internal spaces, e.g., `/mnt/movies/TV Shows` — *do not wrap paths in quotes or use backslashes here*):
```text
[Offline_Movies]
  path = /mnt/movies/Movies
  browseable = yes
  read only = yes
  guest ok = yes
  public = yes
  printable = no

[Offline_Series]
  path = /mnt/movies/Series
  browseable = yes
  read only = yes
  guest ok = yes
  public = yes
  printable = no

```


4. Fire up and enable the file-sharing system:
```bash
sudo systemctl restart smbd
sudo systemctl enable smbd

```



---

## 🐍 Step 2: Local Headless Python Scraper Pipeline

Because the appliance runs entirely offline, we use a custom Python terminal script while the drive is in Read-Write mode to generate the complete visual asset layer directly onto the hard drive structure.

### 1. Setup Your TMDb API Key

1. Register a free account at [themoviedb.org](https://www.themoviedb.org/).
2. Navigate to **Settings > API** and generate a free API Key.

### 2. The Deployment Script (`~/tmm_scraper.py`)

Create the scraping automation file:

```bash
nano ~/tmm_scraper.py

```

Paste the following functional production scraper:

```python
import os
import re
import urllib.parse
import xml.etree.ElementTree as ET
import requests

# --- CONFIGURATION LAYER ---
TMDB_API_KEY = "YOUR_TMDB_API_KEY_HERE"
TARGET_DIR = "/mnt/movies"
BASE_URL = "[https://api.themoviedb.org/3](https://api.themoviedb.org/3)"
IMG_URL = "[https://image.tmdb.org/t/p/w500](https://image.tmdb.org/t/p/w500)"

def clean_filename(name):
    # Strip extensions and replace symbols with spaces
    base = os.path.splitext(name)[0]
    cleaned = re.sub(r'[\._-]', ' ', base)
    # Attempt to extract a 4-digit year loop
    year_match = re.search(r'\b(19|20)\d{2}\b', cleaned)
    year = year_match.group(0) if year_match else None
    if year:
        cleaned = cleaned.replace(year, "").strip()
    return re.sub(r'\s+', ' ', cleaned).strip(), year

def download_asset(url, dest_path):
    try:
        response = requests.get(url, timeout=15)
        if response.status_code == 200:
            with open(dest_path, 'wb') as f:
                f.write(response.content)
    except Exception:
        pass

def scrape_library():
    headers = {"Authorization": f"Bearer {TMDB_API_KEY}"} if len(TMDB_API_KEY) > 40 else {}
    params_key = {"api_key": TMDB_API_KEY} if len(TMDB_API_KEY) <= 40 else {}

    for root, _, files in os.walk(TARGET_DIR):
        for file in files:
            if not file.lower().endswith(('.mkv', '.mp4', '.avi')):
                continue
                
            video_path = os.path.join(root, file)
            base_name = os.path.splitext(file)[0]
            nfo_path = os.path.join(root, f"{base_name}.nfo")
            poster_path = os.path.join(root, f"{base_name}-poster.jpg")
            fanart_path = os.path.join(root, f"{base_name}-fanart.jpg")
            
            # Skip if metadata is already pre-baked
            if os.path.exists(nfo_path):
                continue
                
            title, year = clean_filename(file)
            print(f"Scraping metadata for: {title} ({year or 'Unknown'})")
            
            # Query Search API
            query_params = {**params_key, "query": title}
            if year:
                query_params["year"] = year
            
            try:
                search_res = requests.get(f"{BASE_URL}/search/movie", params=query_params, headers=headers, timeout=10).json()
                if not search_res.get('results'):
                    continue
                
                movie_data = search_res['results'][0]
                movie_id = movie_data['id']
                
                # Fetch Extended Credits Details (Actors)
                detail_params = {**params_key, "append_to_response": "credits"}
                full_details = requests.get(f"{BASE_URL}/movie/{movie_id}", params=detail_params, headers=headers, timeout=10).json()
                
                # Generate NFO Structural Tree
                movie_xml = ET.Element("movie")
                ET.SubElement(movie_xml, "title").text = full_details.get('title', title)
                ET.SubElement(movie_xml, "year").text = full_details.get('release_date', '0000-00-00')[:4]
                ET.SubElement(movie_xml, "rating").text = str(full_details.get('vote_average', 0.0))
                ET.SubElement(movie_xml, "plot").text = full_details.get('overview', '')
                ET.SubElement(movie_xml, "tagline").text = full_details.get('tagline', '')
                
                for genre in full_details.get('genres', []):
                    ET.SubElement(movie_xml, "genre").text = genre['name']
                
                # Extract Top 10 Billing Actors
                cast = full_details.get('credits', {}).get('cast', [])[:10]
                for member in cast:
                    actor_node = ET.SubElement(movie_xml, "actor")
                    ET.SubElement(actor_node, "name").text = member.get('name', '')
                    ET.SubElement(actor_node, "role").text = member.get('character', '')
                    if member.get('profile_path'):
                        ET.SubElement(actor_node, "thumb").text = f"{IMG_URL}{member['profile_path']}"
                
                # Write XML File to storage block
                tree = ET.ElementTree(movie_xml)
                with open(nfo_path, 'wb') as f:
                    tree.write(f, encoding='utf-8', xml_declaration=True)
                
                # Fetch Graphic Identity Assets
                if full_details.get('poster_path'):
                    download_asset(f"{IMG_URL}{full_details['poster_path']}", poster_path)
                if full_details.get('backdrop_path'):
                    download_asset(f"{IMG_URL}{full_details['backdrop_path']}", fanart_path)
                    
            except Exception as e:
                print(f"Skipping error entry: {e}")

if __name__ == "__main__":
    scrape_library()

```

Run the routine to index your files cleanly:

```bash
python3 ~/tmm_scraper.py

```

---

## 🔒 Step 3: Filesystem Locking & Read-Only Automation

Once your metadata assets are stored cleanly on the drive, we lock down the hardware environment completely.

### 1. Toggle Hard Drive to True Read-Only

Open your storage mount table:

```bash
sudo nano /etc/fstab

```

Modify the hard drive parameters, switching `rw` to strict read-only tracking configurations: **`ro,noatime`**.

```text
UUID=YOUR-TARGET-UUID-HERE /mnt/movies auto ro,user,nofail,noatime 0 0

```

Re-mount the engine blocks:

```bash
sudo umount /mnt/movies && sudo mount -a

```

### 2. Freeze the Raspberry Pi OS via OverlayFS

We freeze the operating system using native tools so that no disk writes can occur on the micro SD card.

1. Run the core system utility configuration tool:
```bash
sudo raspi-config

```


2. Navigate using your arrow keys: **Performance Options > Overlay Filesystem**.
3. Select **Yes** to enable the underlying overlay structure.
4. Select **Yes** to make the boot partition strictly write-protected.
5. Finish out and click **Yes** to let the system execute a clean reboot.

---

## 🔄 Step 4: System Management (Maintenance Mode Toggling)

Whenever you want to add new movies to the drive or tweak the Pi's server parameters, use this maintenance workflow script to unlock and lock the system cleanly via command arguments.

1. Create the system maintenance tool:
```bash
nano ~/toggle_mode.sh

```


2. Paste the unified shell controller logic:

```bash
#!/bin/bash

# Ensure script runs with root level permissions
if [ "$EUID" -ne 0 ]; then
  echo "Error: Please run this control utility using sudo."
  exit 1
fi

case "$1" in
  writable)
    echo "🔓 Unlocking System Framework into Read-Write Mode..."
    # Disable OS Overlay protection flags for next boot sequence
    raspi-config nonint do_overlayfs 1
    # Remount physical drive block directly into an open read-write layer
    mount -o remount,rw /mnt/movies
    echo "System is primed for content updates. Please execute a 'sudo reboot' now."
    ;;
  
  readonly)
    echo "🔒 Freezing System Framework into Bulletproof Static State..."
    # Lock OS Overlay protection flags back on for subsequent boots
    raspi-config nonint do_overlayfs 0
    # Force hard disk partition to drop active write streams and mount read-only
    mount -o remount,ro /mnt/movies
    echo "System is locked. Plugs can be pulled safely. Please execute a 'sudo reboot' now."
    ;;
  
  *)
    echo "Usage: sudo $0 {writable|readonly}"
    exit 1
    ;;
esac
```

3. Set execution rights:
```bash
chmod +x ~/toggle_mode.sh

```



* **To Add Movies:** Run `sudo ~/toggle_mode.sh writable && sudo reboot`. (The system boots open; run your python scraper or copy files over, then lock it back up).
* **To Finalize Vault:** Run `sudo ~/toggle_mode.sh readonly && sudo reboot`. (The system freezes permanently).

---

## 📺 Step 5: Nova Video Player Interface Setup (Fire TV)

1. **Install App:** Open the Amazon Appstore on your Fire TV, search for **Nova Video Player**, and install it (App icon features a prominent blue chevron bracket launcher symbol).
2. **Mount the Movies Share:**
* Go to the left sidebar menu and navigate to **Network**.
* Select **SMBJ** (optimized Java-based modern SMB protocol stack).
* Select **Add Network Shortcut / Add Server (+)**.
* **Server Address / IP:** Enter your Pi's local address (e.g., `192.168.1.50`).
* **Port:** Leave blank or default to `445`.
* **Path:** Enter `/Offline_Movies` *(Note the exact case matching your Samba config).*
* **Authentication:** Select **Guest** (leave username/password empty). Click OK.


3. **Mount the Series Share:** Repeat the exact step above, but change the **Path** entry parameter field to `/Offline_Series`.
4. **Initialize Library Sync:**
* Click into the newly mapped `/Offline_Movies` network folder view.
* Press **Up** on your remote control to highlight the header dashboard menu banner.
* Select **"Index this folder"** and toggle the media parsing target type classification explicitly to **Movies**.
* Repeat this inside the `/Offline_Series` folder, setting the target classification parse mapping to **TV Shows / Series**.


5. **Enforce Static Local Priority (Crucial):**
* Go to Nova's main **Settings** dashboard layout menu (gear icon on the left panel layout view).
* Navigate to **Scraper / Library Data Management**.
* Toggle **OFF** *"Automatically search information online"* or *"Fetch metadata missing states via web channels"*.



Nova will instantly parse the static layout `.nfo` files and `-poster.jpg` graphics directly off the Pi's read-only file shares. Your home screen will immediately populate with a fast, gorgeous, fully searchable visual interface that works instantly—even if your internet service provider goes completely down.


## Why Drive Spindown Still Matters in Read-Only Mode

When you set the filesystem to Read-Only, you eliminate **software wear** (no data fragmentation, no accidental corruptions, no file system writes). However, a mechanical hard drive still suffers from **physical/mechanical wear**:

1. **Bearing and Motor Lifespan:** As long as the drive is spinning at 5400 or 7200 RPM, the internal spindle motor and fluid dynamic bearings are experiencing continuous kinetic friction and producing heat. Stopping the rotation cuts this wear entirely.
2. **Power Consumption & Heat:** Even if no files are being read, a spinning drive consumes roughly 4–5 watts of power continuously, dissipating it as heat inside the drive enclosure. A spun-down drive drops to an ultra-low power state (~0.5 watts) and stays completely cool.
3. **The Power-Cut Safety Net:** If the drive motor is already at a complete mechanical stop (0 RPM) when you suddenly flip the $12\text{V}$ main power switch, there is absolutely zero physical stress on the drive components. The magnetic head is already safely parked, and the platters are stationary.

---

## 🛠️ Step-by-Step Spindown Implementation

Since your entire library is read-only, we can configure a rock-solid spindown utility called `hdparm` or use a lightweight Python daemon that monitors local network/system activity. However, for an offline appliance setup, the most reliable tool that works perfectly across NTFS, ext4, and different USB-to-SATA bridge controllers is a lightweight background daemon called **`hd-idle`**.

Log into your Pi via SSH (ensure it is in **Read-Write** mode using your `toggle_mode.sh writable` script before proceeding).

### 1. Install and Compile `hd-idle`

The standard repository versions can sometimes be finicky with modern USB drives, so compiling the tiny, zero-dependency `hd-idle` utility ensures it handles your external 1TB drive perfectly.

```bash
sudo apt install build-essential git -y
cd ~
git clone https://github.com/adelolmo/hd-idle.git
cd hd-idle
make
sudo make install

```

### 2. Identify Your Drive Name

Find the raw disk identifier name of your 1TB drive (e.g., `sda`, `sdb`):

```bash
lsblk

```

*(Look for your 1TB disk. Note down the disk name, usually `sda`—do not use the partition name like `sda1`).*

### 3. Configure the Spindown Time

1. Open the configuration file for the service:
```bash
sudo nano /etc/default/hd-idle

```


2. Paste or modify the settings to look like this:
```text
# Enable the daemon
START_HD_IDLE=true

# Spindown configuration: -i 0 disables default, -a target specifies drive, -i 600 sets idle time to 10 minutes (600 seconds)
HD_IDLE_OPTS="-i 0 -a sda -i 600"

```


*(Replace `sda` if your drive identifier was different. `600` seconds means if Nova hasn't streamed a video or crawled the folder for 10 minutes, the drive will mechanically sleep).*
3. Save and close the editor (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 4. Enable the System Daemon

Create the automation script to let it start natively on boot:

```bash
sudo cp debian/hd-idle.init /etc/init.d/hd-idle
sudo chmod +x /etc/init.d/hd-idle
sudo update-rc.d hd-idle defaults
sudo systemctl start hd-idle

```

---

## 📝 Update Your `README.md` File

To make sure your master documentation is fully complete, copy and paste this new block into your existing `README.md` file right before the **Step 3: Filesystem Locking** section:

```markdown
### 4. Dynamic Hard Drive Mechanical Spindown Setup
To eliminate mechanical friction, reduce ambient heat, and protect the drive components before a hard power drop, compile `hd-idle` to spin down the drive platters after 10 minutes of complete network/system inactivity.

1. Fetch and compile the lightweight zero-dependency utility:
   ```bash
   sudo apt install build-essential git -y
   cd ~ && git clone [https://github.com/adelolmo/hd-idle.git](https://github.com/adelolmo/hd-idle.git)
   cd hd-idle && make && sudo make install

```

2. Edit the environmental default parameters:
```bash
sudo nano /etc/default/hd-idle

```


Populate the runtime parameters exactly as follows (targeting the raw disk `sda` on a 600-second idle threshold window):
```text
START_HD_IDLE=true
HD_IDLE_OPTS="-i 0 -a sda -i 600"

```


3. Establish execution rights and link the initialization daemon to system boot pipelines:
```bash
sudo cp debian/hd-idle.init /etc/init.d/hd-idle
sudo chmod +x /etc/init.d/hd-idle
sudo update-rc.d hd-idle defaults
sudo systemctl start hd-idle

```



```

---

## 🔒 The Final Lock Routine

Now that every single layer of your appliance is built, tested, and automated, it's time to trigger the final lockdown phase:

1. Execute your master management utility to lock the system:
   ```bash
   sudo ~/toggle_mode.sh readonly

```

2. Trigger the final system reboot cycle:
```bash
sudo reboot

```



Your Frozen-Pi Media Vault is officially complete! The system is now fully static, write-protected, capable of safely spinning down its mechanical parts when you stop watching, and completely ready for you to pull the power cable whenever you like. Excellent engineering!
---
Direct installation via `apt` is much cleaner, and no, you **do not need the hardware serial number** to target it!

Looking at your `lsblk` output, your physical 1TB USB drive maps directly to the raw storage node **`sda`** (with the partition `sda1` mounted at `/mnt/drive`). We can target `sda` directly in the configuration file.

Here are the exact terminal steps to install, configure, and lock it down:

---

### Step 1: Install the Package Natively

Since you are in Read-Write mode, bypass compiling entirely and grab the official build:

```bash
sudo apt update
sudo apt install hd-idle -y

```

---

### Step 2: Configure the Spindown Target

1. Open the configuration file generated by the installation:
```bash
sudo nano /etc/default/hd-idle

```


2. Look for the `START_HD_IDLE` and `HD_IDLE_OPTS` lines. Modify them to match your raw disk name (`sda`) and set the timeout to 10 minutes (`600` seconds):
```text
# Enable the daemon on system boot
START_HD_IDLE=true

# Global default disabled (-i 0), target disk explicitly set (-a sda -i 600)
HD_IDLE_OPTS="-i 0 -a sda -i 600"

```


3. Save and close the editor (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### Step 3: Start and Enable the Background Monitor

Force the system framework to load the daemon and start tracking drive idle timeouts immediately:

```bash
sudo systemctl enable hd-idle
sudo systemctl restart hd-idle

```

---

### Step 4: Verify It Is Running

Make sure the service is running happily in the background without errors:

```bash
sudo systemctl status hd-idle

```

*(You should see a green `active (running)` status indicator).*

---

### Step 5: Execute the Final System Lockdown

Now that your network configurations, Samba paths, metadata structure, and mechanical safety switches are fully deployed, run your management tool to freeze the entire appliance framework into its permanent, read-only state:

```bash
sudo ~/toggle_mode.sh readonly
sudo reboot

```

Once the Pi finishes rebooting, you are completely done. The system is solid, immune to power cuts, and will quietly spin down your drive platters whenever you step away from the TV!

