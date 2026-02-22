# Vision Verify — Error Handling Reference

This reference covers all error scenarios for the vision-verify skill, including detection methods, remediation steps, retry behavior, input validation, and graceful degradation guidance.

---

## Error Response Format

Every error returned by vision-verify uses this standard object format:

```json
{
  "type": "recording-failed|cli-missing|model-unavailable|file-too-large|timeout|parse-error|empty-video",
  "message": "human-readable error description",
  "remediation": "actionable steps to resolve"
}
```

| Field | Type | Description |
|:------|:-----|:------------|
| `type` | enum | One of the 7 error type identifiers listed above |
| `message` | string | Human-readable description of what went wrong |
| `remediation` | string | Actionable steps the agent or developer can take to resolve the issue |

---

## Error Scenarios

### 1. `recording-failed`

**Detection:** No video file found at the expected path under `test-results/` after the test run completes.

**Example message:**
```
No video file found at test-results/<test-name>/video.webm after test run.
```

**Example error object:**
```json
{
  "type": "recording-failed",
  "message": "No video file found at test-results/my-test/video.webm after test run completed.",
  "remediation": "Verify Playwright video recording is enabled. Check that `use.video.mode` is set to 'on' in your test run configuration, not 'off' or 'retain-on-failure'. Confirm the test actually executed (not skipped or errored before page load). Check that the test-results/ directory is writable."
}
```

**Remediation steps:**
1. Confirm `use.video.mode` is set to `'on'` in the per-test-run configuration
2. Verify the test did not fail before the browser page was loaded (no page = no recording)
3. Check disk space and write permissions on the `test-results/` directory
4. Re-run the test with `--reporter=list` to see per-test artifact paths
5. Confirm the Playwright version supports video recording (v1.9+)

---

### 2. `cli-missing`

**Detection:** The `gemini` command is not found in PATH. Indicated by shell exit code 127 or the message "command not found" in stderr.

**Example message:**
```
gemini: command not found
```

**Example error object:**
```json
{
  "type": "cli-missing",
  "message": "The `gemini` command was not found in PATH (exit code 127).",
  "remediation": "Install Gemini CLI and ensure it is accessible in your PATH. See installation instructions below."
}
```

**Installation instructions:**

Install Gemini CLI via npm:
```bash
npm install -g @google/gemini-cli
```

Or follow the official instructions at: https://github.com/google-gemini/gemini-cli

After installation, verify it is accessible:
```bash
which gemini
gemini --version
```

