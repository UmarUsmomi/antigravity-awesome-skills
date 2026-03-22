---
name: Antigravity-in-chrome-troubleshooting
description: Diagnose and fix Antigravity in Chrome MCP extension connectivity issues. Use when mcp__Antigravity-in-chrome__* tools fail, return "Browser extension is not connected", or behave erratically.
---

# Antigravity in Chrome MCP Troubleshooting

Use this skill when Antigravity in Chrome MCP tools fail to connect or work unreliably.

## When to Use

- `mcp__Antigravity-in-chrome__*` tools fail with "Browser extension is not connected"
- Browser automation works erratically or times out
- After updating Antigravity Code or Antigravity.app
- When switching between Antigravity Code CLI and Antigravity.app (Cowork)
- Native host process is running but MCP tools still fail

## When NOT to Use

- **Linux or Windows users** - This skill covers macOS-specific paths and tools (`~/Library/Application Support/`, `osascript`)
- General Chrome automation issues unrelated to the Antigravity extension
- Antigravity.app desktop issues (not browser-related)
- Network connectivity problems
- Chrome extension installation issues (use Chrome Web Store support)

## The Antigravity.app vs Antigravity Code Conflict (Primary Issue)

**Background:** When Antigravity.app added Cowork support (browser automation from the desktop app), it introduced a competing native messaging host that conflicts with Antigravity Code CLI.

### Two Native Hosts, Two Socket Formats

| Component | Native Host Binary | Socket Location |
|-----------|-------------------|-----------------|
| **Antigravity.app (Cowork)** | `/Applications/Antigravity.app/Contents/Helpers/chrome-native-host` | `/tmp/Antigravity-mcp-browser-bridge-$USER/<PID>.sock` |
| **Antigravity Code CLI** | `~/.local/share/Antigravity/versions/<version> --chrome-native-host` | `$TMPDIR/Antigravity-mcp-browser-bridge-$USER` (single file) |

### Why They Conflict

1. Both register native messaging configs in Chrome:
   - `com.anthropic.Antigravity_browser_extension.json` → Antigravity.app helper
   - `com.anthropic.Antigravity_code_browser_extension.json` → Antigravity Code wrapper

2. Chrome extension requests a native host by name
3. If the wrong config is active, the wrong binary runs
4. The wrong binary creates sockets in a format/location the MCP client doesn't expect
5. Result: "Browser extension is not connected" even though everything appears to be running

### The Fix: Disable Antigravity.app's Native Host

**If you use Antigravity Code CLI for browser automation (not Cowork):**

```bash
# Disable the Antigravity.app native messaging config
mv ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_browser_extension.json \
   ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_browser_extension.json.disabled

# Ensure the Antigravity Code config exists and points to the wrapper
cat ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_code_browser_extension.json
```

**If you use Cowork (Antigravity.app) for browser automation:**

```bash
# Disable the Antigravity Code native messaging config
mv ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_code_browser_extension.json \
   ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_code_browser_extension.json.disabled
```

**You cannot use both simultaneously.** Pick one and disable the other.

### Toggle Script

Add this to `~/.zshrc` or run directly:

```bash
chrome-mcp-toggle() {
    local CONFIG_DIR=~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts
    local Antigravity_APP="$CONFIG_DIR/com.anthropic.Antigravity_browser_extension.json"
    local Antigravity_CODE="$CONFIG_DIR/com.anthropic.Antigravity_code_browser_extension.json"

    if [[ -f "$Antigravity_APP" && ! -f "$Antigravity_APP.disabled" ]]; then
        # Currently using Antigravity.app, switch to Antigravity Code
        mv "$Antigravity_APP" "$Antigravity_APP.disabled"
        [[ -f "$Antigravity_CODE.disabled" ]] && mv "$Antigravity_CODE.disabled" "$Antigravity_CODE"
        echo "Switched to Antigravity Code CLI"
        echo "Restart Chrome and Antigravity Code to apply"
    elif [[ -f "$Antigravity_CODE" && ! -f "$Antigravity_CODE.disabled" ]]; then
        # Currently using Antigravity Code, switch to Antigravity.app
        mv "$Antigravity_CODE" "$Antigravity_CODE.disabled"
        [[ -f "$Antigravity_APP.disabled" ]] && mv "$Antigravity_APP.disabled" "$Antigravity_APP"
        echo "Switched to Antigravity.app (Cowork)"
        echo "Restart Chrome to apply"
    else
        echo "Current state unclear. Check configs:"
        ls -la "$CONFIG_DIR"/com.anthropic*.json* 2>/dev/null
    fi
}
```

Usage: `chrome-mcp-toggle` then restart Chrome (and Antigravity Code if switching to CLI).

## Quick Diagnosis

```bash
# 1. Which native host binary is running?
ps aux | grep chrome-native-host | grep -v grep
# Antigravity.app: /Applications/Antigravity.app/Contents/Helpers/chrome-native-host
# Antigravity Code: ~/.local/share/Antigravity/versions/X.X.X --chrome-native-host

# 2. Where is the socket?
# For Antigravity Code (single file in TMPDIR):
ls -la "$(getconf DARWIN_USER_TEMP_DIR)/Antigravity-mcp-browser-bridge-$USER" 2>&1

# For Antigravity.app (directory with PID files):
ls -la /tmp/Antigravity-mcp-browser-bridge-$USER/ 2>&1

# 3. What's the native host connected to?
lsof -U 2>&1 | grep Antigravity-mcp-browser-bridge

# 4. Which configs are active?
ls ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic*.json
```

## Critical Insight

**MCP connects at startup.** If the browser bridge wasn't ready when Antigravity Code started, the connection will fail for the entire session. The fix is usually: ensure Chrome + extension are running with correct config, THEN restart Antigravity Code.

