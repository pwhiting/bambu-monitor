# Bambu P1S Print Monitor

A simple Unix-style pipeline for monitoring your Bambu Lab 3D printer's progress by downloading timelapse videos, extracting the latest frame, and uploading to a web server for remote viewing.

## Overview

This tool consists of three chainable scripts:

1. **`latest_avi`** - Downloads the latest timelapse video from your Bambu printer
2. **`last_image`** - Extracts the last frame from a video file
3. **`scp_image`** - Uploads the image to a remote server via SCP

## Requirements

### Software Dependencies

- Python 3.6+
- `curl` - for FTP downloads
- `ffmpeg` and `ffprobe` - for video frame extraction and conversion
- `scp` - for uploading to remote server (optional)

**Install on Ubuntu/Debian:**
```bash
sudo apt install curl ffmpeg openssh-client python3
```

**Install on macOS:**
```bash
brew install curl ffmpeg python3
```

### Bambu Printer Setup

1. On your Bambu P1S printer, go to **Settings → WLAN**
2. Note your **Access Code** (8-digit number)
3. Note your printer's hostname or IP address (e.g., `p1s.whitings.org` or `10.0.0.13`)

## Installation

1. Clone or download these files:
   - `latest_avi`
   - `last_image`
   - `scp_image`

2. Make scripts executable:
```bash
chmod +x latest_avi last_image scp_image
```

## First Run - Step by Step

Let's walk through each step individually to make sure everything works:

### Step 1: Download a Video

First, configure and download the latest timelapse:
```bash
./latest_avi --once
```

You'll be prompted for:
- **Printer hostname/IP**: Your printer's address (e.g., `p1s.whitings.org`)
- **Access code**: 8-digit code from printer's WiFi settings  
- **Update interval**: How often to check for new videos in seconds (e.g., `300` for 5 minutes)

This downloads the video to `./bambu_videos/video_YYYY-MM-DD_HH-MM-SS.avi` and prints the path.

### Step 2: Watch the Video

Convert the AVI to MP4 and watch it to verify it's what you expect:
```bash
# Find the latest video
VIDEO=$(ls -t bambu_videos/*.avi | head -1)

# Convert to MP4
ffmpeg -i "$VIDEO" -c:v libx264 -preset fast -crf 22 test_video.mp4

# Watch it (macOS)
open test_video.mp4

# Or (Linux)
xdg-open test_video.mp4
```

### Step 3: Extract the Last Frame

Now extract the last frame from that video:
```bash
./last_image "$VIDEO" test_snapshot.jpg
```

This creates `test_snapshot.jpg`. View it to make sure it captured the end of the print:
```bash
# macOS
open test_snapshot.jpg

# Linux  
xdg-open test_snapshot.jpg
```

### Step 4: Upload the Image

Finally, test uploading to your web server:
```bash
./scp_image test_snapshot.jpg user@yourserver.com:path/to/webdir/
```

Replace `user@yourserver.com:path/to/webdir/` with your actual server details.

If all four steps work, you're ready to run the full pipeline!

## Running the Full Pipeline

### Run Once (Manual)

Download, extract, and upload the latest print snapshot:
```bash
./latest_avi --once | ./last_image snapshot.jpg | ./scp_image user@yourserver.com:bambu/
```

Or if you've set up defaults in `scp_image`, even simpler:
```bash
./latest_avi --once | ./last_image | ./scp_image
```

### Continuous Monitoring (Loop Mode)

Run continuously, checking for new prints every 5 minutes (or your configured interval):
```bash
./latest_avi | ./last_image snapshot.jpg | ./scp_image user@yourserver.com:bambu/
```

This will:
1. Check for new timelapse videos every N seconds
2. Download only if the video has changed
3. Extract the last frame
4. Upload to your web server
5. Repeat indefinitely

Stop with `Ctrl+C`.

## Configuration

### Printer Configuration

The `.bambu_config` file is created on first run:
```bash
# Bambu Lab Printer Configuration
PRINTER_HOST=p1s.whitings.org
ACCESS_CODE=12345678

# Update interval in seconds (e.g., 300 = 5 minutes)
UPDATE_INTERVAL=300

# FTP Settings (defaults usually work)
FTP_PORT=990
FTP_USER=bblp
TIMELAPSE_PATH=/timelapse/
```

