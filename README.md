# Chrome Enterprise Policies via Kernel CLI

This guide demonstrates how to configure Chromium Enterprise Policies in a Kernel browser session using the Kernel CLI's filesystem and process execution APIs.

## Overview

It **is possible** to tweak Chromium enterprise policies in a Kernel browser session by:
1. Writing policy JSON files to `/etc/chromium/policies/managed/` using the filesystem API
2. Restarting Chromium via `supervisorctl restart chromium` using the process exec API
3. Verifying the policies took effect by visiting `chrome://policy`

## Prerequisites

- Kernel CLI installed (`brew install kernel/tap/kernel`)
- A Kernel API key

## Quick Start

### 1. Set your API key

```bash
export KERNEL_API_KEY='your_api_key_here'
```

### 2. Create a browser session

```bash
kernel browsers create --timeout 600 -o json
```

Save the `session_id` from the output (e.g., `guf89hv1yyh2d2oe2eyhauh7`).

### 3. Check existing policies

```bash
kernel browsers process exec <session_id> --as-root -- cat /etc/chromium/policies/managed/policy.json
```

### 4. Create a custom policy file locally

```bash
cat > /tmp/custom_policy.json << 'EOF'
{
  "IncognitoModeAvailability": 1,
  "BookmarkBarEnabled": true,
  "DeveloperToolsAvailability": 2,
  "DefaultBrowserSettingEnabled": false,
  "HomepageLocation": "https://example.com/managed-by-enterprise-policy"
}
EOF
```

### 5. Upload the policy file to the browser VM

```bash
kernel browsers fs write-file <session_id> \
  --source /tmp/custom_policy.json \
  --path /etc/chromium/policies/managed/custom_policy.json \
  --mode 0644
```

### 6. Restart Chromium to apply policies

```bash
kernel browsers process exec <session_id> --as-root -- supervisorctl restart chromium
```

Wait a few seconds for Chromium to fully restart.

### 7. Verify policies using Playwright Execution API

Use the Playwright Execution API to navigate to `chrome://policy`, export the policies as JSON, and take a screenshot:

```bash
kernel browsers playwright exec <session_id> --code '
await page.goto("chrome://policy");
await page.waitForTimeout(2000);

// Click reload policies
await page.click("#reload-policies");
await page.waitForTimeout(1000);

// Export policies to clipboard
await page.click("#more-actions-button");
await page.waitForTimeout(500);
await page.click("#copy-policies");
await page.waitForTimeout(500);

// Read policies from clipboard
const policies = await page.evaluate(async () => {
  return await navigator.clipboard.readText();
});

return JSON.parse(policies);
'
```

To take a screenshot of the policy page:

```bash
kernel browsers take-screenshot <session_id> --to policy_screenshot.png
```

### 8. Alternatively, verify via process exec

You can also check the policy files directly on the filesystem:

```bash
kernel browsers process exec <session_id> --as-root -- ls -la /etc/chromium/policies/managed/
kernel browsers process exec <session_id> --as-root -- cat /etc/chromium/policies/managed/custom_policy.json
```

## Policy File Locations

In Kernel's browser VMs (based on Linux/Chromium):

| Path | Description |
|------|-------------|
| `/etc/chromium/policies/managed/` | Mandatory policies (enforced, users cannot override) |
| `/etc/chromium/policies/recommended/` | Recommended policies (users can override) |
| `/etc/chromium/master_preferences` | Initial browser preferences |

## Example Policies

### Disable Incognito Mode

```json
{
  "IncognitoModeAvailability": 1
}
```

Values: `0` = Enabled, `1` = Disabled, `2` = Force (only incognito)

### Disable Developer Tools

```json
{
  "DeveloperToolsAvailability": 2
}
```

Values: `0` = Allowed, `1` = Allowed for extensions, `2` = Disabled

### Set Homepage

```json
{
  "HomepageLocation": "https://your-company.com",
  "HomepageIsNewTabPage": false,
  "RestoreOnStartup": 4,
  "RestoreOnStartupURLs": ["https://your-company.com"]
}
```

### Disable Password Manager

```json
{
  "PasswordManagerEnabled": false
}
```

### Force Extensions

```json
{
  "ExtensionInstallForcelist": [
    "extension_id_here;https://clients2.google.com/service/update2/crx"
  ]
}
```

## Complete Script Example

