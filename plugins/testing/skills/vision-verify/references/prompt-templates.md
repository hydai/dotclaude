# Vision Verify — Prompt Templates Reference

Detailed prompt engineering guidance for vision-verify. Load this file when constructing analysis prompts for specific test types.

---

## Full Analysis Prompt Template

Copy-paste this template and replace the placeholders before piping to the gemini CLI.

```
You are reviewing a screen recording of a web application test.

## Test Description
{TEST_DESCRIPTION}

## Expected Behavior
{EXPECTED_BEHAVIOR}

## Verification Checklist

### Layout Correctness
- Elements are positioned as expected and remain stable throughout the recording
- No unintended element overlap at any point in the flow
- Spacing and alignment are consistent (margins, paddings, gaps)
- Responsive behavior is correct at the recorded viewport size
- Scroll behavior is smooth and content does not jump or reflow unexpectedly

### Visual Regression
- Colors match the expected palette — no washed-out, inverted, or incorrect tones
- Fonts render with correct weight, size, and family throughout
- Icons and images load and display without distortion, placeholder boxes, or broken assets
- No missing UI elements (buttons, labels, form fields) that should be visible
- Styling is consistent — no unstyled or partially-styled components

### UI Behavior
- Animations play smoothly without stuttering, freezing, or skipping frames
- Transitions (page changes, modal open/close, accordion expand) complete fully
- Interactive states respond visually: buttons show active/pressed state, inputs show focus ring
- Hover and focus states are visually distinct where applicable
- Loading states (spinners, skeletons, progress indicators) appear and resolve correctly

### Accessibility Surface
- Text has sufficient contrast against its background — no low-contrast text combinations
- Font sizes are readable — no text that appears too small to read comfortably
- Focus indicators are visible when keyboard navigation occurs during the recording
- Interactive touch targets (buttons, links, form fields) appear large enough to tap accurately

## Verification Focus
{VERIFICATION_FOCUS}

## Output Format
Respond with valid JSON only — no markdown fences, no prose before or after the JSON object:
{
  "verdict": "pass|fail|warning",
  "summary": "one-sentence summary of the overall visual quality",
  "details": [
    {
      "category": "layout|visual-regression|behavior|accessibility",
      "description": "specific description of the finding",
      "severity": "critical|major|minor",
      "timestamp": "0:05-0:08"
    }
  ],
  "suggestions": ["actionable next steps for the developer or test author"]
}

Verdict definitions:
- "pass" — no visual issues detected; the UI looks correct
- "fail" — one or more issues that likely indicate real bugs requiring action
- "warning" — potential issues that may need human review; or the recording quality is insufficient for confident analysis

Analyze this test recording: @{VIDEO_PATH}
```

**Placeholders:**

| Placeholder | Required | Description |
|:------------|:---------|:------------|
| `{TEST_DESCRIPTION}` | Yes | What the test does — the user journey being exercised (e.g., "User fills out a 3-step registration form and submits") |
| `{EXPECTED_BEHAVIOR}` | Yes | What correct visual behavior looks like (e.g., "Each step slides in from the right, progress indicator advances, success banner appears after submission") |
| `{VERIFICATION_FOCUS}` | No | A short list of categories to emphasize. Omit this section or write "All categories" if no special focus is needed |
| `{VIDEO_PATH}` | Yes | Absolute or relative path to the WebM video file (e.g., `test-results/checkout-flow/video.webm`) |

---

## Verification Checklist by Category

Use these checklist items when building custom prompts or when reviewing Gemini's findings manually.

### Layout Correctness

- Elements positioned as expected — no elements displaced to wrong quadrant or off-screen
- No unintended overlap — elements that should be visually separate do not overlap at any frame
- Proper spacing — consistent margins, padding, and gaps throughout the flow
- Responsive behavior — layout adapts correctly at the recorded viewport; no horizontal overflow or collapsed columns that should be side-by-side
- Scroll behavior — page scrolls smoothly; sticky headers/footers stay in place; no content reflow after scroll

### Visual Regression