Confirm it is working end-to-end (see [Gemini CLI Prerequisites](#gemini-cli-prerequisites) below).

---

### 3. `model-unavailable`

**Detection:** Gemini CLI exits with a model-not-found error, quota-exceeded message, or permission-denied response. Common indicators in stderr: "model not found", "QUOTA_EXCEEDED", "403 Forbidden", "model is not supported".

**Example message:**
```
Error: Model 'gemini-3-pro-preview' not found or quota exceeded.
```

**Example error object:**
```json
{
  "type": "model-unavailable",
  "message": "Gemini CLI returned a model error: 'gemini-3-pro-preview' not found or quota exceeded.",
  "remediation": "Check that the model name is correct and available to your account. Verify API credentials are valid and quota has not been exhausted. Do not retry — this is likely a persistent configuration issue."
}
```

**Remediation steps:**
1. Verify the model name — `gemini-3-pro-preview` may have been renamed or deprecated; check the Gemini API docs for current model identifiers
2. Check API quota in the Google Cloud Console or AI Studio
3. Confirm API credentials are configured and valid
4. If the model has been renamed, override the default by passing the correct model name to the skill
5. Do NOT retry — this is a persistent configuration issue, not a transient failure

**IMPORTANT:** Do not retry on `model-unavailable`. Retrying will not resolve model name errors, quota exhaustion, or permission issues. Surface the error immediately.

---

### 4. `file-too-large`

**Detection:** File size check performed before submission. Default limit is 20MB. Check with `wc -c < video_path` and compare against 20971520 bytes (20 * 1024 * 1024).

**Example message:**
```
Video file size 34.2MB exceeds the 20MB limit.
```

**Example error object:**
```json
{
  "type": "file-too-large",
  "message": "Video file test-results/my-test/video.webm is 34.2MB, which exceeds the 20MB limit.",
  "remediation": "Reduce test duration to produce a shorter recording, lower the viewport resolution (e.g., 1280x720 instead of 1920x1080), or split the test into smaller segments."
}
```

**Remediation steps:**
1. Reduce test duration — focus the test on the specific UI interaction that needs visual verification
2. Lower viewport resolution in the Playwright config: `size: { width: 1280, height: 720 }`
3. Split the test into smaller, focused tests and verify each separately
4. Check file size before submission:
   ```bash
   # Check file size in bytes
   wc -c < test-results/my-test/video.webm

   # Check file size in MB (macOS/Linux)
   ls -lh test-results/my-test/video.webm
   ```

The 20MB limit is the default. If the Gemini CLI upload limit changes, the `maxFileSizeMB` input parameter can be adjusted accordingly.

---

### 5. `timeout`

**Detection:** Gemini CLI does not return a response within the configured timeout period (default: 120 seconds).

**Example message:**
```
Gemini CLI did not respond within 120 seconds.
```

**Example error object:**
```json
{
  "type": "timeout",
  "message": "Gemini CLI analysis timed out after 120 seconds.",
  "remediation": "Retry once with the same parameters. If the timeout recurs, check network connectivity, verify the Gemini API is reachable, and consider reducing video file size."
}
```

**Retry behavior:** Retry **once** with the same parameters before returning the error. If the retry also times out, return the `timeout` error immediately.

**Remediation steps:**
1. Retry the analysis once with identical parameters
2. If the retry fails, check network connectivity to Google's APIs
3. Verify the Gemini API status page for outages
4. Reduce video file size — larger files take longer to upload and process
5. Check for proxy or firewall restrictions blocking outbound connections

---

### 6. `parse-error`

**Detection:** Gemini CLI returns output that does not match the expected JSON structure. The response cannot be parsed as a valid analysis result with `verdict`, `summary`, `details`, and `suggestions` fields.

**Example message:**
```
Gemini response is not valid JSON or does not match the expected result schema.
```

**Example error object:**
```json
{
  "type": "parse-error",
  "message": "Gemini response could not be parsed as a structured analysis result.",
  "remediation": "The raw response is included as a 'warning' verdict for manual review. Do not retry with the same prompt — the model may not be following the output format instructions."
}
```

**Behavior on parse-error:** Return a `warning` verdict that includes the raw Gemini response in the `summary` or `details` field. This allows the developer to manually interpret the output rather than silently discarding it.

```json
{
  "verdict": "warning",
  "summary": "Visual analysis returned an unparseable response. Raw output included below for manual review.",
  "details": [],
  "suggestions": [
    "Review the raw Gemini output below to determine if visual issues were detected.",
    "<raw Gemini output here>"
  ]
}
```

**IMPORTANT:** Do NOT retry with the same prompt on a parse error. The model is unlikely to change its output format on a re-run. Surface the raw response instead.

---

### 7. `empty-video`

**Detection:** The video file exists at the expected path but has 0 bytes or 0 duration. Check with `[ -s video_path ]` (returns false if empty).

**Example message:**
```
Video file exists but is empty (0 bytes).
```

**Example error object:**
```json
{
  "type": "empty-video",
  "message": "Video file test-results/my-test/video.webm exists but has 0 bytes.",
  "remediation": "The test may not have produced visible browser output. Verify the test navigated to a page and that the browser was visible during the run. Re-run the test with headed mode to diagnose."
}
```

**Remediation steps:**
1. Confirm the test actually navigated to a page (not errored before `page.goto()`)
2. Re-run the test in headed mode to observe what the browser displays: `npx playwright test --headed`
3. Check that no browser crash or early exit occurred during recording
4. Verify the Playwright version and video recording settings are compatible

---

## Retry Pattern

| Error Type | Retry Behavior |
|:-----------|:--------------|
| `timeout` | Retry **once** with the same parameters before returning the error |
| `model-unavailable` | **Do NOT retry** — this is a persistent configuration issue |
| `parse-error` | **Do NOT retry** with the same prompt — return raw response as `warning` verdict |
| `recording-failed` | **Do NOT retry** — surface the error immediately |
| `cli-missing` | **Do NOT retry** — surface the error and installation instructions immediately |
| `file-too-large` | **Do NOT retry** — surface the error with size reduction guidance immediately |
| `empty-video` | **Do NOT retry** — surface the error with test execution guidance immediately |

---

## Input Validation

Perform these checks before submitting to Gemini CLI. Catch problems early to give clearer error messages.

### File Existence

Verify the video file exists at the expected path before attempting submission:

```bash
# Check if the file exists
test -f "test-results/my-test/video.webm" && echo "exists" || echo "missing"
```

If missing, return `recording-failed` error immediately without attempting CLI invocation.

### File Size

Check file size against the 20MB limit (configurable via `maxFileSizeMB` input parameter):

```bash
# Get file size in bytes (macOS/Linux)
wc -c < "test-results/my-test/video.webm"

# Get file size human-readable
ls -lh "test-results/my-test/video.webm"

# Check if over 20MB threshold (20 * 1024 * 1024 = 20971520 bytes)
[ $(wc -c < "test-results/my-test/video.webm") -gt 20971520 ] && echo "too large" || echo "ok"
```

If over the limit, return `file-too-large` error immediately without attempting CLI invocation.

### File Format

Verify the file has a `.webm` extension:

```bash
# Check extension
[[ "test-results/my-test/video.webm" == *.webm ]] && echo "valid" || echo "invalid format"
```

Only `.webm` files are supported. Playwright records in WebM format by default.

### Empty File

Check for zero-byte files even when the path resolves correctly:

```bash
# Check if file is non-empty
[ -s "test-results/my-test/video.webm" ] && echo "non-empty" || echo "empty"
```

If the file is 0 bytes, return `empty-video` error immediately without attempting CLI invocation.

### Validation Order

Run checks in this order to give the most specific error:

1. File existence → `recording-failed` if missing
2. Empty file → `empty-video` if 0 bytes
3. File format → surface format warning if not `.webm`
4. File size → `file-too-large` if over limit
5. CLI availability → `cli-missing` if `gemini` not in PATH
6. Proceed with submission

---

## Gemini CLI Prerequisites

The following must be in place before vision-verify can execute analysis:

1. **`gemini` command in PATH** — The Gemini CLI binary must be installed and accessible as `gemini` in the shell environment where Playwright tests run.

2. **API credentials configured** — The skill does not manage authentication. Credentials must be configured independently (e.g., via `GEMINI_API_KEY` environment variable or `gcloud auth`). The skill has no knowledge of credential state.

3. **Model availability** — The `gemini-3-pro-preview` model (or any overridden model name) must be available to the authenticated account. Vision-capable models may require specific API tiers or be subject to quota limits.

### Verifying Gemini CLI is Working

Run this quick check to confirm the CLI is installed, authenticated, and responsive:

```bash
echo "hello" | gemini -y
```

Expected: The CLI responds with a short greeting or acknowledgment. No error about missing credentials, quota, or command not found.

If this command fails, diagnose the issue before running vision-verify:

- `command not found` → Install Gemini CLI (see [cli-missing](#2-cli-missing) above)
- Authentication error → Configure API credentials per the Gemini CLI documentation
- Quota error → Check quota in Google Cloud Console or AI Studio
- Model error → Verify model availability for your account

---

## Graceful Degradation

Vision-verify is advisory only. It must never block test authoring or fail the test suite.

### When Gemini CLI is Unavailable

If the `gemini` command is not found (exit code 127):

1. Log the skip reason clearly: "Skipping visual verification — Gemini CLI not installed (`gemini` not found in PATH)"
2. Continue test authoring without visual verification
3. Suggest the developer install Gemini CLI for future runs
4. Do NOT fail the test, throw an error that stops execution, or mark the test as failed

### When Analysis Returns an Unparseable Response

If Gemini's output cannot be parsed as the expected JSON structure:

1. Return a `warning` verdict (not a hard failure)
2. Include the raw Gemini output in the response so nothing is silently discarded
3. Suggest the developer review the raw output manually
4. Do NOT retry with the same prompt

### Core Principle

> The skill NEVER blocks or fails the test suite. It is purely advisory.

A `fail` verdict from visual analysis means "investigate this" — not "stop the CI pipeline". The agent surfaces findings to the developer and continues. The decision to act on a visual finding belongs to the developer, not to automation.

---

## Troubleshooting Decision Tree

Use this decision tree to diagnose failures systematically:

```
vision-verify failed or returned unexpected results
│
├── Video file missing at expected path?
│   └── YES → recording-failed
│       ├── Check use.video.mode = 'on' in Playwright config
│       ├── Verify test navigated to a page before failing
│       └── Check test-results/ is writable
│
├── Video file exists but is 0 bytes?
│   └── YES → empty-video
│       ├── Run test in headed mode to observe browser
│       └── Check for early test exit before page load
│
├── `gemini` command not found (exit code 127)?
│   └── YES → cli-missing
│       └── Install Gemini CLI: npm install -g @google/gemini-cli
│
├── Gemini CLI returned model error or quota error?
│   └── YES → model-unavailable
│       ├── Verify model name (gemini-3-pro-preview may be renamed)
│       ├── Check API quota in Google Cloud Console
│       └── Verify API credentials are valid
│
├── Video file is larger than 20MB?
│   └── YES → file-too-large
│       ├── Reduce test duration
│       ├── Lower viewport resolution (e.g., 1280x720)
│       └── Split test into smaller segments
│
├── Gemini CLI did not respond within 120 seconds?
│   └── YES → timeout
│       ├── Retry once with same parameters
│       ├── If retry also times out: check network connectivity
│       └── Consider reducing video file size
│
├── Gemini returned output but it's not valid JSON?
│   └── YES → parse-error
│       ├── Return warning verdict with raw response included
│       ├── Do NOT retry with same prompt
│       └── Developer reviews raw output manually
│
└── All checks pass but verdict seems wrong?
    ├── Review the analysis prompt — is test context accurate?
    ├── Verify the video shows the expected UI interactions
    └── Consider adjusting verification focus categories
```