## Full Reset Procedure (Antigravity Code CLI)

```bash
# 1. Ensure correct config is active
mv ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_browser_extension.json \
   ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_browser_extension.json.disabled 2>/dev/null

# 2. Update the wrapper to use latest Antigravity Code version
cat > ~/.Antigravity/chrome/chrome-native-host << 'EOF'
#!/bin/bash
LATEST=$(ls -t ~/.local/share/Antigravity/versions/ 2>/dev/null | head -1)
exec "$HOME/.local/share/Antigravity/versions/$LATEST" --chrome-native-host
EOF
chmod +x ~/.Antigravity/chrome/chrome-native-host

# 3. Kill existing native host and clean sockets
pkill -f chrome-native-host
rm -rf /tmp/Antigravity-mcp-browser-bridge-$USER/
rm -f "$(getconf DARWIN_USER_TEMP_DIR)/Antigravity-mcp-browser-bridge-$USER"

# 4. Restart Chrome
osascript -e 'quit app "Google Chrome"' && sleep 2 && open -a "Google Chrome"

# 5. Wait for Chrome, click Antigravity extension icon

# 6. Verify correct native host is running
ps aux | grep chrome-native-host | grep -v grep
# Should show: ~/.local/share/Antigravity/versions/X.X.X --chrome-native-host

# 7. Verify socket exists
ls -la "$(getconf DARWIN_USER_TEMP_DIR)/Antigravity-mcp-browser-bridge-$USER"

# 8. Restart Antigravity Code
```

## Other Common Causes

### Multiple Chrome Profiles

If you have the Antigravity extension installed in multiple Chrome profiles, each spawns its own native host and socket. This can cause confusion.

**Fix:** Only enable the Antigravity extension in ONE Chrome profile.

### Multiple Antigravity Code Sessions

Running multiple Antigravity Code instances can cause socket conflicts.

**Fix:** Only run one Antigravity Code session at a time, or use `/mcp` to reconnect after closing other sessions.

### Hardcoded Version in Wrapper

The wrapper at `~/.Antigravity/chrome/chrome-native-host` may have a hardcoded version that becomes stale after updates.

**Diagnosis:**
```bash
cat ~/.Antigravity/chrome/chrome-native-host
# Bad: exec "/Users/.../.local/share/Antigravity/versions/2.0.76" --chrome-native-host
# Good: Uses $(ls -t ...) to find latest
```

**Fix:** Use the dynamic version wrapper shown in the Full Reset Procedure above.

### TMPDIR Not Set

Antigravity Code expects `TMPDIR` to be set to find the socket.

```bash
# Check
echo $TMPDIR
# Should show: /var/folders/XX/.../T/

# Fix: Add to ~/.zshrc
export TMPDIR="${TMPDIR:-$(getconf DARWIN_USER_TEMP_DIR)}"
```

## Diagnostic Deep Dive

```bash
echo "=== Native Host Binary ==="
ps aux | grep chrome-native-host | grep -v grep

echo -e "\n=== Socket (Antigravity Code location) ==="
ls -la "$(getconf DARWIN_USER_TEMP_DIR)/Antigravity-mcp-browser-bridge-$USER" 2>&1

echo -e "\n=== Socket (Antigravity.app location) ==="
ls -la /tmp/Antigravity-mcp-browser-bridge-$USER/ 2>&1

echo -e "\n=== Native Host Open Files ==="
pgrep -f chrome-native-host | xargs -I {} lsof -p {} 2>/dev/null | grep -E "(sock|Antigravity-mcp)"

echo -e "\n=== Active Native Messaging Configs ==="
ls ~/Library/Application\ Support/Google/Chrome/NativeMessagingHosts/com.anthropic*.json 2>/dev/null

echo -e "\n=== Custom Wrapper Contents ==="
cat ~/.Antigravity/chrome/chrome-native-host 2>/dev/null || echo "No custom wrapper"

echo -e "\n=== TMPDIR ==="
echo "TMPDIR=$TMPDIR"
echo "Expected: $(getconf DARWIN_USER_TEMP_DIR)"
```

## File Reference

| File | Purpose |
|------|---------|
| `~/.Antigravity/chrome/chrome-native-host` | Custom wrapper script for Antigravity Code |
| `/Applications/Antigravity.app/Contents/Helpers/chrome-native-host` | Antigravity.app (Cowork) native host |
| `~/.local/share/Antigravity/versions/<version>` | Antigravity Code binary (run with `--chrome-native-host`) |
| `~/Library/Application Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_browser_extension.json` | Config for Antigravity.app native host |
| `~/Library/Application Support/Google/Chrome/NativeMessagingHosts/com.anthropic.Antigravity_code_browser_extension.json` | Config for Antigravity Code native host |
| `$TMPDIR/Antigravity-mcp-browser-bridge-$USER` | Socket file (Antigravity Code) |
| `/tmp/Antigravity-mcp-browser-bridge-$USER/<PID>.sock` | Socket files (Antigravity.app) |

## Summary

1. **Primary issue:** Antigravity.app (Cowork) and Antigravity Code use different native hosts with incompatible socket formats
2. **Fix:** Disable the native messaging config for whichever one you're NOT using
3. **After any fix:** Must restart Chrome AND Antigravity Code (MCP connects at startup)
4. **One profile:** Only have Antigravity extension in one Chrome profile
5. **One session:** Only run one Antigravity Code instance

---

*Original skill by [@jeffzwang](https://github.com/jeffzwang) from [@ExaAILabs](https://github.com/ExaAILabs). Enhanced and updated for current versions of Antigravity Desktop and Antigravity Code.*

