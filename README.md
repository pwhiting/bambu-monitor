# Bambu P1S Print Monitor

A Unix-style pipeline system for monitoring your Bambu Lab 3D printer. Downloads timelapse videos, extracts frames, checks for print failures using AI, and sends phone notifications when things go wrong.

## Quick Start - Getting It Running

The system is a set of independent modules that you can run on the command line separately. Each script logs heavily to stderr (so you can see what's happening) but only outputs to stdout when it needs to pass data to the next script in the pipeline.

### Initial Setup

1. **Clone the repo and set up your environment:**

```bash
cd ~/bambu-monitor
chmod +x last_avi last_image scp_image check notify loop

# Create your start script from the sample
cp start.sample start
```

2. **Edit your `start` script with your credentials:**

```bash
export OPENAI_API_KEY=sk-proj-your-key-here
export PUSHCUT_WEBHOOK_SECRET=your-webhook-here
export PYTHONUNBUFFERED=1
cd ~/bambu-monitor
PATH=$PATH:~/bambu-monitor

last_avi | loop last_image | loop scp_image {} user@server.com:bambu/snapshot.jpg | loop check | loop notify Bambu
```

You'll need:
- **OpenAI API key** - for the AI vision check (only uses cheap models like GPT-4o-mini)
- **Pushcut webhook secret** - from the Pushcut app (free tier is fine)

For Pushcut: Download the app, log in, create a notification named "Bambu" (or whatever you want), and grab your webhook secret from the settings.

### Step-by-Step Testing

**Step 1: Download a video**

Simply run `last_avi`. It will ask you some questions the first time:
- Printer IP address (find it on the printer's network settings)
- 8-character access code (from the printer's WiFi settings on the front panel)
- Update interval in seconds (300 = 5 minutes)

```bash
./last_avi --once
```

If all goes well, it will download the latest timelapse to `./bambu_videos/`. It keeps track of state, so the next time you run it, it probably won't download another unless the video has changed.

You can verify the video works:
```bash
VIDEO=$(ls -t bambu_videos/*.avi | head -1)
ffmpeg -i "$VIDEO" output.mp4 && open output.mp4
```

**Step 2: Extract the last frame**

Once you have a video, test the image extractor:

```bash
last_image bambu_videos/video_2025-12-19_18-58-33.avi
```

You should see:
```
last_image: Input: bambu_videos/video_2025-12-19_18-58-33.avi
last_image: Output: /home/pete/bambu-monitor/bambu_images/latest.jpg
last_image: Processing video: bambu_videos/video_2025-12-19_18-58-33.avi
last_image: Extracting last frame → /home/pete/bambu-monitor/bambu_images/latest.jpg
last_image: ✓ Extracted last frame (74,436 bytes)
last_image: ✓ Saved to: /home/pete/bambu-monitor/bambu_images/latest.jpg
/home/pete/bambu-monitor/bambu_images/latest.jpg
```

Go look at `bambu_images/latest.jpg` and it should be the last frame of the video.

**Step 3: Test the AI check (optional)**

If you have test images, you can verify the AI failure detection:

```bash
check badprint.jpg
check goodprint.jpg
```

Use the `-m` flag to try different models:
```bash
check -m gpt-4o-mini badprint.jpg
```

The script only outputs the filename to stdout if it thinks the print is failing.

**Step 4: Test upload (optional)**

If you want to upload to a web server:

```bash
scp_image bambu_images/latest.jpg user@server.com:bambu/snapshot.jpg
```

**Step 5: Test notification (optional)**

```bash
notify Bambu
```

This should send a notification to your phone/watch via Pushcut.

### Understanding the Pipeline

The way these scripts tie together is through Unix pipes. Each script:
- Prints debug/status info to **stderr** (so you can see what's happening)
- Prints filenames to **stdout** only when it needs to trigger the next step

Since all these scripts needed the same looping logic, I created the `loop` command. It listens on stdin and whenever it gets a line, it runs the command with that line as an argument.

**Example:**
```bash
last_avi | loop last_image
```

When `last_avi` downloads a new video, it prints the filename to stdout. `loop` reads it and runs:
```bash
last_image bambu_videos/video_2025-12-19_18-58-33.avi
```

Then `last_image` extracts the frame and prints the image path to stdout, which continues down the chain.

**The `{}` Placeholder:**

Sometimes you need the stdin value in a specific position (not at the end). Use `{}` like `find -exec`:

```bash
echo "source.jpg" | loop scp_image {} user@server.com:destination.jpg
```

This runs:
```bash
scp_image source.jpg user@server.com:destination.jpg
```

If there's no `{}`, the stdin value just gets appended to the end (default behavior).

**Full Pipeline:**
```bash
last_avi | loop last_image | loop scp_image {} sdr@server.com:bambu/snapshot.jpg | loop check | loop notify Bambu
```

This:
1. `last_avi` checks every N minutes for new videos and downloads them
2. `loop last_image` extracts the last frame when a video arrives
3. `loop scp_image` uploads the image to your web server
4. `loop check` analyzes the image with AI to detect failures
5. `loop notify Bambu` sends a notification if a failure is detected

Each step only fires when the previous step outputs something, so you only get notifications when there's actually a new image that looks like a failing print.

## Requirements

### Software Dependencies

- Python 3.6+
- `curl` - for FTPS downloads from printer
- `ffmpeg` - for video frame extraction
- `scp` / `openssh-client` - for uploading to web server (optional)
- OpenAI API key - for AI print failure detection (optional)
- Pushcut app - for phone notifications (optional)

**Install on Ubuntu/Debian:**
```bash
sudo apt install curl ffmpeg openssh-client python3 python3-pip
pip3 install requests openai
```

**Install on macOS:**
```bash
brew install curl ffmpeg python3
pip3 install requests openai
```

## Script Reference

### last_avi

Downloads the latest timelapse video from your Bambu printer via FTPS.

```bash
last_avi [OPTIONS]
```

**Options:**
- `--once` - Run once and exit (no loop)
- `-i SECONDS` - Update interval in seconds (overrides config)
- `--debug` - Show connection details

**Output:**
- Prints video path to stdout only when a new/changed video is downloaded
- Status messages go to stderr

**Examples:**
```bash
# Run once
./last_avi --once

# Loop with 60 second interval
./last_avi -i 60

# Debug mode
./last_avi --debug --once
```

**Configuration:**

First run creates `.bambu_config`:
```bash
PRINTER_HOST=192.168.1.100
ACCESS_CODE=12345678
UPDATE_INTERVAL=300
FTP_PORT=990
FTP_USER=bblp
TIMELAPSE_PATH=/timelapse/
```

This is the only script that runs in a loop on its own. It checks back every N seconds and downloads the video only when it changes. Unfortunately, I couldn't find a "most recent image" endpoint and couldn't download partial AVIs, so we download the whole thing. It's LAN traffic, so shouldn't be a problem.

### last_image

Extracts the last frame from a video file.

```bash
last_image [VIDEO_FILE] [OPTIONS]
```

**Arguments:**
- `VIDEO_FILE` - Path to video file (required)

**Options:**
- `--output PATH`, `-o PATH` - Output path (default: `./bambu_images/latest.jpg`)

**Output:**
- Prints image path to stdout on success
- Status messages to stderr

**Examples:**
```bash
# Default output location
last_image bambu_videos/video_2025-12-19_18-58-33.avi

# Custom output
last_image bambu_videos/video_2025-12-19_18-58-33.avi -o /tmp/snapshot.jpg

# In pipeline
last_avi --once | loop last_image
```

### loop

Reads lines from stdin and executes a command for each line.

```bash
loop <command> [args...]
```

**Placeholder Syntax:**
- Use `{}` to insert the stdin value at a specific position
- Without `{}`, stdin value is appended to the end

**Examples:**
```bash
# Append stdin to end (default)
echo "file.mp4" | loop last_image

# Use {} placeholder
echo "source.jpg" | loop cp {} destination.jpg

# Multiple placeholders
echo "input.txt" | loop convert {} -resize 800x600 {}.resized.jpg
```

**Debug Mode:**
```bash
# See what loop receives
echo "test" | DEBUG=1 loop echo
```

### scp_image

Uploads a file to a remote server via SCP.

```bash
scp_image <file> <destination>
```

**Arguments:**
- `file` - File to upload
- `destination` - SCP destination (user@host:path)

**Output:**
- Prints file path to stdout (passes through for pipeline)
- Status messages to stderr

**Examples:**
```bash
# Direct usage
scp_image snapshot.jpg user@server.com:bambu/

# In pipeline with {}
echo "snapshot.jpg" | loop scp_image {} user@server.com:bambu/
```

### check

Analyzes an image using AI vision to detect print failures.

```bash
check [OPTIONS] <image>
```

**Arguments:**
- `image` - Path to image file

**Options:**
- `-m MODEL`, `--model MODEL` - Model to use (default: gpt-4o-mini)

**Output:**
- Prints image path to stdout ONLY if failure detected
- Model response logged to stderr

**Examples:**
```bash
# Check a single image
check bambu_images/latest.jpg

# Use different model
check -m gpt-4o badprint.jpg

# In pipeline - only continues if failure detected
last_image video.avi | loop check | loop notify Bambu
```

The AI is instructed to be moderately conservative - it won't flag prints that are just starting, and looks for clear signs of failure like spaghetti, detachment, or layer shifts.

### notify

Sends a notification via Pushcut webhook.

```bash
notify <notification_name> [trigger_value]
```

**Arguments:**
- `notification_name` - Name of Pushcut notification to trigger
- `trigger_value` - Optional value to pass through pipeline

**Environment:**
- `PUSHCUT_WEBHOOK_SECRET` - Required environment variable

**Output:**
- Passes trigger_value to stdout if provided
- Status messages to stderr

**Examples:**
```bash
# Simple notification
notify Bambu

# In pipeline (passes through trigger value)
echo "image.jpg" | loop notify Bambu
```

**Pushcut Setup:**
1. Download Pushcut app
2. Create a notification (any name, e.g., "Bambu")
3. Get webhook secret from settings
4. Add to your start script: `export PUSHCUT_WEBHOOK_SECRET=your-secret-here`

## Complete Pipeline Examples

### Full Monitoring with Notifications
```bash
export OPENAI_API_KEY=sk-proj-...
export PUSHCUT_WEBHOOK_SECRET=...
export PYTHONUNBUFFERED=1

last_avi | \
  loop last_image | \
  loop scp_image {} user@server.com:bambu/snapshot.jpg | \
  loop check | \
  loop notify Bambu
```

### Local Monitoring (No Upload, No Notifications)
```bash
last_avi | loop last_image
```

### Upload Only (No AI Check)
```bash
last_avi | loop last_image | loop scp_image {} user@server.com:bambu/snapshot.jpg
```

### Check Only (No Upload)
```bash
last_avi | loop last_image | loop check | loop notify Bambu
```

### Custom Output Location
```bash
last_avi | loop last_image -o ~/Desktop/current_print.jpg
```

## Running as a Service

### systemd (Linux)

Create `/etc/systemd/system/bambu-monitor.service`:

```ini
[Unit]
Description=Bambu P1S Print Monitor
After=network.target

[Service]
Type=simple
User=youruser
WorkingDirectory=/home/youruser/bambu-monitor
Environment="PYTHONUNBUFFERED=1"
Environment="OPENAI_API_KEY=sk-proj-..."
Environment="PUSHCUT_WEBHOOK_SECRET=..."
ExecStart=/home/youruser/bambu-monitor/start
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

# View logs
sudo journalctl -u bambu-monitor -f
```

### screen/tmux

```bash
screen -S bambu
cd ~/bambu-monitor
./start
# Press Ctrl+A, then D to detach

# Reattach later
screen -r bambu
```

## Web Interface (Optional)

Create a simple `status.html` on your web server:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Print Monitor</title>
    <meta http-equiv="refresh" content="60">
</head>
<body>
    <h1>Current Print Status</h1>
    <img src="snapshot.jpg" style="max-width: 100%; height: auto;">
    <p>Auto-refreshes every 60 seconds</p>
</body>
</html>
```

Place it in the same directory where `scp_image` uploads `snapshot.jpg`.

## Troubleshooting

### Video downloads but last_image never runs

- Make sure `export PYTHONUNBUFFERED=1` is in your start script
- Python's stdout buffering can delay pipe data without this

### loop not receiving data

```bash
# Test loop directly
echo "test" | loop echo

# Enable debug mode
echo "test" | DEBUG=1 loop echo
```

### Stderr messages not showing

The `loop` command lets stderr pass through. If you're not seeing messages from scripts in the pipeline, check that loop isn't being redirected.

### "Failed to list directory"

- Verify printer is on: `ping your-printer-ip`
- Check access code (8 digits from printer's WiFi settings)
- Try with `--debug`: `last_avi --debug --once`

### "ffmpeg not found"

```bash
# Install ffmpeg
sudo apt install ffmpeg  # Ubuntu/Debian
brew install ffmpeg      # macOS
```

### "scp command not found"

```bash
sudo apt install openssh-client  # Ubuntu/Debian
brew install openssh             # macOS
```

### SCP asks for password

Set up SSH keys:
```bash
ssh-keygen -t ed25519
ssh-copy-id user@yourserver.com
```

### OpenAI API errors

- Verify API key: `export OPENAI_API_KEY=sk-proj-...`
- Check you have credits
- Try a different model: `check -m gpt-4o-mini image.jpg`

### Pushcut notifications not working

- Verify webhook secret: `export PUSHCUT_WEBHOOK_SECRET=...`
- Test directly: `notify TestNotification`
- Check notification exists in Pushcut app

### "Video unchanged" but there's a new print

Reset state tracking:
```bash
rm bambu_videos/.last_avi_state
```

## File Locations

- **Configuration**: `.bambu_config`
- **Downloaded videos**: `./bambu_videos/video_*.avi`
- **State tracking**: `./bambu_videos/.last_avi_state`
- **Extracted images**: `./bambu_images/latest.jpg` (default)

## Advanced Usage

### Multiple Printers

Create separate directories:
```bash
mkdir printer1 printer2

cd printer1
ln -s ../last_avi ../last_image ../scp_image ../check ../notify ../loop .
./last_avi --once  # Configure for printer1

cd ../printer2
ln -s ../last_avi ../last_image ../scp_image ../check ../notify ../loop .
./last_avi --once  # Configure for printer2
```

### Process All Videos in Archive

```bash
for video in bambu_videos/*.avi; do
  last_image "$video" -o "frames/$(basename "$video" .avi).jpg"
done
```

### Create Progress Timelapse

```bash
# Extract last frame from each video
for video in bambu_videos/*.avi; do
  last_image "$video" -o "frames/$(basename "$video" .avi).jpg"
done

# Combine into video
ffmpeg -framerate 1 -pattern_type glob -i 'frames/*.jpg' \
  -c:v libx264 -pix_fmt yuv420p all_prints.mp4
```

### Custom AI Models

Try different models for failure detection:
```bash
# Fast and cheap
check -m gpt-4o-mini image.jpg

# More accurate
check -m gpt-4o image.jpg

# Latest
check -m gpt-5.1 image.jpg
```

I'm finding gpt-5.1 seems to be the best at detecting failures.

## How It Works

### Unix Pipeline Philosophy

Each script follows Unix principles:
- Does one thing well
- Accepts input via stdin or arguments  
- Outputs results to stdout
- Logs status to stderr
- Chains with pipes
- Fails gracefully

### The Loop Mechanism

`last_avi` is the only script that loops internally. Everything else is stateless and designed to be called by `loop`:

1. `last_avi` checks for new videos every N seconds
2. When it finds one, it prints the path to stdout
3. `loop` reads that path and calls `last_image` with it
4. `last_image` extracts a frame and prints the image path
5. `loop` reads that and calls `scp_image` 
6. `scp_image` uploads and passes the path through
7. `loop` calls `check` with the image path
8. `check` only outputs if failure detected
9. `loop` calls `notify` only if `check` output something

This creates a reactive pipeline where each stage only fires when the previous stage has something to say.

### Buffering

Python normally buffers stdout when piping. We fix this with:
- `export PYTHONUNBUFFERED=1` - disables Python's pipe buffering
- `sys.stdout.flush()` after every print - forces immediate output
- `loop` uses `readline()` not `for line in sys.stdin:` - avoids iterator buffering

## License

MIT License - modify and distribute freely!

## Credits

Built with Unix philosophy and 3D printing frustration.