You can edit this file directly or delete it to be prompted again.

### Upload Destination

You can configure the default upload destination by editing the `scp_image` script, or pass it as an argument each time.

## Script Usage

### latest_avi

Downloads the latest timelapse video from your Bambu printer via FTPS.
```bash
./latest_avi [OPTIONS]
```

**Options**:
- `--once` - Run once and exit (no loop)
- `-i SECONDS` - Update interval in seconds (overrides config)
- `--debug` - Show connection and debug information

**Output**: 
- Prints the path to downloaded video to stdout (only when video changes)
- Status messages go to stderr

**Examples**:
```bash
# Download once
./latest_avi --once

# Loop with custom interval
./latest_avi -i 60

# Debug mode
./latest_avi --debug --once
```

### last_image

Extracts the last frame from a video file.
```bash
./last_image [VIDEO_FILE] [OUTPUT_FILE]
```

**Arguments**:
- `VIDEO_FILE` - (Optional) Path to video file. If not provided, reads paths from stdin
- `OUTPUT_FILE` - (Optional) Output filename. Defaults to `snapshot.jpg`

**Output**:
- Prints the path to extracted image to stdout
- Status messages go to stderr

**Examples**:
```bash
# From stdin, output to default (snapshot.jpg)
echo "bambu_videos/video_2025-12-18_12-33-03.avi" | ./last_image

# One argument - video file, output to default (snapshot.jpg)
./last_image bambu_videos/video_2025-12-18_12-33-03.avi

# Two arguments - video file and custom output
./last_image bambu_videos/video_2025-12-18_12-33-03.avi /tmp/print.jpg

# In a pipeline with custom output
./latest_avi --once | ./last_image print_monitor/current.jpg
```

### scp_image

Uploads files to a remote server via SCP.
```bash
./scp_image [FILE] [DESTINATION]
```

**Arguments**:
- `FILE` - (Optional) File to upload. If not provided, reads paths from stdin
- `DESTINATION` - (Optional) Remote destination (user@host:path). If not provided, prompts or reads from stdin

**Usage modes**:
```bash
# Two arguments - upload single file
./scp_image snapshot.jpg user@server.com:bambu/

# One argument - destination provided, file from stdin
echo "snapshot.jpg" | ./scp_image user@server.com:bambu/

# No arguments - prompts for destination, then reads files from stdin
./scp_image
# Prompts: Destination (e.g., user@host:path):

# Pipeline mode
./last_image video.avi | ./scp_image user@server.com:bambu/
```

**Output**:
- Status messages to stderr
- Exit code 0 on success, 1 on failure

## Complete Pipeline Examples

### Basic Pipeline
```bash
./latest_avi --once | ./last_image snapshot.jpg | ./scp_image user@server.com:www/bambu/
```

### Continuous Monitoring
```bash
./latest_avi -i 300 | ./last_image print_monitor/current.jpg | ./scp_image user@server.com:www/bambu/
```

### Local Only (No Upload)
```bash
./latest_avi | ./last_image ~/Desktop/print.jpg
```

### Default Output (snapshot.jpg)
```bash
./latest_avi --once | ./last_image | ./scp_image user@server.com:bambu/
```

### Custom Processing
```bash
./latest_avi --once | \
  ./last_image original.jpg | \
  tee >(./scp_image user@server.com:bambu/original/) | \
  convert - -resize 800x600 thumbnail.jpg
```

## Running as a Background Service

### Using systemd (Linux)

Create `/etc/systemd/system/bambu-monitor.service`:
```ini
[Unit]
Description=Bambu P1S Print Monitor
After=network.target

[Service]
Type=simple
User=youruser
WorkingDirectory=/home/youruser/bambu-monitor
ExecStart=/bin/bash -c './latest_avi | ./last_image snapshot.jpg | ./scp_image user@server.com:bambu/'
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable bambu-monitor
sudo systemctl start bambu-monitor
sudo systemctl status bambu-monitor
```

### Using cron (Linux/macOS)

Run every 5 minutes:
```bash
crontab -e
```

