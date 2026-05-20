---
name: meeting-scribe
description: Create professional meeting minutes documents by adapting to any organization's Word template — works with formal memos, bulleted notes, action-item tables, agenda-driven layouts, decision logs, or custom formats. Use this skill whenever the user mentions meeting minutes, meeting notes, kickoff call notes, project meeting documentation, action items from a call, or wants to document what was discussed in a meeting. The skill requires the user to provide an existing meeting minutes template (.docx) which it inspects to learn the structure, then preserves all formatting, branding, headers, logos, and styles while inserting new meeting content. Trigger this skill whenever a meeting transcript needs to be turned into structured documentation, when raw notes need to be formalized, when action items need to be captured in a structured way, or any time someone wants polished meeting documentation that matches their team's existing format.
---

# Meeting Scribe

Creates professional meeting minutes by adapting to the user's existing Word template. Preserves all formatting, branding, headers, logos, and styles. Works across any organization's format — formal memos, simple bulleted notes, action-tracking tables, agenda-driven layouts, decision logs, and custom layouts.

This skill is **template-aware**: it inspects the user's template to learn the structure, then maps meeting content into the correct sections.

## Critical Requirement

The user MUST provide an existing meeting minutes template (.docx file). Acceptable templates include:
- A blank template the user's organization uses
- A previous meeting's minutes (any project) to reference structurally
- A custom layout the user has designed

Without a template, ask the user to provide one before proceeding. Do not generate meeting minutes from scratch — the value of this skill is preserving the user's existing format and branding.

## When to Use This Skill

- User asks to create meeting minutes, meeting notes, or kickoff notes
- User provides a meeting transcript and wants it documented
- User pastes raw notes from a meeting and wants them formalized
- User mentions documenting action items from a call
- User wants to record decisions or discussion points in structured form
- User asks for meeting documentation that matches their team's existing format

## Required Inputs

### 1. Template Document (Required)
A .docx file representing the desired output format. Any format works — the skill adapts.

### 2. Meeting Content (Required)
One of:
- **Transcript** — full or partial audio transcript
- **Typed Notes** — bullet points or paragraph notes from the user
- **Discussion Summary** — high-level points the user types in chat

### 3. Meeting Metadata (Recommended)
- Date and time of the meeting
- Attendees (names, roles, organizations)
- Meeting purpose or project context

If any required metadata isn't provided, ask once after inspecting the template — don't ask before reading the template, because the template may already contain some of this info (e.g., recurring attendees).

## Workflow

### Step 1: Inspect the Template

