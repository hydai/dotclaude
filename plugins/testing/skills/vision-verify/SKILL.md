---
name: vision-verify
description: >
  Use when writing or modifying Playwright tests that involve UI interactions,
  visual behavior verification, animations, layout testing, or when the developer
  explicitly requests visual verification. Activates on mentions of visual testing,
  vision model verification, video-based UI analysis, Playwright visual checks,
  or visual regression detection during test authoring.
---

# Vision Verify — Playwright Visual Verification with Gemini

## Role

You are a visual verification engineer. Your goal is to catch visual bugs that functional Playwright tests miss — broken layouts, overlapping elements, animation glitches, accessibility surface issues — by recording browser sessions and analyzing them with Gemini's vision model.

## Workflow

### Step 1 — Assess Test as Visual Verification Candidate

Determine whether the test involves UI that benefits from visual verification. Not every test is a candidate; skip pure API or data-layer tests.

Indicators that a test is a good candidate:
- Animations or CSS transitions that must play smoothly
- Multi-step visual flows (form wizard, onboarding, checkout)
- Layout-sensitive interactions (responsive breakpoints, grid reflow)
- Drag-and-drop or pointer interactions where position matters
- Developer explicitly requests visual verification

If none of these apply, skip vision-verify and continue with functional testing.

### Step 2 — Configure Video Recording

Provide this Playwright configuration to enable video recording. Apply it per-test-run; do not modify the project's persistent Playwright config.

```typescript
// Per-test-run configuration only — do not commit to playwright.config.ts
use: {
  video: {
    mode: 'on',
    size: { width: 1280, height: 720 } // Match test viewport
  }
}
```

- Format: WebM (Playwright default)
- Resolution: Match the test's viewport size exactly
- Output directory: `test-results/` (Playwright default)
- If the project already has video recording configured, use existing settings

### Step 3 — Run Test and Capture Video

Execute the test with video recording enabled. After the run, verify the video file exists at the expected path under `test-results/`. If no video file is produced, see `references/error-handling.md` for recording failure remediation.

Check the file size before proceeding — the default limit is 20MB. If exceeded, reduce test duration or viewport size.

### Step 4 — Submit Video to Gemini CLI

Construct and execute the analysis command using the stdin pipe pattern:

```bash
echo 'You are reviewing a screen recording of a web application test.

## Test Description
<what the test does>

## Expected Behavior
<what correct visual behavior looks like>

## Verification Checklist
- Layout: Elements positioned correctly, no overlapping, proper spacing
- Visual: Colors, fonts, icons render correctly, no broken assets
- Behavior: Animations smooth, transitions complete, interactions respond
- Accessibility: Sufficient contrast, readable text, visible focus indicators

## Output Format
Respond with JSON:
{
  "verdict": "pass|fail|warning",
  "summary": "one-sentence summary",
  "details": [{"category": "layout|visual-regression|behavior|accessibility", "description": "...", "severity": "critical|major|minor", "timestamp": "0:05-0:08"}],
  "suggestions": ["actionable next steps"]
}

Analyze this test recording: @<video_path>' | gemini -m gemini-3-pro-preview -y
```

**CRITICAL:** Always pipe the prompt to stdin with the `@`-prefixed file reference. Do NOT use the `-p` flag — it does not work correctly with Gemini CLI's vision processing. The `-y` flag is required to auto-confirm.

- Default model: `gemini-3-pro-preview` — override if the model is renamed or unavailable
- Timeout: 120 seconds (retry once on timeout before returning an error)
- Max file size: 20MB before submission

Load `references/prompt-templates.md` when you need customized prompt templates for specific test types (animations, accessibility, responsive layout).

### Step 5 — Parse Structured Result

Parse Gemini's JSON response into the result schema:

| Field | Type | Description |
|:------|:-----|:------------|
| `verdict` | `pass`, `fail`, `warning` | Overall assessment |
| `summary` | string | One-sentence finding |
| `details` | list | Each entry: `category`, `description`, `severity`, `timestamp` |
| `suggestions` | list of strings | Actionable next steps |

If the response does not parse as valid JSON, return a `warning` verdict and include the raw response for manual review. Do not retry with the same prompt on a parse error.

### Step 6 — Act on Verdict

| Verdict | Action |
|:--------|:-------|
| `pass` | Note visual verification passed in test comments or output |
| `warning` | Surface finding to developer with suggestions; include raw response if unparseable |
| `fail` | Investigate the finding — consider adjusting the test, adding assertions, or reporting the visual bug |

Visual verification is advisory only. Never block or fail the test suite based on vision-verify results.

## Quick Reference

### CLI Invocation Pattern

```bash
echo '<prompt with @video_path at end>' | gemini -m gemini-3-pro-preview -y
```

Always use stdin pipe with single quotes. Never use `-p`. Always include `-y`.

### Result Schema

```json
{
  "verdict": "pass | fail | warning",
  "summary": "one-sentence summary",
  "details": [
    {
      "category": "layout | visual-regression | behavior | accessibility",
      "description": "description of the finding",
      "severity": "critical | major | minor",
      "timestamp": "0:05-0:08"
    }
  ],
  "suggestions": ["actionable next step"]
}
```

### Error Types

| Error Type | Meaning |
|:-----------|:--------|
| `recording-failed` | No video file produced after test run |
| `cli-missing` | `gemini` command not found in PATH |
| `model-unavailable` | Model not found or quota exceeded |
| `file-too-large` | Video exceeds 20MB limit |
| `timeout` | Gemini CLI did not respond within 120 seconds |
| `parse-error` | Response is not valid JSON |
| `empty-video` | Video file is zero bytes or zero duration |

### Input Contract Summary

| Input | Required | Default |
|:------|:---------|:--------|
| Video file path (WebM) | Yes | — |
| Test description | Yes | — |
| Expected behavior | Yes | — |
| Verification focus (categories) | No | All categories |
| Model override | No | `gemini-3-pro-preview` |
| Max file size (MB) | No | 20 |
| Timeout (seconds) | No | 120 |

## Anti-Patterns

- **Using `-p` flag with video input** — Does not work with Gemini CLI's vision processing. Always pipe via stdin with `@`-prefixed file path.
- **Running vision-verify on every test** — Only apply to visual verification candidates. API tests, unit tests, and pure data-layer tests gain nothing from visual analysis.
- **Modifying the project's Playwright config permanently** — Video recording is a per-test-run setting. Never commit recording config to `playwright.config.ts`.
- **Blocking test suite on vision-verify results** — Visual verification is advisory. A `fail` verdict means investigate, not halt CI.
- **Retrying on `model-unavailable`** — This is a persistent issue (API access, quota, model deprecation). Surface the error immediately; retrying will not resolve it.

## Reference Guide

Load these files for detailed specifications:

| File | Contains | Load When |
|:-----|:---------|:----------|
| `references/prompt-templates.md` | Full analysis prompt templates, verification checklists, customization guidance | Constructing analysis prompts for specific test types |
| `references/error-handling.md` | All error scenarios, detection methods, remediation steps, retry patterns | Handling failures during visual verification |