- Colors render correctly — brand colors, semantic colors (error red, success green), and neutral backgrounds match expectations
- Fonts render correctly — correct typeface, weight, and size; no fallback system fonts substituted unexpectedly
- Icons display correctly — vector icons are sharp and sized correctly; raster icons are not blurry or pixelated
- Images load fully — no broken image placeholders, partial loads, or distorted aspect ratios
- No missing UI elements — all buttons, labels, form fields, navigation items, and decorative elements visible as expected
- Consistent styling — no components that appear unstyled, partially styled, or using a different theme

### UI Behavior

- Animations play smoothly — no stuttering, frame skipping, or janky motion during transitions
- Transitions complete fully — modal open/close, page transitions, accordion expand/collapse finish without clipping
- Interactive states are visible — pressed/active states on buttons, selected state on checkboxes and radios
- Hover and focus states appear where expected — underlines on links, outlines on focused inputs
- Loading states resolve — spinners, skeleton screens, and progress bars appear and then disappear when content loads
- Error and success states display — validation errors appear inline, success confirmations are shown

### Accessibility Surface

- Text contrast — body text, headings, placeholder text, and labels are legible against their backgrounds
- Readable text sizes — no text so small it would be difficult to read without zooming
- Visible focus indicators — when the recording captures keyboard navigation, focus rings or outlines are visible
- Adequate touch targets — buttons, links, and form controls appear large enough to tap on a touch screen (minimum ~44px visual size)

---

## Output Format Instructions

The prompt must request JSON output matching the F3 schema. Include this exact output format section in every analysis prompt to ensure parseable responses.

**Schema:**

```json
{
  "verdict": "pass|fail|warning",
  "summary": "one-sentence summary",
  "details": [
    {
      "category": "layout|visual-regression|behavior|accessibility",
      "description": "description of the finding",
      "severity": "critical|major|minor",
      "timestamp": "0:05-0:08"
    }
  ],
  "suggestions": ["actionable next steps"]
}
```

**Field guidance for Gemini:**

- `verdict` — A single string: `"pass"`, `"fail"`, or `"warning"`. Use `"warning"` when issues are ambiguous or when recording quality prevents confident assessment.
- `summary` — One sentence. State the verdict reason clearly: "The form submission flow renders correctly with no visual issues detected." or "The modal close animation clips the header element on viewport widths below 768px."
- `details` — One entry per distinct finding. Leave as an empty array `[]` on a clean pass.
  - `category` — Must be one of the four defined values: `"layout"`, `"visual-regression"`, `"behavior"`, `"accessibility"`
  - `description` — Describe the specific issue with enough detail to locate it in the UI: "The 'Submit' button overlaps the bottom navigation bar during the confirmation step" not "button issue found"
  - `severity` — `"critical"` for issues that would block users or constitute clear defects; `"major"` for significant visual bugs that degrade experience; `"minor"` for cosmetic imperfections
  - `timestamp` — The time range in the video where the issue is visible, formatted as `"M:SS-M:SS"` (e.g., `"0:12-0:15"`). Use `"0:00-end"` if the issue is present throughout.
- `suggestions` — Concrete, actionable items. Prefer test assertions, bug report descriptions, or developer investigation steps over vague advice.

**Parse error handling:** If Gemini's response is not valid JSON, vision-verify returns a `"warning"` verdict and includes the raw text for manual review. To minimize parse errors, the prompt instructs Gemini to return JSON only with no surrounding markdown or prose.

---

## Example Prompts

### Example 1 — Multi-Step Form Submission Flow

