+++
date = '2025-10-25T15:34:30+09:00'
draft = false
title = 'macOS Automation'
+++


## 🪶 plist → sh → py: Automating Python Tasks on macOS

### ✅ Objective

When you need to run a Python script automatically on macOS, the most lightweight and stable method is to use the built-in launchd scheduler.
This post explains how to schedule daily automation by chaining together three components: a property list (.plist), a shell script (.sh), and a Python script (.py).

## 🧩 Overall Execution Flow: plist → sh → py


```scss
launchd (.plist)
    ↓
bash script (.sh)
    ↓
Python script (.py)

```

.plist — Defines the job schedule and triggers execution.

.sh — Prepares the environment, handles paths and logs, and calls Python.

.py — Runs the actual application logic.

This three-layer structure ensures stability, reproducibility, and maintainability.

## ① plist: LaunchAgent Configuration

~/Library/LaunchAgents/com.example.daily.plist

```xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.example.daily</string>

  <key>ProgramArguments</key>
  <array>
    <string>/Users/YourUser/scripts/daily_task.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key><integer>20</integer>
    <key>Minute</key><integer>0</integer>
  </dict>

  <key>KeepAlive</key><true/>
  <key>RunAtLoad</key><true/>
  <key>StandardOutPath</key><string>/tmp/daily_task.log</string>
  <key>StandardErrorPath</key><string>/tmp/daily_task.err</string>
</dict>
</plist>

```

### 🔍 Key Points

| Element                                 | Description                                                                                                              |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Label**                               | Unique job identifier. The reverse domain format (`com.example.daily`) prevents conflicts with other system or app jobs. |
| **ProgramArguments**                    | Specifies the file to execute (the shell script in this case). Arguments are written as an array per XML rules.          |
| **StartCalendarInterval**               | Defines when to run. Here, it runs every day at 20:00.                                                                   |
| **KeepAlive / RunAtLoad**               | Ensures the job can restart after sleep or immediately upon login.                                                       |
| **StandardOutPath / StandardErrorPath** | Redirects output and error streams to log files for debugging.                                                           |


### 🔧 Control Commands

```bash
launchctl load ~/Library/LaunchAgents/com.example.daily.plist
launchctl list | grep example
launchctl unload ~/Library/LaunchAgents/com.example.daily.plist
```


## ② sh: Preparing the Environment and Logging

This intermediate shell script bridges the restrictive launchd environment and your Python environment.
It sets the proper paths, enables the virtual environment, and handles logging and notifications.

```bash
#!/bin/bash
export PATH=/Users/YourUser/dev/App/env/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin
cd /Users/YourUser/dev/App
export PYTHONPATH=.
/Users/YourUser/dev/App/env/bin/python3 scripts/daily_summary_job.py >> /Users/YourUser/dev/App/tmp/daily.log 2>&1
osascript -e 'display notification "Job completed" with title "NewsLite"'
echo "Job completed at $(date)" >> /Users/YourUser/dev/App/tmp/daily.log
echo "--"
```



### 🧩 Explanation of Each Step


| Item                                          | Reason / Effect                                                                                                                                            |
| --------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`which python3` to confirm absolute path**  | `launchd` does not inherit your usual `$PATH`, so specifying the exact Python binary (e.g., from a virtual environment) avoids “command not found” errors. |
| **`export PATH=...`**                         | Rebuilds the environment with explicit paths to Homebrew, `/usr/local/bin`, and your virtual environment, since `launchd` starts with a minimal PATH.      |
| **`cd /Users/.../App`**                       | Ensures relative paths in your code resolve correctly.                                                                                                     |
| **`export PYTHONPATH=.`**                     | Adds the project root to Python’s module search path, allowing local imports.                                                                              |
| **`>> ... 2>&1` unified logging**             | Appends both standard output and errors to the same log file. `2>&1` redirects descriptor 2 (stderr) to descriptor 1 (stdout).                             |
| **Store logs inside the project, not `/tmp`** | `/tmp` is ephemeral and can be wiped after reboot. Keeping logs in `App/tmp/` allows persistent debugging.                                                 |
| **`osascript` notification**                  | Displays a native macOS notification—something cron cannot do. Useful for confirming task completion.                                                      |
| **Timestamp (`date`) output**                 | Records the exact execution time for easy verification of the schedule.                                                                                    |




### 💡 What 2>&1 Really Means

In shell syntax,

```bash
>> logfile 2>&1

```

means:

“Append both standard output (1) and standard error (2) to the same file.”

Without it, only normal output is logged—errors would still go to the console.
With 2>&1, everything ends up in one comprehensive log, making troubleshooting much easier.

## ③ py: The Main Python Logic

Finally, your Python script performs the actual task—fetching data, generating summaries, or any business logic you need.

```python
# scripts/daily_summary_job.py

from datetime import datetime

def main():
    print(f"[{datetime.now()}] Daily job started.")
    # (Your data fetching / transformation / output logic here)
    print("Daily job completed successfully.")

if __name__ == "__main__":
    main()

```


## 🧭 Putting It All Together


| Layer       | File                      | Role                                                           |
| ----------- | ------------------------- | -------------------------------------------------------------- |
| **① plist** | `com.example.daily.plist` | Defines the execution schedule and hands control to `launchd`. |
| **② sh**    | `daily_task.sh`           | Sets up environment variables, logging, and notifications.     |
| **③ py**    | `daily_summary_job.py`    | Runs the main business logic.                                  |



By separating these responsibilities,
you get:

- predictable, reproducible behavior under macOS’s strict launchd environment,

- complete log control,

- and a workflow that requires no third-party tools.