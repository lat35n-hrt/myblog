+++
date = '2026-01-02T22:52:54+09:00'
draft = false
title = 'Fixing Node.js Version Reverting in New Terminal Windows'
categories = ["Node"]
+++


## The Issue

After switching Node versions using `nvm use`, I found that opening a new terminal window would revert back to the old version.

This article explains how to solve this problem permanently.



## Version Information (Test Environment)

macOS (Intel): 13.7.2
Node.js: 20.2.0 (when problem occurred) → 22.12.0 (after fix)
npm: 10.9.0
nvm: 0.40.3
Vite: 7.2.4
React: 19.2.0


## Problem Symptoms

When using tools like Vite, you may encounter the following error:

```
You are using Node.js 20.2.0. Vite requires Node.js version 20.19+ or 22.12+.
Please upgrade your Node.js version.
```

![error node version](error_node_version.png)


You might think "Wait, I just switched to Node 22..." but checking confirms that it has indeed reverted to the older version.

## Why Does This Happen?

The root cause of this problem is that **nvm is not automatically loaded when opening a new terminal window**.

### How nvm Works

nvm (Node Version Manager) is a tool that allows you to switch between multiple Node.js versions. However, nvm doesn't work automatically. It needs to be initialized every time you start a shell (terminal).

Without initialization, the system falls back to using the old Node.js version that came pre-installed on your system.

## Solution

To permanently solve this problem, we need to add nvm initialization code to your shell configuration files.

### Step 1: Check Current Status

First, open a new terminal window and run the following commands:

```bash
command -v nvm || echo "nvm not loaded"
node -v
```

If you see `nvm not loaded`, it means nvm is not being loaded. This is the cause of the problem.

### Step 2: Add nvm Initialization to Shell Configuration Files

Run the following command to add nvm settings to your `.zshrc` file:

```bash
cat >> ~/.zshrc <<'EOF'

# --- nvm (Homebrew) ---
export NVM_DIR="$HOME/.nvm"
if [ -s "/opt/homebrew/opt/nvm/nvm.sh" ]; then
  . "/opt/homebrew/opt/nvm/nvm.sh"
elif [ -s "/usr/local/opt/nvm/nvm.sh" ]; then
  . "/usr/local/opt/nvm/nvm.sh"
fi

# Automatically apply default Node version
if command -v nvm >/dev/null 2>&1; then
  nvm use --silent default >/dev/null 2>&1
fi
EOF
```

Additionally, add the same settings to `.zprofile`:

```bash
cat >> ~/.zprofile <<'EOF'

# --- nvm (Homebrew) ---
export NVM_DIR="$HOME/.nvm"
if [ -s "/opt/homebrew/opt/nvm/nvm.sh" ]; then
  . "/opt/homebrew/opt/nvm/nvm.sh"
elif [ -s "/usr/local/opt/nvm/nvm.sh" ]; then
  . "/usr/local/opt/nvm/nvm.sh"
fi
EOF
```

### Step 3: Apply the Configuration

Verify that the snippet was appended to ~/.zshrc:

```bash
tail -n 20 ~/.zshrc
```


Restart your shell with the following command:

```bash
exec zsh -l
```

### Step 4: Set Default Version

Set your desired Node.js version as the default:

```bash
nvm alias default 22.12.0
```

This command ensures that Node.js 22.12.0 will be automatically used every time you open a new terminal.

### Step 5: Verify

**Open a new terminal window** and run:

```bash
node -v
```

If it shows `v22.12.0`, you're all set!

## What Does This Configuration Fix?

This configuration ensures the following:

- nvm is automatically loaded every time you open a new terminal
- The default Node.js version you specified is automatically activated
- The system will no longer revert to the old system Node.js version

## Notes

- Running the above commands multiple times will create duplicate entries in your configuration files. If you've already configured them, open `.zshrc` or `.zprofile` in a text editor and remove any duplicate sections
- On Mac, the nvm installation location differs depending on your Mac type (Apple Silicon or Intel). The scripts above work with both

## Summary

The problem of Node.js version reverting can be solved by configuring automatic nvm initialization.

Once configured, you won't need to run the `nvm use` command every time, making your development experience much smoother.