```bash
echo 'You are reviewing a screen recording of a web application test.

## Test Description
User completes a 3-step registration form: personal details (step 1), account settings (step 2), and plan selection (step 3), then submits. Each step transition is triggered by clicking "Next". After submission, a success confirmation page is shown.

## Expected Behavior
Each step slides in from the right as the user advances. The progress indicator at the top of the form updates to show the current step highlighted. Input fields show inline validation errors when left empty and submitted. On the final confirmation page, a green success banner appears at the top, the registered email address is displayed, and a "Go to Dashboard" button is visible.

## Verification Checklist

### Layout Correctness
- Elements are positioned as expected and remain stable throughout the recording
- No unintended element overlap at any point in the flow
- Spacing and alignment are consistent (margins, paddings, gaps)
- Responsive behavior is correct at the recorded viewport size
- Scroll behavior is smooth and content does not jump or reflow unexpectedly

### Visual Regression
- Colors match the expected palette — no washed-out, inverted, or incorrect tones
- Fonts render with correct weight, size, and family throughout
- Icons and images load and display without distortion, placeholder boxes, or broken assets
- No missing UI elements (buttons, labels, form fields) that should be visible
- Styling is consistent — no unstyled or partially-styled components

### UI Behavior
- Animations play smoothly without stuttering, freezing, or skipping frames
- Transitions (page changes, modal open/close, accordion expand) complete fully
- Interactive states respond visually: buttons show active/pressed state, inputs show focus ring
- Hover and focus states are visually distinct where applicable
- Loading states (spinners, skeletons, progress indicators) appear and resolve correctly

### Accessibility Surface
- Text has sufficient contrast against its background — no low-contrast text combinations
- Font sizes are readable — no text that appears too small to read comfortably
- Focus indicators are visible when keyboard navigation occurs during the recording
- Interactive touch targets (buttons, links, form fields) appear large enough to tap accurately

## Verification Focus
Emphasize layout correctness (step transition layout, form field positioning) and accessibility (input labels, focus indicators on form controls, contrast on validation error messages).

## Output Format
Respond with valid JSON only — no markdown fences, no prose before or after the JSON object:
{
  "verdict": "pass|fail|warning",
  "summary": "one-sentence summary of the overall visual quality",
  "details": [
    {
      "category": "layout|visual-regression|behavior|accessibility",
      "description": "specific description of the finding",
      "severity": "critical|major|minor",
      "timestamp": "0:05-0:08"
    }
  ],
  "suggestions": ["actionable next steps for the developer or test author"]
}

Analyze this test recording: @test-results/registration-flow/video.webm' | gemini -m gemini-3-pro-preview -y
```

---

### Example 2 — Drag-and-Drop Reordering Interaction

```bash
echo 'You are reviewing a screen recording of a web application test.

## Test Description
User reorders items in a task list using drag-and-drop. The test drags the third item to the first position, then drags the last item to the second position. Each drag operation involves pressing the drag handle, moving the item to the target position, and releasing.

## Expected Behavior
When a drag begins, the dragged item lifts visually (slight shadow or opacity change) and the remaining items reflow to show a drop target indicator (highlighted gap or placeholder). While dragging, items smoothly animate out of the way. On drop, the item snaps into its new position and the lift effect is removed. The final list order reflects both reordering operations. No items disappear, duplicate, or revert to original position after drop.

## Verification Checklist

### Layout Correctness
- Elements are positioned as expected and remain stable throughout the recording
- No unintended element overlap at any point in the flow
- Spacing and alignment are consistent (margins, paddings, gaps)
- Responsive behavior is correct at the recorded viewport size
- Scroll behavior is smooth and content does not jump or reflow unexpectedly

### Visual Regression
- Colors match the expected palette — no washed-out, inverted, or incorrect tones
- Fonts render with correct weight, size, and family throughout
- Icons and images load and display without distortion, placeholder boxes, or broken assets
- No missing UI elements (buttons, labels, form fields) that should be visible
- Styling is consistent — no unstyled or partially-styled components

### UI Behavior
- Animations play smoothly without stuttering, freezing, or skipping frames
- Transitions (page changes, modal open/close, accordion expand) complete fully
- Interactive states respond visually: buttons show active/pressed state, inputs show focus ring
- Hover and focus states are visually distinct where applicable
- Loading states (spinners, skeletons, progress indicators) appear and resolve correctly

### Accessibility Surface
- Text has sufficient contrast against its background — no low-contrast text combinations
- Font sizes are readable — no text that appears too small to read comfortably
- Focus indicators are visible when keyboard navigation occurs during the recording
- Interactive touch targets (buttons, links, form fields) appear large enough to tap accurately

## Verification Focus
Emphasize UI behavior: drag lift animation, drop target indicator visibility, reflow smoothness, and snap-to-position on drop. Also check that item text and drag handle icons remain legible during the drag motion.

## Output Format
Respond with valid JSON only — no markdown fences, no prose before or after the JSON object:
{
  "verdict": "pass|fail|warning",
  "summary": "one-sentence summary of the overall visual quality",
  "details": [
    {
      "category": "layout|visual-regression|behavior|accessibility",
      "description": "specific description of the finding",
      "severity": "critical|major|minor",
      "timestamp": "0:05-0:08"
    }
  ],
  "suggestions": ["actionable next steps for the developer or test author"]
}

Analyze this test recording: @test-results/drag-reorder/video.webm' | gemini -m gemini-3-pro-preview -y
```

