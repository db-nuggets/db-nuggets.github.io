---
author: Darryl Buswell
pubDatetime: 2026-02-12T18:00:00Z
modDatetime: 2026-02-12T18:00:00Z
title: "Bringing Apple's Dynamic Wallpapers to Windows"
slug: bringing-apples-dynamic-wallpapers-to-windows
featured: false
draft: false
tags:
  - windows
  - powershell
  - python
  - automation
  - weekendproject
description: "A set of scripts to extract, map, and schedule Apple's day/night HEIC wallpapers on Windows."
---

I caught up with a friend for lunch who just picked up a new MacBook earlier that morning. And they of course had to show it off while we ate. The thing was nice and all, but what actually grabbed my attention was the dynamic wallpaper.

I knew MacOS had animated backgrounds. But this background updated based on the actual time of day, which was something I hadn't seen before. Later that night, I did some searching and found [Dynamic Wallpaper Club](https://dynamicwallpaper.club/), which hosts a collection of these 24-hour day/night cycle wallpapers. The catch? They come exclusively in Apple's HEIC image format.

As Windows doesn't natively support HEIC, I decided to hack something together to get dynamic wallpapers on my machine.

I did this in three-parts:
1.  A **Python HEIC extractor** to pull the images and their timing metadata out of HEIC files.
2.  A **JSON configuration** to define themes and a weekly schedule.
3.  A set of **PowerShell scripts** to create scheduled tasks that update the wallpaper automatically.

## Part 1: Deconstructing the HEIC File

The first hurdle was to reverse-engineer the `.heic` file. I needed not just the images, but the information about when each image should be displayed. I discovered this data is often stored in XMP metadata, which contains a binary Apple Property List (`plist`).

I wrote a Python script to handle this extraction. It uses `pillow_heif` to peek inside the HEIC container and `plistlib` to parse the metadata.

Here’s a simplified look at the logic:

`extractor.py`
```python
import base64
import json
import os
import plistlib
import re
from pillow_heif import open_heif

def extract_metadata(heif_file, output_folder):
    """
    Extracts XMP metadata, finds the embedded plist, and decodes it.
    """
    metadata = {}
    xmp_data = heif_file.info.get('xmp')
    if not xmp_data:
        return

    # Heuristic to find base64 encoded plists
    plist_candidates = re.findall(r"([A-Za-z0-9+/=]{100,})", xmp_data.decode('utf-8'))
    for candidate in plist_candidates:
        try:
            decoded = base64.b64decode(candidate)
            if decoded.startswith(b'bplist'):
                plist_data = plistlib.loads(decoded)
                metadata['plist'] = plist_data
                break
        except Exception:
            continue
    
    # Generate a triggers.json file from the plist data
    create_triggers(metadata, output_folder, "triggers.json")

def extract_images_from_heic(file_path, output_root):
    """
    Processes a HEIC file: extracts all images and metadata into a subfolder.
    """
    heif_file = open_heif(file_path)
    # ... (code to loop through and save each image) ...
    extract_metadata(heif_file, file_output_folder)
```

The script iterates through a folder of `.heic` files. For each one, it creates a new folder containing all the individual JPGs and, most importantly, a `triggers.json` file that maps a time of day to the corresponding image file.

`example/triggers.json`
```json
{
  "triggers": [
    {
      "time": "00:00:00",
      "image": "Atacama_1.png"
    },
    {
      "time": "06:00:00",
      "image": "Atacama_2.png"
    },
    {
      "time": "09:00:00",
      "image": "Atacama_3.png"
    },
    {
      "time": "12:00:00",
      "image": "Atacama_0.png"
    },
    {
      "time": "15:00:00",
      "image": "Atacama_4.png"
    },
    {
      "time": "17:00:00",
      "image": "Atacama_5.png"
    },
    {
      "time": "19:00:00",
      "image": "Atacama_6.png"
    },
    {
      "time": "20:00:00",
      "image": "Atacama_7.png"
    },
    {
      "time": "22:00:00",
      "image": "Atacama_8.png"
    }
  ]
}
```

## Part 2: Creating a Theme Library

With the ability to create image "themes," I needed a way to organize them. So I created a simple JSON-based configuration file where a `schedule` is defined, and `triggers` are linked to their respective `triggers.json` files.

`configs/default.json`
```json
{
  "schedule": {
    "Monday": "Catalina",
    "Tuesday": "BigSur",
    "Wednesday": "Catalina"
    // etc.
  },
  "triggers": {
    "Catalina": "themes/catalina/triggers.json",
    "BigSur": "themes/big_sur/triggers.json"
  }
}
```
This makes it easy to change the weekly wallpaper schedule without touching the core scripts.

## Part 3: Automation with PowerShell and Task Scheduler

The final piece was to make it all run automatically. To avoid having another service constantly running in the background, I chose to use the built-in Windows Task Scheduler.

I created three PowerShell scripts to pull it all together:
-   `Install-Task.ps1`: The main setup script. It cleans up old tasks and then creates new daily and logon tasks based on the specified configuration.
-   `Background-Scheduler.ps1`: A smart entry point used by the logon task. It checks the current day, looks up the schedule in your config file, and applies the correct theme.
-   `Set-Background.ps1`: The worker script. It's triggered on a schedule (e.g., hourly), reads the active theme's `triggers.json`, and sets the desktop background to the correct image for the current time.

Running the installer is a one-time step:
```powershell
.\Install-Task.ps1 -Config "default.json"
```
After that, everything runs silently in the background.

## Conclusion

While it's all a little hacky, it works well enough. I can now grab any dynamic wallpaper from [Dynamic Wallpaper Club](https://dynamicwallpaper.club/) and have a background which changes throughout the day. All without needing to buy a Mac.

If you want to try it out yourself, you can find the complete code and instructions on my GitHub repository.

[**View the project on GitHub: buswedg/windows-helpers**](https://github.com/buswedg/windows-helpers/tree/main/background-scheduler)
