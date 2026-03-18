---
name: design-reference
description: Use when a Jira ticket has Figma links, design screenshots, or UI mockups — opens designs via browser, captures screenshots, maintains design context during implementation, and compares implementation against design during review
user-invocable: false
---

# Design Reference

## Overview

Open, capture, and maintain design context from Figma links and image attachments found in Jira tickets. Provides visual reference during implementation and enables comparison during review.

## When This Skill Activates

- Called from `android:jira-fetching` when design references are detected
- Called manually when user provides a Figma link or design screenshot
- Called during review to compare implementation vs design

## Step 1: Collect Design Sources

From the Jira ticket data, gather all design references:

**Figma links** (from description or remote links):
```
[
  { "type": "figma", "url": "https://www.figma.com/...", "title": "Login Screen Design" },
  ...
]
```

**Image attachments** (from `fields.attachment[]`):
```
[
  { "type": "attachment", "filename": "screenshot.png", "url": "[content URL]", "thumbnail": "[thumb URL]", "mimeType": "image/png" },
  ...
]
```

**User-provided links** (from conversation):
```
[
  { "type": "user-provided", "url": "https://www.figma.com/...", "title": "User provided" },
  ...
]
```

## Step 2: Open Figma Designs

For each Figma link, use the Playwright MCP to capture the design:

### Navigate to Figma
```
mcp__plugin_playwright_playwright__browser_navigate → url: [figma URL]
```

**Wait for load:**
```
mcp__plugin_playwright_playwright__browser_wait_for → state: "networkidle" or timeout 15s
```

### Handle Figma Auth

Figma may require login. If the page shows a login prompt:
> Figma requires authentication. Options:
> - Log in via browser (I'll open it for you to authenticate)
> - Use a public/shared link instead
> - Skip Figma capture — I'll work from the description only

If the Figma file is publicly accessible or user is already logged in, proceed.

### Capture Screenshots

Take a screenshot of the Figma design:
```
mcp__plugin_playwright_playwright__browser_take_screenshot
```

**For multi-page Figma files:**
1. Take a snapshot to identify the page structure:
   ```
   mcp__plugin_playwright_playwright__browser_snapshot
   ```
2. Look for frame/page names relevant to the ticket (match keywords from Jira summary)
3. Navigate to each relevant frame
4. Capture each frame as a separate screenshot

**For single-screen designs:**
- One screenshot is enough
- Zoom to fit if the design is larger than viewport

### Resize for Better Capture
```
mcp__plugin_playwright_playwright__browser_resize → width: 1440, height: 900
```
Use a standard desktop viewport for consistent captures.

## Step 3: View Image Attachments

For Jira image attachments, download and display them:

**Read the image via its content URL:**
The attachment `content` field contains a direct URL to the file. Use:
```
mcp__plugin_playwright_playwright__browser_navigate → url: [attachment content URL]
mcp__plugin_playwright_playwright__browser_take_screenshot
```

Or if the image is directly viewable, use the Read tool on the downloaded file.

**For multiple attachments:**
- Capture each one
- Name them clearly: "Attachment 1: [filename]", "Attachment 2: [filename]"

## Step 4: Present Design Context

After capturing all designs, present a summary:

> **Design References Captured:**
>
> **Figma Designs:**
> 1. [Page/Frame name] — [screenshot captured]
>    - Key elements: [header, form fields, buttons, etc.]
>    - Colors: [primary colors observed]
>    - Layout: [grid, stack, sidebar, etc.]
>
> **Attachments:**
> 1. [filename] — [screenshot captured]
>    - Shows: [what the screenshot depicts]
>
> **Design Notes:**
> - [Any text annotations visible in the design]
> - [Spacing/sizing patterns observed]
> - [Interactive states shown (hover, active, disabled)]

## Step 5: Maintain Context During Implementation

During implementation, the design context should be available for reference. Key information to retain:

**Visual inventory:**
- Component list (buttons, inputs, cards, etc.)
- Color values (if visible in Figma)
- Typography (font sizes, weights)
- Spacing patterns
- Layout structure (flex, grid, stack)

**Interaction patterns:**
- Navigation flows
- State changes (loading, error, empty, success)
- Animations/transitions mentioned

**When the user asks about design details:**
- Reference the captured screenshots
- If the detail isn't visible, suggest re-opening Figma for a closer look

## Step 6: Compare Implementation vs Design (Review Mode)

When called during review/finish-task:

### Capture Implementation Screenshot

If the implementation is running locally or can be previewed:
```
mcp__plugin_playwright_playwright__browser_navigate → url: [local dev URL, e.g., localhost:3000]
mcp__plugin_playwright_playwright__browser_take_screenshot
```

For mobile apps (Android/iOS), this may not be possible via browser. In that case:
> To compare with the design, please take a screenshot of the running app and share it, or provide the emulator screen.

### Side-by-Side Comparison

Compare the captured design screenshot with the implementation screenshot:

> **Design vs Implementation Comparison:**
>
> **Matches:**
> - [Layout structure matches]
> - [Color scheme matches]
> - [Typography matches]
>
> **Differences:**
> - [Spacing: design shows 16px gap, implementation appears different]
> - [Color: button color differs]
> - [Missing: error state not implemented]
> - [Extra: implementation has element not in design]
>
> **Recommendations:**
> - [Specific fixes needed]

## Handling Edge Cases

**Figma link is expired or moved:**
> The Figma link appears to be invalid or the file has moved.
> - Paste an updated Figma link
> - Continue without design reference
> - Check with the designer

**Figma file is too complex (many pages/frames):**
> This Figma file has [N] pages. Which ones are relevant?
> [List page names if visible]

Let the user select specific pages to capture.

**No browser available (Playwright MCP not connected):**
> Cannot open Figma — Playwright browser is not available.
> - Describe the design from the Jira description
> - Work from the attachment thumbnails
> - Skip design reference

Fall back gracefully — design reference is helpful but not blocking.

**Attachments are not images:**
- PDF → note as "PDF attachment — open manually"
- Sketch/XD files → note as "Sketch file — open in Sketch"
- Only process image types (png, jpg, gif, svg, webp)

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Blocking on Figma auth | Fall back to description/attachments if auth fails |
| Ignoring mobile viewport | Resize browser to match target device for mobile designs |
| Capturing too many pages | Focus on pages relevant to the ticket's scope |
| Not re-checking design during review | Always compare implementation vs design before PR |
| Assuming design is complete | Designs may be WIP — ask user if unsure |
| Skipping dark mode designs | Check if Figma has both light and dark variants |