Unpack the .docx (it's a ZIP archive) and read its structure:

```bash
mkdir -p /home/claude/template-work
cd /home/claude/template-work
unzip -o "/path/to/template.docx"
```

Read `word/document.xml` and identify:

- **Document header** — e.g., "MEMORANDUM", "MEETING MINUTES", "MEETING NOTES", or a company-specific header
- **Metadata fields** — e.g., To/From/Date/Re, or Date/Project/Attendees, or custom labels
- **Section markers** — look for ALL CAPS section names, bold paragraph styles, or distinctive lead-in phrases ("Attendees included:", "The following was discussed:", "Action Items:")
- **List structures** — bullet styles, numbered lists, or table-based action items
- **Tables** — many templates use tables for action items (with Owner / Due Date columns)
- **Closing conventions** — sign-off lines, footer text, signature blocks

### Step 2: Model the Template Structure

After inspection, internally outline the template:

```
TEMPLATE STRUCTURE
- Header: [detected header text and style]
- Metadata Fields: [list, in order]
- Body Sections: [list, in order]
- Action Items Format: [bulleted list | numbered list | table]
- Closing: [detected closing convention]
- Branding: [logo present yes/no, header/footer customization yes/no]
```

If anything is ambiguous, ask the user once to clarify — but only the genuinely unclear parts. Don't interrogate.

### Step 3: Gather Meeting Content

From the user-provided transcript or notes, extract:
- Attendees with their roles and organizations
- Discussion topics (themes, decisions, key points — NOT verbatim)
- Action items with owners and due dates (where mentioned)
- Open questions or follow-ups

If the transcript lacks context the user hasn't provided (project name, meeting purpose), ask once before proceeding.

### Step 4: Confirm the Mapping

Before editing the template, briefly confirm:

> "Your template has sections for [Attendees / Discussion / Action Items / etc.]. I'll fill those in with the content from the transcript. Anything you want me to add or restructure?"

This catches mismatches before the edit. Keep it short — one or two sentences.

### Step 5: Edit the Template XML

Use `str_replace` on `word/document.xml` to update content while preserving all formatting.

**Critical XML preservation rules:**
- Only replace text inside `<w:t>` tags
- Keep ALL `<w:rPr>` (run properties — fonts, bold, italic, color) intact
- Keep ALL `<w:pPr>` (paragraph properties — alignment, spacing, paragraph style) intact
- Preserve `xml:space="preserve"` attributes on whitespace nodes
- **Preserve trailing whitespace in field labels.** Templates often pad labels like `"To:           "` (with 11 spaces) for column alignment. When you replace the field value, do NOT modify the label's whitespace — replace only the value text that follows. Failing to preserve this padding causes the replacement text to sit flush against the colon and visibly misalign the document.
- For new bullet points, copy the XML structure of an existing bullet, then modify only the text content
- For new table rows, copy an existing row's XML and modify only the cell text within `<w:t>` tags

**Handling multi-occurrence placeholders (CRITICAL):**

When the same placeholder text appears in multiple places — most commonly in table rows where every body row contains `[Action description]` or similar — a naive `str_replace` on the placeholder text alone will only replace the *first* occurrence and leave the others untouched.

The correct approach depends on what you're doing:

- **Replacing identical rows with different content (most common case):** Match the full `<w:tr>...</w:tr>` XML block for each row and replace each one explicitly. Or: concatenate all identical placeholder rows into one combined string and replace that combined string with all your new rows concatenated together. The latter is cleaner when expanding a 3-row template into 6 actual rows.

- **Replacing identical bullets:** Same principle. Match the full `<w:p>...</w:p>` block for each bullet, not just the inner text. If you have three `[Discussion point N]` bullets and need five real points, replace the third placeholder bullet with three concatenated new bullets (so the count grows from 3 to 5).

- **One-by-one replacement using count parameter:** Some `str_replace` implementations support a count argument (e.g., `xml.replace(old, new, 1)` in Python). Use this when each occurrence needs distinct content and the surrounding XML is identical.

Never assume `str_replace` is global by default — verify the behavior, especially on table-based templates.

**Distinguishing placeholders from intentional default content:**

Not every text string in a template is a placeholder. Some templates ship with intentional default values that should stay as-is unless the user provides overrides. Common examples:

- Status columns with default value `"Open"` (intended; keep unless action is actually complete)
- "TBD" or "Pending" entries (sometimes placeholders, sometimes intentional)
- Closing boilerplate like "Please review and notify the author of any corrections..." (intentional; never replace)
- Sign-off lines, footer text, copyright notices (intentional)

A reliable heuristic: text inside `[square brackets]` is almost always a placeholder. Text without brackets is more likely intentional. If unclear, leave the content alone and ask the user.

**Finding sections in the XML:**
Search for distinctive text fragments that mark each section. Examples:
- `"Attendees included:"` → attendees section follows
- `"The following summarizes"` → discussion section follows
- `"Action Items"` or `"ACTION ITEMS"` → action items section follows
- `"Who was there"`, `"What we talked about"`, `"Next steps"` → informal section labels (casual templates)
- `"To:"`, `"From:"`, `"Date:"`, `"Re:"` → metadata field labels
- `"Project:"`, `"Facilitator:"`, `"Notetaker:"` → inline metadata fields (PM-style templates)

If the template uses paragraph styles instead of distinctive text, search for the style reference (e.g., `<w:pStyle w:val="Heading1"/>`) to locate sections.

### Step 6: Repack the Document

```bash
cd /home/claude/template-work
zip -r "/mnt/user-data/outputs/Meeting_Minutes_[Project]_[YYYY-MM-DD].docx" .
```

Use a filename that reflects the meeting context. Replace `[Project]` with the project or topic name (underscores for spaces) and `[YYYY-MM-DD]` with the meeting date.

### Step 7: Present the Document

If the `present_files` tool is available, use it to share the output file. Otherwise, tell the user the output path so they can download.

## Recognizing Common Template Patterns

Most meeting templates fall into one of these patterns. Use the indicators to quickly classify:

### Memo / Formal Letter Format
- Indicators: "MEMORANDUM" header, To/From/Date/Re fields, formal opening paragraph, formal closing
- Examples: law firm summaries, engineering firm project memos, government agency notes
- Section names typically in ALL CAPS

### Action-First Format
- Indicators: Action items prominently placed (often at top), less discussion narrative, table layout for actions
- Examples: agile teams, project management offices, ops standups
- Heavy emphasis on owners and due dates

### Agenda-Driven Format
- Indicators: Numbered agenda items as section headers, discussion summarized under each
- Examples: board meetings, committee meetings, recurring team syncs
- Often includes "Old Business / New Business" subsections

### Decision Log Format
- Indicators: Date column, decision column, owner column, table layout
- Examples: architecture review boards, governance committees
- Minimal narrative — focused on what was decided

### Free-Form Notes
- Indicators: Minimal structure, just bulleted notes under date headers
- Examples: 1-on-1s, brainstorms, informal team syncs
- Treat as flexible — match the existing tone

If the template doesn't clearly match any of these, treat it as a custom format and work from what's actually in the document. Don't force it into a known pattern.

## Handling Edge Cases

### Template uses placeholders like [DATE] or [ATTENDEES]
Replace placeholders directly. Don't add new section structure — the user has already designed it.

### Template uses tables for action items
- Find the action items table by looking for header cells like "Action", "Owner", "Due Date", "Status"
- Identify the body row pattern (every row in the table will have nearly identical XML structure with placeholder text like `[Action description]`)
- See **Step 5 → Handling multi-occurrence placeholders** for the correct way to expand or replace table rows. Each row must be replaced as a complete `<w:tr>...</w:tr>` XML block — not by replacing the inner placeholder text alone, which only hits the first occurrence
- Preserve cell formatting, borders, widths, and shading by copying the existing row's structure exactly and modifying only `<w:t>` text content
- Status/state columns with default values like "Open", "Pending", or "TBD" are usually intentional defaults — keep them unless the user indicates the action is already complete

### Template has logos or images
The unzip/zip approach preserves them automatically. Do NOT modify the `word/media/` folder.

### Template has multiple meetings in one document (running log)
Add a new entry at the end, copying the structure of the most recent entry. Don't restructure the prior entries.

### Multiple participants from the same organization
Group them under one organization line:
`• [Organization] – [Name 1], [Title]; [Name 2], [Title]`

### Transcript is very long (>10 pages)
Summarize aggressively. Capture decisions, not every utterance. The output should be readable in under 3 minutes — typically 60-80% reduction from transcript length.

### Action items lack owners or dates
Flag this to the user at the end:
> "These action items don't have owners assigned — want me to suggest based on who discussed each one, or leave blank for you to fill in?"

### Template is unusual or skill cannot confidently identify sections
Show the user what was detected and ask for confirmation before editing:
> "I read the template but I'm not 100% sure where the Discussion section ends and Action Items begins. Can you point me at a section header or specific text I should use as the boundary?"

## Quality Standards

- **Concise** — summarize, don't transcribe. Target 60-80% reduction from transcript length.
- **Decision-oriented** — capture what was decided, not just what was discussed.
- **Action items must be actionable** — "Coordinate with [agency] to schedule review by [date]" not "agency stuff".
- **Match template tone** — formal language for formal templates, casual for casual.
- **Attributable when relevant** — when a specific person made a key statement or commitment, name them.
- **Preserve voice** — if the template's existing entries use first person plural ("we discussed"), match that. If third person ("the team reviewed"), match that.

## Example Interaction

**User**: I need meeting minutes from yesterday's design review. Here's our team's template [attaches MeetingNotes_2026Q2.docx] and here's the transcript [pastes transcript].

**Claude**:
1. [Unpacks template, reads word/document.xml]
2. "Got it. Your template uses an agenda-driven format with sections for Attendees, Agenda Items (numbered), Discussion, Decisions, and Action Items in a table. I'll map the transcript into that structure."
3. [Asks any genuinely necessary clarifying questions, e.g., "I see 6 attendees in the transcript but only 4 listed in last week's minutes — should I add the new ones?"]
4. [Edits the XML, repacks the .docx, saves to outputs]
5. "Done — here's the file. The action items table has 5 items; the last one didn't have an owner mentioned, so I left that cell blank for you to fill in."

## Filename Convention

Default filename format: `Meeting_Minutes_[Project]_[YYYY-MM-DD].docx`

Examples:
- `Meeting_Minutes_Q3_Roadmap_2026-08-12.docx`
- `Meeting_Minutes_Acme_Kickoff_2026-05-20.docx`
- `Meeting_Minutes_Architecture_Review_2026-11-04.docx`

If the user prefers a different convention, ask once and use that throughout the session.

## Notes

This skill re-inspects the template on each invocation, which is fast and works reliably even when users switch between multiple templates. If a user repeatedly uses the same template, they can simply re-attach it each time — no per-template configuration needed.

For organizations with strict brand standards, the unzip/zip preservation approach maintains all custom XML including:
- Embedded fonts
- Custom paragraph and character styles
- Headers and footers
- Document properties (metadata)
- Embedded images and logos

---

*Maintained by Aedonyx — AI Literacy for Knowledge Workers — aedonyx.io*
