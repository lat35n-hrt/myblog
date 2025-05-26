+++
date = '2025-05-26T18:30:44+09:00'
draft = false
title = 'Verify External File Copy'
+++


# How I Troubleshooted a Mac File Copy Error: Verifying Missing Files, Resource Forks, and Useful Commands
While transferring GoPro files (MP4/JPG) from my Mac to an external USB drive, I encountered a puzzling issue: after an interrupted copy, restarting the copy operation seemed to work—but when I checked, the number of files didn’t match. Here's how I investigated and resolved the issue using find, comm, and some macOS-specific knowledge.

## Situation: Copy Interrupted by Closing the Mac
I started copying around 130 files from my Mac (macOS) to an external USB drive. Midway through, I accidentally closed my Mac, interrupting the process.

After reopening and attempting the copy again, the system stopped almost immediately. Since many files had already been copied, macOS likely skipped files with the same name. But I wasn’t sure if all the files made it through safely.

## Goal and Constraints
Goal: Ensure all original files were successfully copied to the external device before deleting the originals from my Mac.

## Constraints:

The interrupted copy may have missed some files.

The external device may now include temporary or duplicate files.

Data loss must be avoided at all costs.

## Step 1: File Count Check
The first step was to check how many files were on the external device. However, a simple find command returned too many files, including macOS system files like .Spotlight-V100, .Trashes, and .fseventsd.

Improved Command (with exclusion):

``` bash
find /Volumes/GoPro2022 \( -path "*/.Spotlight-V100" -o -path "*/.Trashes" -o -path "*/.fseventsd" \) -prune -o -type f \( -iname "*.jpg" -o -iname "*.mp4" \) -print | wc -l
-prune: Skips unwanted system directories

```

-type f: Targets only files

-iname: Case-insensitive file extension filter

Result:

244
Strange—there were only 130 files in the original folder. Where did the rest come from?

## Step 2: Identifying Resource Forks (._ Files)
macOS can generate resource fork files (starting with ._) when copying via Finder. These are hidden files that store metadata or Finder-specific info.

### Command to Find Resource Fork Files:

``` bash
find /Volumes/GoPro2022 -type f -name "._*" | wc -l

```
Result: 115

So the full count becomes roughly 129 (actual files) + 115 (resource forks) ≈ 244.
Now the mystery of the extra files was solved.

## Step 3: File Name Comparison to Find Missing Files
To ensure no files were missed, I compared the file names only (ignoring paths and metadata) between the source and destination.

### Extract Basename and Sort (External Device):

``` bash
find /Volumes/GoPro2022 \( -path "*/.Spotlight-V100" -o -path "*/.Trashes" -o -path "*/.fseventsd" \) -prune -o -type f ! -name "._*" \( -iname "*.jpg" -o -iname "*.mp4" \) -print0 | xargs -0 -n 1 basename | sort > external_files.txt

```



### Extract Basename and Sort (Mac Source):

``` bash
find /Users/machine/Documents/HERO7_BLACK/2022/ -type f ! -name "._*" \( -iname "*.jpg" -o -iname "*.mp4" \) -print0 | xargs -0 -n 1 basename | sort > mac_files.txt

```

### Compare with comm:


``` bash
comm -23 mac_files.txt external_files.txt > missing_files.txt

```

Result: One file was missing.

## Step 4: Root Cause
It turns out the missing file was due to manual error: I forgot to select one file during the initial drag-and-drop copy.

If comm returns an empty result, it means all files (based on name) match between source and destination.

## Key Takeaways
Topic	Key Points
What is a Resource Fork?	macOS stores metadata as hidden ._ files during Finder-based copies. These are unnecessary on other OSes and can be safely deleted if unused.
Mastering find	Use -prune to exclude unwanted paths, -iname for case-insensitive matching, and -print0 with xargs -0 to handle filenames with spaces.
File Difference Detection with comm	Compare sorted file name lists to detect missing or unmatched files.
Safe Copy Verification	Building a checklist of filename comparison, excluding metadata, ensures reliable copy validation.

## Conclusion
Resource forks caused misleading file counts

File name–based diffing using comm helped verify copy accuracy

Manual drag-and-drop mistakes happen—verification is key


## Bonus: Remove Resource Forks
If you want to clean up ._ files from your external device:



``` bash
find /Volumes/GoPro2022 -type f -name "._*" -delete

```

## ⚠️ Note: These files may be regenerated when copying with Finder. Consider using rsync with appropriate flags to avoid this.

By walking through this real-world situation, I gained valuable insights into macOS’s file system behavior, terminal command usage, and data integrity checks. I hope this article helps anyone dealing with similar file copy issues!

