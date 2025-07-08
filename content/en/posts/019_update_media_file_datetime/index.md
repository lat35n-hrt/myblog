+++
date = '2025-07-08T19:41:51+09:00'
draft = false
title = 'How I Bulk-Updated MP4 Recording Dates [Aligning Exif and Finder Timestamps]'
categories = ["macos"]
+++




## Environment
- Mac OS: Ventura 13.7.2

- Camera: DJI Pocket 2

- SD Card: SanDisk Extreme 256GB microSD SDXC UHS-1 U3 V30

## 🎯 Why I Did This
When I reviewed videos shot at the Osaka Expo, I found their timestamps were still set to the default initial values.
This made organizing and searching difficult, so I decided to fix both the Exif headers and the “Created” and “Modified” dates visible in Finder.

⚠️ This procedure is for personal use only. Also, note that not all date fields can be modified.

## 1. Bulk Update Exif Dates
Using ExifTool, you can shift DateTimeOriginal (shooting date), CreateDate, and ModifyDate all at once.

📌 Example command:
```` bash
exiftool "-AllDates+=9:5:1 13:17:27" -overwrite_original /Volumes/DCIM/100MEDIA/*.MP4
-AllDates+=9:5:1 13:17:27
````

→ Adds 9 years, 5 months, 1 day, 13 hours, 17 minutes, and 27 seconds to all date fields.

-overwrite_original
→ Overwrites the original files.

✅ Affected Exif fields:
DateTimeOriginal

CreateDate

ModifyDate

⚠️ Note:
Finder’s “Created” and “Modified” timestamps do not change after this step.

## 2. Update Finder’s “Created” and “Modified” Dates
ExifTool can also adjust the filesystem timestamps to match the Exif data.

📌 Example command:
```` bash
exiftool -P -overwrite_original \
  "-FileModifyDate<CreateDate" \
  "-FileCreateDate<CreateDate" \
  /Volumes/Untitled/DCIM/100MEDIA/*.MP4
````

-P preserves the original file permissions.

<CreateDate copies the Exif CreateDate value to the filesystem timestamps.

## ⚠️ Handling setfile errors
This step requires macOS’s setfile command, which is included in Xcode Command Line Tools.
If not installed, run:

```` bash
xcode-select --install
````

## 3. Checking Metadata
To check all timestamp metadata in a file, use:

````bash
exiftool -time:all -a -G1 -s /Volumes/Untitled/DCIM/100MEDIA/test/DJI_0215.MP4
````

🧾 Sample output (excerpt):
```` yaml

[System] FileModifyDate     : 2025:06:09 08:01:14+09:00
[QuickTime] CreateDate      : 2025:06:09 08:01:14
[Track1] MediaCreateDate    : 2016:01:07 18:43:47
[Track2] MediaCreateDate    : 2016:01:07 18:43:47
````

...
## 🚫 Why TrackCreateDate and MediaCreateDate Cannot Be Changed
These fields inside [Track1] to [Track4] are difficult or impossible to edit:

❓ Reason:
They reside in the MP4 file’s binary structure (moov atom) and are not supported for rewriting by ExifTool.

For videos with multiple tracks, safely rewriting these requires specialized video editing software that can fully reconstruct the file structure.

## ✍️ Summary
| Item                          | Editable?                             |
| ----------------------------- | ------------------------------------- |
| Exif shooting dates           | ✅ Can be updated (via `AllDates`)     |
| Finder created/modified dates | ✅ Can be updated (Xcode tools needed) |
| Track-level internal dates    | ❌ Cannot be modified (binary data)    |



ExifTool is powerful but doesn’t support editing MP4 track-level timestamps. Still, aligning Exif and filesystem timestamps is often sufficient in practice.
Try this workflow first to keep your media files organized!

