# Bambu P1S Print Monitor

A Unix-style pipeline system for monitoring your Bambu Lab 3D printer. Downloads timelapse videos, extracts frames, checks for print failures using AI, and sends phone notifications when things go wrong.

## Quick Start - Getting It Running

The system is a set of independent modules that you can run on the command line separately. Each script logs heavily to stderr (so you can see what's happening) but only outputs to stdout when it needs to pass data to the next script in the pipeline.

### Initial Setup

1. **Clone the repo and set up your environment:**
```bash