# Human-in-the-Loop Login State Persistence

## Overview

NavCLI supports a **human-in-the-loop** workflow for authentication, allowing users to manually log in to websites and persist the session for headless browser automation.

This is essential because:
- Many websites use complex authentication (CAPTCHA, 2FA, security questions)
- Automated login is often blocked or rate-limited
- Some sites require human verification

## Why Session Persistence Matters

### The Chrome Cookie Encryption Problem

On macOS, Chrome stores cookies in an **encrypted format** using the system keychain. This means:

| Data Type | Readable by Playwright? |
|-----------|------------------------|
| Bookmarks | ✓ Yes |
| History | ✓ Yes |
| Extensions | ✓ Yes |
| Preferences | ✓ Yes |
| **Cookies** | ✗ No (encrypted) |

Playwright cannot decrypt Chrome's encrypted cookies when running in headless mode.

### The Solution: Human-in-the-Loop

1. Start browser in **non-headless mode** (`--no-headless`)
2. Manually complete authentication
3. **Save session** to a file
4. **Load session** in headless mode for automation

## Quick Start

### Step 1: Start Non-Headless Browser

```bash
# Start with GUI visible
nav-server --no-headless
```

A browser window will open. Manually:
1. Navigate to the target website
2. Complete login (enter credentials, 2FA, etc.)
3. Verify you're logged in

### Step 2: Save Session

```bash
# Save cookies and session data
curl -X POST "http://localhost:8765/cmd/session/save?path=session.json"

# Response:
# {"success": true, "feedback": {"result": "saved to session.json: 7 cookies, 1 origins"}}
```

### Step 3: Use in Headless Mode

```bash
# Stop the no-headless server (Ctrl+C or pkill)
# Start headless server
nav-server

# Load the saved session
curl -X POST "http://localhost:8765/cmd/session/load?path=session.json"

# Now you can access authenticated content
curl -X POST "http://localhost:8765/cmd/goto?url=https://example.com/protected"
curl http://localhost:8765/query/tables  # Returns full data
```

## Environment Variables

### Using Custom Browser

Some websites require specific browser features. You can specify a custom Chromium executable:

```bash
export PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
nav-server --no-headless
```

### User Data Directory (Advanced)

You can also specify a user data directory for persistent settings:

```bash
export NAVCLI_USER_DATA_DIR="$HOME/Library/Application Support/Google/Chrome/Default"
nav-server --no-headless
```

**Note:** This loads browser settings but may not decrypt existing cookies from a different Chrome installation.

## API Reference

### Save Session

```http
POST /cmd/session/save?path=<file_path>
```

**Parameters:**
- `path` (required): Path to save the session file

**Response:**
```json
{
  "success": true,
  "command": "save_session",
  "feedback": {
    "action": "saved session",
    "result": "saved to session.json: 7 cookies, 1 origins"
  }
}
```

### Load Session

```http
POST /cmd/session/load?path=<file_path>
```

**Parameters:**
- `path` (required): Path to the session file

**Response:**
```json
{
  "success": true,
  "command": "load_session",
  "feedback": {
    "action": "loaded session",
    "result": "loaded session from session.json"
  }
}
```

### Get Cookies

```http
GET /cmd/cookies
```

**Response:**
```json
{
  "success": true,
  "command": "cookies",
  "feedback": {
    "action": "got cookies",
    "result": "found 6 cookies"
  }
}
```

## Best Practices

### 1. Security

- **Never commit session files to version control**
- Add to `.gitignore`:
  ```
  *.json
  !README.md
  ```
- Use descriptive filenames: `google-session.json`, `jisilu-session.json`

### 2. Session Expiry

Web sessions expire. Watch for:
- Logged-out state (reduced data)
- Redirects to login page
- Session-specific errors

Re-authenticate when needed.

### 3. Multiple Accounts

Maintain separate session files:

```bash
# Personal account
curl -X POST "http://localhost:8765/cmd/session/save?path=personal.json"

# Work account
curl -X POST "http://localhost:8765/cmd/session/save?path=work.json"

# Load specific account
curl -X POST "http://localhost:8765/cmd/session/load?path=personal.json"
```

### 4. Verify Before Automation

Always verify the session is valid:

```bash
# Check if logged in
curl http://localhost:8765/query/state | jq '.state.title'

# Test authenticated endpoint
curl http://localhost:8765/query/tables | jq '.state.tables | length'
```

## Troubleshooting

### Cookies Not Loaded

**Symptom:** Session loads but still shows guest state.

**Solution:**
1. Verify session file exists and has content:
   ```bash
   cat session.json | jq '.cookies | length'
   ```
2. Ensure correct path when loading
3. Try re-authenticating

### Session Expired

**Symptom:** Data missing or redirected to login.

**Solution:**
1. Re-run human-in-the-loop login
2. Save new session
3. Load new session

### Browser Won't Start

**Symptom:** `Address already in use` or launch errors.

**Solution:**
```bash
# Kill existing server
pkill -f nav-server

# Start fresh
nav-server --no-headless
```

## Complete Example: Jisilu.cn

[Jisilu](https://www.jisilu.cn) is a Chinese investment data platform that requires login to view complete data.

### Guest Mode (Limited)

```bash
nav-server
curl -X POST "http://localhost:8765/cmd/goto?url=https://www.jisilu.cn/data/cf/#stock"
curl http://localhost:8765/query/tables
# Returns only 20 rows with "tourist" message
```

### With Login Persistence

```bash
# 1. Start non-headless
nav-server --no-headless
# → Browser opens, manually log in to jisilu.cn

# 2. Save session
curl -X POST "http://localhost:8765/cmd/session/save?path=jisilu.json"
# → Saved 7 cookies, 1 origins

# 3. Stop server (Ctrl+C)

# 4. Start headless
nav-server

# 5. Load session
curl -X POST "http://localhost:8765/cmd/session/load?path=jisilu.json"

# 6. Access full data
curl -X POST "http://localhost:8765/cmd/goto?url=https://www.jisilu.cn/data/cf/#stock"
curl http://localhost:8765/query/tables
# → Returns all 31+ rows
```

## Related Documentation

- [CLI Commands Reference](../CLAUDE.md)
- [API Endpoints](../CLAUDE.md#api-protocol)
- [PRD: Product Requirements](./NAVCLI_PRD.md)