---

## Prompt Customization Guidance

Adjust the `{VERIFICATION_FOCUS}` section and reorder or weight checklist items based on the test type.

### Form Flows

**Emphasize:** Layout correctness and accessibility.

Add to the Verification Focus section:
```
Focus on:
- Layout: form field positioning, label alignment, error message placement, button layout
- Accessibility: input labels are visible and associated, validation error messages have sufficient contrast,
  focus moves predictably between fields, required field indicators are visible
```

Rationale: Form flows often have layout regressions from CSS changes to input grids. Accessibility issues in forms (missing labels, low-contrast errors) are high-impact because they affect users directly.

### Animations and Transitions

**Emphasize:** UI behavior and timing.

Add to the Verification Focus section:
```
Focus on:
- Behavior: animation smoothness (no stuttering or frame drops), transition completeness (no clipped or
  interrupted animations), timing (transitions complete within the expected duration), state consistency
  (element is in the correct visual state before and after the animation)
```

Rationale: Functional tests can verify that a transition occurred but cannot confirm it played smoothly. Vision model analysis is particularly useful for detecting janky motion or animations that clip or freeze mid-play.

### Responsive Layouts

**Emphasize:** Layout correctness at multiple viewport sizes.

Add to the Verification Focus section:
```
Focus on:
- Layout: horizontal overflow (nothing extends beyond the viewport edge), column/grid behavior at this
  viewport width, navigation collapse (hamburger menu vs. full nav), image and media scaling,
  touch target sizes appropriate for mobile viewports
```

When testing responsive behavior, run separate test recordings at each target breakpoint (e.g., 375px mobile, 768px tablet, 1280px desktop) rather than one recording that resizes. Submit each to vision-verify separately with the viewport size noted in the Test Description.

### Data-Heavy UIs (Tables, Charts, Dashboards)

**Emphasize:** Visual regression and readability.

Add to the Verification Focus section:
```
Focus on:
- Visual regression: table borders and row striping render correctly, chart colors are distinct and
  legible (no color confusion between data series), axis labels and legends are visible, data cells
  are not truncated without ellipsis indicators
- Layout: column widths are stable (no layout shift when data loads), sticky header behavior works
  correctly as the user scrolls through long tables
```

Rationale: Data-heavy UIs are prone to subtle rendering issues — chart colors blending together, table cells overflowing, axis labels being cut off — that functional tests do not check.

---

## CLI Invocation Pattern

All prompts must use the stdin pipe pattern with the `@`-prefixed file reference. The `-p` flag does not work correctly with Gemini CLI's vision processing.

**Correct pattern:**

```bash
echo "<prompt_text>: @<video_path>" | gemini -m gemini-3-pro-preview -y
```

For multi-line prompts (recommended for readability), use single quotes around the entire prompt body:

```bash
echo '<full multi-line prompt ending with: @path/to/video.webm>' | gemini -m gemini-3-pro-preview -y
```

**Incorrect — do not use:**

```bash
gemini -m gemini-3-pro-preview -p "<prompt>" @<video_path>  # WRONG: -p flag does not work with video
```

**Flags reference:**

| Flag | Required | Purpose |
|:-----|:---------|:--------|
| `-m gemini-3-pro-preview` | Yes | Specifies the vision-capable model. Override if the model is renamed or unavailable. |
| `-y` | Yes | Auto-confirms the upload prompt — required for non-interactive use in test scripts |

**Model override:** Replace `gemini-3-pro-preview` with an alternative vision-capable Gemini model name if the default model is deprecated or unavailable. The invocation pattern and prompt structure remain the same.