Add:
```cron
*/5 * * * * cd /home/youruser/bambu-monitor && ./latest_avi --once | ./last_image snapshot.jpg | ./scp_image user@server.com:bambu/ 2>&1 | logger -t bambu-monitor
```

### Using screen/tmux (Quick & Dirty)
```bash
screen -S bambu
cd ~/bambu-monitor
./latest_avi | ./last_image snapshot.jpg | ./scp_image user@server.com:bambu/
# Press Ctrl+A, then D to detach
```

Reattach later:
```bash
screen -r bambu
```

## Troubleshooting

### "Failed to list directory"

- Verify printer is on and connected to network: `ping p1s.whitings.org`
- Check access code is correct (8 digits from printer's WiFi settings)
- Try with `--debug` flag: `./latest_avi --debug --once`

### "No timelapse videos found"

- Print something first! The printer only creates timelapse videos after a print completes
- Check if timelapses are enabled in Bambu Studio/Handy settings

### "Extraction failed" or "ffmpeg error"

- Verify ffmpeg is installed: `ffmpeg -version`
- Try manually extracting: `ffmpeg -sseof -1 -i ./bambu_videos/video_*.avi -update 1 test.jpg`
- Check video file isn't corrupted: `ffmpeg -i ./bambu_videos/video_*.avi -f null -`

### "SCP failed" or "Upload failed"

- Test SSH access: `ssh user@yourserver.com`
- Set up SSH key authentication to avoid password prompts:
```bash
  ssh-keygen -t ed25519
  ssh-copy-id user@yourserver.com
```
- Check permissions on remote directory

### "Video unchanged" but I know there's a new print

- The script tracks both filename AND filesize to detect changes
- If you manually deleted the video on the printer, reset the state:
```bash
  rm bambu_videos/.last_avi_state
```

### Scripts not executable
```bash
chmod +x latest_avi last_image scp_image
```

## File Locations

- **Configuration**: `.bambu_config` (created on first run)
- **Downloaded videos**: `./bambu_videos/video_*.avi`
- **State tracking**: `./bambu_videos/.last_avi_state`
- **Extracted image**: `snapshot.jpg` (default, configurable as second argument)

## Advanced Usage

### Multiple Printers

Create separate directories for each printer:
```bash
mkdir printer1 printer2
cd printer1
ln -s ../latest_avi
ln -s ../last_image
ln -s ../scp_image
./latest_avi --once  # Configure for printer1
```

### Process Old Videos

Extract frames from all videos in the archive:
```bash
for video in bambu_videos/*.avi; do
  ./last_image "$video" "frames/$(basename "$video" .avi).jpg"
done
```

### Create Timelapse of Timelapses
```bash
# Extract last frame from each video
for video in bambu_videos/*.avi; do
  ./last_image "$video" "frames/$(basename "$video" .avi).jpg"
done

# Combine into video
ffmpeg -framerate 1 -pattern_type glob -i 'frames/*.jpg' \
  -c:v libx264 -pix_fmt yuv420p all_prints.mp4
```

### Watch for New Prints (No Loop)

If you prefer using a system scheduler instead of the built-in loop:
```bash
# In cron or systemd timer
./latest_avi --once | ./last_image snapshot.jpg | ./scp_image user@server.com:bambu/
```

## How It Works

The pipeline follows Unix philosophy - each script does one thing well:

1. **latest_avi**: 
   - Connects to printer via FTPS
   - Lists timelapse directory
   - Compares latest video's name and size to cached state
   - Downloads only if changed
   - Outputs video path to stdout (only on change)

2. **last_image**:
   - Reads video path from stdin or argument
   - Uses ffmpeg to extract last frame
   - Outputs image path to stdout

3. **scp_image**:
   - Reads file path from stdin or argument
   - Uploads via SCP to remote server
   - Reports success/failure to stderr

All status messages go to stderr, so stdout stays clean for piping.

## License

MIT License - feel free to modify and distribute!

## Contributing

Improvements welcome! These scripts follow Unix philosophy:
- Do one thing well
- Accept input on stdin or arguments
- Output results to stdout
- Log status to stderr
- Chain with pipes