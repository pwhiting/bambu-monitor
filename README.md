# Bambu P1S Print Monitor

A Unix-style pipeline system for monitoring your Bambu Lab 3D printer. Downloads timelapse videos, extracts frames, checks for print failures using AI, and sends phone notifications when things go wrong.

## Quickstart

Get up and running in 3 steps:

**1. Copy and edit the start script:**
```bash
cd ~/bambu-monitor
cp start.sample start
nano start  # or vim, emacs, etc.
```

**2. Edit these values in `start`:**
- `BAMBU_PRINTER_IP` - Your printer's IP address (find in printer's Network settings)
- `BAMBU_ACCESS_CODE` - 8-character code from printer's WiFi settings
- `OPENAI_API_KEY` - Your OpenAI API key from https://platform.openai.com
- `TOPIC` - Pick a unique ntfy.sh topic name (e.g., `bambu-printer-xyz123`)

**3. Run it:**
```bash
chmod +x start
./start
```

That's it! Download the ntfy app on your phone, subscribe to your topic, and you'll get notified of print failures.

## Command Reference

Quick overview of each command:

- **`last-avi`** - Downloads the most recent video from printer, in a loop, only downloading if the video changes; prints filename to stdout
- **`watchdog`** - Reads stdin, passes it through to stdout, and executes a command if timeout is exceeded with no new input
- **`last-image`** - Extracts the last frame from an avi file and saves it as a jpg
- **`extract-images`** - (for testing only) Extracts multiple frames from an avi file with optional step interval
- **`prev-images`** - Finds previous images in a sequence based on filename pattern
- **`check`** - Analyzes one or more images using AI to detect print failures; only outputs if failure detected
- **`notify`** - Sends a notification to ntfy.sh with a custom message
- **`loop`** - Reads lines from stdin and executes a command for each line
- **`scp-image`** - Uploads an image file to a remote server via SCP

---

## Detailed Setup Guide