```bash
#!/bin/bash
set -e

export KERNEL_API_KEY='your_api_key_here'

# Create browser
echo "Creating browser session..."
SESSION_JSON=$(kernel browsers create --timeout 600 -o json)
SESSION_ID=$(echo "$SESSION_JSON" | jq -r '.session_id')
LIVE_VIEW=$(echo "$SESSION_JSON" | jq -r '.browser_live_view_url')

echo "Session ID: $SESSION_ID"
echo "Live View: $LIVE_VIEW"

# Create policy file
echo "Creating policy file..."
cat > /tmp/enterprise_policy.json << 'EOF'
{
  "IncognitoModeAvailability": 1,
  "BookmarkBarEnabled": true,
  "DeveloperToolsAvailability": 2,
  "HomepageLocation": "https://example.com/managed"
}
EOF

# Upload policy
echo "Uploading policy..."
kernel browsers fs write-file "$SESSION_ID" \
  --source /tmp/enterprise_policy.json \
  --path /etc/chromium/policies/managed/enterprise.json \
  --mode 0644

# Verify upload
echo "Verifying policy file..."
kernel browsers process exec "$SESSION_ID" --as-root -- cat /etc/chromium/policies/managed/enterprise.json

# Restart Chromium
echo "Restarting Chromium..."
kernel browsers process exec "$SESSION_ID" --as-root -- supervisorctl restart chromium

# Wait for restart
sleep 5

# Verify policies via Playwright Execution API
echo "Verifying policies via chrome://policy..."
kernel browsers playwright exec "$SESSION_ID" --code '
await page.goto("chrome://policy");
await page.waitForTimeout(2000);
await page.click("#reload-policies");
await page.waitForTimeout(1000);
await page.click("#more-actions-button");
await page.waitForTimeout(500);
await page.click("#copy-policies");
await page.waitForTimeout(500);
const policies = await page.evaluate(async () => await navigator.clipboard.readText());
return JSON.parse(policies);
'

# Take screenshot
echo "Taking screenshot..."
kernel browsers take-screenshot "$SESSION_ID" --to chrome_policy.png

echo "Done! Check chrome_policy.png to verify policies."
echo "Live view: $LIVE_VIEW"
```

## Important Notes

1. **Session Persistence**: Policy files are only persisted within the current browser session. Creating a new browser session starts with a fresh filesystem.

2. **Policy Merging**: Multiple JSON files in `/etc/chromium/policies/managed/` are merged. Later files in alphabetical order can override earlier ones.

3. **Root Access**: Writing to `/etc/chromium/policies/` requires root access. The `--as-root` flag or filesystem API handles this automatically.

4. **Restart Required**: Most enterprise policies require a Chromium restart to take effect. Use `supervisorctl restart chromium` after uploading policy files.

## API Reference

### Filesystem Operations

```bash
# Write a file
kernel browsers fs write-file <session_id> --source <local_file> --path <remote_path> --mode <octal>

# Read a file
kernel browsers fs read-file <session_id> --path <remote_path>

# List files
kernel browsers fs list-files <session_id> --path <directory>
```

### Process Execution

```bash
# Execute command as root
kernel browsers process exec <session_id> --as-root -- <command> [args...]

# Execute command as user
kernel browsers process exec <session_id> -- <command> [args...]
```

### Playwright Execution

```bash
# Execute Playwright code
kernel browsers playwright exec <session_id> --code '<playwright_code>'

# Take screenshot
kernel browsers take-screenshot <session_id> --to <output.png>
```

### Computer Controls (Alternative)

```bash
# Type text
kernel browsers computer type <session_id> --text "text to type"

# Press keys
kernel browsers computer press-key <session_id> --key <key> [--hold-key <modifier>]

# Take screenshot
kernel browsers computer screenshot <session_id> --to <output.png>
```

## Verified Policies

The following policies were tested and confirmed working:

| Policy | Value | Effect |
|--------|-------|--------|
| `IncognitoModeAvailability` | `1` | Disables incognito mode |
| `BookmarkBarEnabled` | `true` | Shows bookmark bar |
| `DeveloperToolsAvailability` | `2` | Disables DevTools |
| `DefaultBrowserSettingEnabled` | `false` | Disables "make default" prompts |
| `HomepageLocation` | URL | Sets custom homepage |
| `PasswordManagerEnabled` | `false` | Disables password saving |
| `AutofillCreditCardEnabled` | `false` | Disables credit card autofill |
| `TranslateEnabled` | `false` | Disables translation prompts |

## References

- [Chrome Enterprise Policy List](https://chromeenterprise.google/policies/)
- [Kernel Browser Documentation](https://docs.onkernel.com/browsers)
- [Kernel CLI Reference](https://docs.onkernel.com/reference/cli)