The system is a set of independent modules that you can run on the command line separately. Each script logs heavily to stderr (so you can see what's happening) but only outputs to stdout when it needs to pass data to the next script in the pipeline.

### Initial Setup

1. **Clone the repo and set up your environment:**

```bash
cd ~/bambu-monitor
chmod +x last-avi last-image extract-images prev-images scp-image check notify watchdog loop

# Create your start script from the sample
cp start.sample start
```

2. **Edit your `start` script with your credentials:**

```bash
# Printer Configuration
export BAMBU_PRINTER_IP=192.168.1.100
export BAMBU_ACCESS_CODE=12345678

# OpenAI API Key
export OPENAI_API_KEY=sk-proj-your-key-here
export PYTHONUNBUFFERED=1

# ntfy.sh topic
TOPIC=your-unique-topic

cd ~/bambu-monitor
PATH=$PATH:~/bambu-monitor

last-avi | watchdog -t 900 notify $TOPIC "no activity - check printer" | loop last-image | loop scp-image {} user@server.com:bambu/snapshot.jpg | loop check | loop notify $TOPIC
```

You'll need:
- **Printer IP and Access Code** - from your printer's settings
- **OpenAI API key** - for the AI vision check (uses GPT-5.1 by default)
- **ntfy.sh topic** - a unique topic name for notifications (completely free)

For ntfy.sh: Just pick a unique topic name (like `bambu-printer-xyz123`). Download the ntfy app on your phone and subscribe to that topic. No signup required!

### Step-by-Step Testing

**Step 1: Download a video**

Set your printer configuration and run `last-avi`:

```bash
export BAMBU_PRINTER_IP=192.168.1.100      # Your printer's IP
export BAMBU_ACCESS_CODE=12345678           # 8-char code from printer

./last-avi --once
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
last-image bambu_videos/video_2025-12-19_18-58-33.avi
```

You should see:
```
last-image: Input: bambu_videos/video_2025-12-19_18-58-33.avi
last-image: Output: /home/pete/bambu-monitor/bambu_images/latest.jpg
last-image: Processing video: bambu_videos/video_2025-12-19_18-58-33.avi
last-image: Extracting last frame → /home/pete/bambu-monitor/bambu_images/latest.jpg
last-image: ✓ Extracted last frame (74,436 bytes)
last-image: ✓ Saved to: /home/pete/bambu-monitor/bambu_images/latest.jpg
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
scp-image bambu_images/latest.jpg user@server.com:bambu/snapshot.jpg
```

**Step 5: Test notification (optional)**

```bash
notify your-unique-topic "test message"
```

This should send a notification to your phone via ntfy.sh. Make sure you've subscribed to the topic in the ntfy app first!

### Understanding the Pipeline

The way these scripts tie together is through Unix pipes. Each script:
- Prints debug/status info to **stderr** (so you can see what's happening)
- Prints filenames to **stdout** only when it needs to trigger the next step

Since all these scripts needed the same looping logic, I created the `loop` command. It listens on stdin and whenever it gets a line, it runs the command with that line as an argument.

**Example:**
```bash
last-avi | loop last-image
```

When `last-avi` downloads a new video, it prints the filename to stdout. `loop` reads it and runs:
```bash
last-image bambu_videos/video_2025-12-19_18-58-33.avi
```

Then `last-image` extracts the frame and prints the image path to stdout, which continues down the chain.

**The `{}` Placeholder:**

Sometimes you need the stdin value in a specific position (not at the end). Use `{}` like `find -exec`:

```bash
echo "source.jpg" | loop scp-image {} user@server.com:destination.jpg
```

This runs:
```bash
scp-image source.jpg user@server.com:destination.jpg
```

If there's no `{}`, the stdin value just gets appended to the end (default behavior).

**Full Pipeline:**
```bash
last-avi | loop last-image | loop scp-image {} sdr@server.com:bambu/snapshot.jpg | loop check | loop notify Bambu
```

This:
1. `last-avi` checks every N minutes for new videos and downloads them
2. `loop last-image` extracts the last frame when a video arrives
3. `loop scp-image` uploads the image to your web server
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
- ntfy app - for phone notifications (optional, completely free)

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

### last-avi

Downloads the latest timelapse video from your Bambu printer via FTPS.

```bash
last-avi [OPTIONS]
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
./last-avi --once

# Loop with 60 second interval
./last-avi -i 60

# Debug mode
./last-avi --debug --once
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

### last-image

Extracts the last frame from a video file.

```bash
last-image [VIDEO_FILE] [OPTIONS]
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
last-image bambu_videos/video_2025-12-19_18-58-33.avi

# Custom output
last-image bambu_videos/video_2025-12-19_18-58-33.avi -o /tmp/snapshot.jpg

# In pipeline
last-avi --once | loop last-image
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
echo "file.mp4" | loop last-image

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

### scp-image

Uploads a file to a remote server via SCP.

```bash
scp-image <file> <destination>
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
scp-image snapshot.jpg user@server.com:bambu/

# In pipeline with {}
echo "snapshot.jpg" | loop scp-image {} user@server.com:bambu/
```

### check

Analyzes one or more images using AI vision to detect print failures. Supports multi-image analysis to detect failures over a sequence of frames.

```bash
check [OPTIONS] <images...>
```

**Arguments:**
- `images` - One or more image paths (comma-delimited or separate arguments)

**Options:**
- `-m MODEL`, `--model MODEL` - Model to use (default: gpt-5.1)

**Output:**
- Prints first image path to stdout ONLY if failure detected
- Model response logged to stderr

**Examples:**
```bash
# Check a single image
check bambu_images/latest.jpg

# Check multiple images (separate arguments)
check image1.jpg image2.jpg image3.jpg

# Check multiple images (comma-delimited)
check "image1.jpg,image2.jpg,image3.jpg"

# Use different model
check -m gpt-4o badprint.jpg

# In pipeline - only continues if failure detected
last-image video.avi | loop check | loop notify Bambu

# Multi-image check in pipeline
extract-images -n 4 video.avi | loop check | loop notify Bambu
```

**Multi-Image Analysis:**

When multiple images are provided, they should be ordered oldest to newest. The AI will analyze the sequence looking for:
- Plastic moving with the extruder (blobs stuck to the nozzle)
- Plastic moving laterally between frames on the build plate
- Progressive failure patterns across frames

The build plate dropping and extruder moving between frames is normal, but the plastic on the build plate shouldn't shift laterally.

The AI is instructed to be moderately conservative - it won't flag prints that are just starting, and looks for clear signs of failure.

### extract-images

Extracts multiple frames from a video file with optional step interval.

```bash
extract-images [OPTIONS] <video>
```

**Arguments:**
- `video` - Path to video file (required)

**Options:**
- `-n COUNT` - Number of frames to extract (default: 1)
- `-s STEP`, `--step STEP` - Step size between frames (default: 1)
- `-p PATH`, `--path PATH` - Output directory (default: ./bambu_images)

**Output:**
- Prints comma-separated list of image paths (oldest to newest)
- Status messages to stderr

**Examples:**
```bash
# Extract last 4 frames from a 20-frame video (frames 17, 18, 19, 20)
extract-images -n 4 video.avi

# Extract every other frame, 4 frames total (frames 14, 16, 18, 20)
extract-images -n 4 -s 2 video.avi

# Custom output directory
extract-images -n 3 -p ~/snapshots video.avi

# In pipeline with multi-image check
extract-images -n 4 video.avi | loop check | loop notify Bambu
```

**How it works:**

The script extracts the last n frames from the video, working backwards from the end with the specified step size. Frames are named using the pattern `image_<timestamp>_f<frame_number>.jpg` and output in oldest-to-newest order (perfect for piping to `check` for sequence analysis).

### prev-images

Finds previous images in a sequence based on filename pattern.

```bash
prev-images [OPTIONS] <image_file>
```

**Arguments:**
- `image_file` - Path to the current image file (required)

**Options:**
- `-n COUNT` - Number of previous images to find (default: 1)

**Output:**
- Prints comma-separated list of image paths (oldest to newest, including current)
- Status messages to stderr

**Examples:**
```bash
# Find the previous image before f18
prev-images bambu_images/image_2025-12-19_08-38-21_f18.jpg
# Output: bambu_images/image_2025-12-19_08-38-21_f16.jpg,bambu_images/image_2025-12-19_08-38-21_f18.jpg

# Find the previous 3 images
prev-images -n 3 bambu_images/image_2025-12-19_08-38-21_f18.jpg
# Output: bambu_images/image_2025-12-19_08-38-21_f2.jpg,...f10.jpg,...f16.jpg,...f18.jpg

# In pipeline for sequence analysis
prev-images -n 3 bambu_images/latest.jpg | loop check | loop notify Bambu
```

**How it works:**

For files matching the pattern `*_f<number>.jpg`, finds other files in the same directory with the same base name and lower frame numbers. Returns them in oldest-to-newest order. Files that don't match the pattern are returned as-is.

### notify

Sends a notification via ntfy.sh.

```bash
notify <topic> [message words...]
```

**Arguments:**
- `topic` - ntfy.sh topic name (required)
- `message words...` - Optional message words (joined with spaces, default: "check your print")

**Output:**
- Status messages to stderr

**Examples:**
```bash
# Simple notification with default message
notify my-bambu-topic

# Custom message (multiple words joined)
notify my-bambu-topic Print failed!

# Multi-word message
notify my-bambu-topic "Check your print now"

# In pipeline
echo "image.jpg" | loop notify my-bambu-topic Possible failure detected
```

**ntfy.sh Setup:**
1. Pick a unique topic name (e.g., `bambu-printer-abc123`)
2. Download ntfy app on your phone (iOS/Android)
3. Subscribe to your topic in the app
4. Use the topic name as the first argument to `notify`

No signup or account required - it's completely free and open source!

### watchdog

Monitors stdin activity and executes a command if no input is received for a timeout period. Passes all input through to stdout.

```bash
watchdog [OPTIONS] <command...>
```

**Options:**
- `-t SECONDS`, `--timeout SECONDS` - Timeout in seconds (default: 900 = 15 minutes)

**Arguments:**
- `command...` - Command to execute on timeout (all remaining arguments)

**Output:**
- Passes stdin through to stdout
- Status messages to stderr
- Executes command when timeout expires

**Examples:**
```bash
# Alert if no videos downloaded for 15 minutes
last-avi | watchdog notify my-topic "no activity - check printer"

# Custom timeout (5 minutes)
last-avi | watchdog -t 300 notify my-topic printer stalled

# In full pipeline
last-avi | watchdog -t 900 notify my-topic "no activity" | loop last-image
```

**How it works:**

The watchdog sits transparently in a pipeline, passing all stdin through to stdout unchanged. Every time data passes through, it resets its timer. If the timer expires (no data for the timeout period), it executes the specified command to alert you (output goes to stderr), then continues monitoring. The watchdog only alerts once per timeout - it won't repeat until new data arrives. This is useful for detecting when upstream processes stop producing output, like when your printer stops recording videos.

## Complete Pipeline Examples

### Full Monitoring with Notifications
```bash
export OPENAI_API_KEY=sk-proj-...
export PYTHONUNBUFFERED=1

TOPIC=your-unique-topic
last-avi | \
  watchdog -t 900 notify $TOPIC "no activity - check printer" | \
  loop last-image | \
  loop scp-image {} user@server.com:bambu/snapshot.jpg | \
  loop check | \
  loop notify $TOPIC
```

### Local Monitoring (No Upload, No Notifications)
```bash
last-avi | loop last-image
```

### Upload Only (No AI Check)
```bash
last-avi | loop last-image | loop scp-image {} user@server.com:bambu/snapshot.jpg
```

### Check Only (No Upload)
```bash
TOPIC=your-unique-topic
last-avi | loop last-image | loop check | loop notify $TOPIC
```

### Custom Output Location
```bash
last-avi | loop last-image -o ~/Desktop/current_print.jpg
```

### Multi-Image Sequence Analysis
```bash
TOPIC=your-unique-topic

# Extract and check last 4 frames for progressive failure detection
last-avi | loop extract-images -n 4 | loop check | loop notify $TOPIC

# Extract with step interval (every other frame)
last-avi | loop extract-images -n 4 -s 2 | loop check | loop notify $TOPIC

# Use prev-images to check current frame plus previous 3
last-avi | loop last-image | loop prev-images -n 3 | loop check | loop notify $TOPIC
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

Place it in the same directory where `scp-image` uploads `snapshot.jpg`.

## Troubleshooting

### Video downloads but last-image never runs

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
- Try with `--debug`: `last-avi --debug --once`

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

### ntfy.sh notifications not working

- Test directly: `notify your-topic "test message"`
- Make sure you've subscribed to the topic in the ntfy app
- Check the ntfy.sh web interface: https://ntfy.sh/your-topic
- Verify curl is installed: `which curl`

### "Video unchanged" but there's a new print

Reset state tracking:
```bash
rm bambu_videos/.last-avi_state
```

## File Locations

- **Configuration**: `.bambu_config`
- **Downloaded videos**: `./bambu_videos/video_*.avi`
- **State tracking**: `./bambu_videos/.last-avi_state`
- **Extracted images**: `./bambu_images/latest.jpg` (default)

## Advanced Usage

### Multiple Printers

Create separate directories:
```bash
mkdir printer1 printer2

cd printer1
ln -s ../last-avi ../last-image ../scp-image ../check ../notify ../loop .
./last-avi --once  # Configure for printer1

cd ../printer2
ln -s ../last-avi ../last-image ../scp-image ../check ../notify ../loop .
./last-avi --once  # Configure for printer2
```

### Process All Videos in Archive

```bash
for video in bambu_videos/*.avi; do
  last-image "$video" -o "frames/$(basename "$video" .avi).jpg"
done
```

### Create Progress Timelapse

```bash
# Extract last frame from each video
for video in bambu_videos/*.avi; do
  last-image "$video" -o "frames/$(basename "$video" .avi).jpg"
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

`last-avi` is the only script that loops internally. Everything else is stateless and designed to be called by `loop`:

1. `last-avi` checks for new videos every N seconds
2. When it finds one, it prints the path to stdout
3. `loop` reads that path and calls `last-image` with it
4. `last-image` extracts a frame and prints the image path
5. `loop` reads that and calls `scp-image` 
6. `scp-image` uploads and passes the path through
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
