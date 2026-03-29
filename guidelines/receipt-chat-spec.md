# Receipt Chat — Implementation Specification

> **Purpose:** Complete implementation spec for the DeepFlow "Receipt Chat" panel — a thermal-receipt-printer-inspired chat UI.
> **Target:** Claude Code implementation agent.
> **Context:** This chat panel sits inside the resizable panel system defined in [brief.md](brief.md). It replaces the default chat panel styling with a receipt-printer aesthetic.
> **Architecture:** Chat behaviour (three-layer architecture, WebSocket events, context inference) is defined in [chat-architecture.html](chat-architecture.html). This spec covers **visual design and React components only**.

---

## 1. Design Philosophy

This is NOT a messaging app. It is a receipt printer.

- **Receipt printer / thermal paper aesthetic.** Think old accountant receipt printers: continuous paper tape, monospace type, minimal ornamentation. The chat panel reads like a printed receipt unrolling from a machine.
- **Text-first, flat, minimal chrome.** No bubbles, no avatars in messages, no rounded cards. Just text on paper.
- **Monospace typography throughout.** Every character in the chat panel uses a monospace font. No exceptions.
- **Full width of its panel.** The chat panel sits inside `react-resizable-panels`. The receipt tape fills the entire width of the panel (no centred strip, no max-width wrapper on the container — only on the content).
- **Dark mode with paper-like content area.** The container background is dark (like looking at a receipt printer). The receipt tape itself is a slightly lighter "paper" colour. The contrast creates the illusion of paper on a dark desk.
- **No visual hierarchy through size.** User and machine messages are the same font size. Hierarchy is communicated through colour, borders, and indentation only.

---

## 2. Design Tokens / CSS Variables

All values extracted from the working prototype. These MUST be used exactly.

### Colours

| Token | Value | Usage |
|-------|-------|-------|
| `--rc-bg-dark` | `#0a0c10` | Dark background behind paper (visible in thread breaks) |
| `--rc-paper` | `#1a1f2a` | Receipt paper / tape background |
| `--rc-paper-edge` | `#222833` | Paper edge / subtle borders |
| `--rc-text-primary` | `#c8cdd5` | Main text colour (user and machine messages) |
| `--rc-text-dim` | `#8b919a` | Secondary text (disclosure labels, tool icons) |
| `--rc-text-muted` | `#555b65` | Tertiary text (timestamps, datelines, decorative elements) |
| `--rc-cyan` | `#00D4F5` | Accent colour — user message border, prompt character, input focus, brand hex icon |
| `--rc-cyan-dim` | `rgba(0, 212, 245, 0.15)` | Subtle cyan tint (reserved for hover states) |
| `--rc-green` | `#4ade80` | Success — completed task checkmarks |
| `--rc-amber` | `#f59e0b` | Running/in-progress — spinner icons, running status |
| `--rc-red` | `#f87171` | Error — error message border and icon |
| `--rc-red-dim` | `rgba(248, 113, 113, 0.08)` | Error message background tint |

### Typography

| Token | Value |
|-------|-------|
| `--rc-font-mono` | `'JetBrains Mono', 'SF Mono', 'Fira Code', monospace` |
| Base font size | `13px` |
| Line height | `1.65` |
| Font weight (normal) | `400` |
| Font weight (light) | `300` (timestamps, datelines, brand text) |
| Font weight (medium) | `500` (prompt character ›) |
| Timestamp font size | `9.5px` |
| Disclosure label font size | `11.5px` |
| Disclosure content font size | `12px` |
| Brand text font size | `10px` |
| Dateline font size | `10.5px` |
| Chip font size | `11px` |
| Input prompt font size | `14px` |

### Spacing

| Token | Value | Usage |
|-------|-------|-------|
| Receipt tape padding | `20px` horizontal, `20px` top, `24px` bottom | Content inset from panel edges |
| Message vertical padding | `6px 0` | Each message block |
| Message gap (small) | `14px` | Between consecutive messages in a conversation |
| Message gap (large) | `22px` | Between conversation blocks |
| Message margin-top (adjacent) | `2px` | Between `.msg + .msg` |
| Thread header padding | `16px 0 4px` | Above thread header |
| Thread dateline padding | `4px 0 14px` | Below dateline |
| Input area padding | `12px 20px 16px` | Input section |
| Input chip gap | `6px 10px` | Between chips (row-gap × column-gap) |
| Input chip padding | `4px 0 4px 10px` | Chip internal spacing |
| Disclosure toggle padding | `5px 8px` | Toggle button |
| Disclosure content padding | `6px 8px 8px 22px` | Expanded content |
| Disclosure content margin-left | `12px` | Indentation from left edge |

### Borders

| Element | Border |
|---------|--------|
| User message left border | `3px solid var(--rc-cyan)` |
| Error message left border | `3px solid var(--rc-red)` |
| Chip left border | `2.5px solid <colour>` |
| Input field bottom border | `1px solid #252a35` (default), `var(--rc-cyan)` (focus) |
| Input area top border | `1px solid #1e2430` |
| Container left/right border | `1px solid #000` |
| Disclosure content left border | `1px solid #252a35` |
| Dateline rule (thread break) | `1px solid #252a35` |

### Animation Timings

| Animation | Duration | Easing |
|-----------|----------|--------|
| Print-in (message appear) | `0.3s` | `ease-out` |
| Print-in stagger per message | `0.05s` increments | — |
| Disclosure chevron rotate | `0.2s` | default |
| Disclosure toggle background | `0.15s` | default |
| Input field border focus | `0.2s` | default |
| Send button opacity/colour | `0.15s` | default |
| Task link underline | `0.15s` | default |

---

## 3. Layout Structure

### Panel Position

The receipt chat occupies the **Chat Panel** slot in the shell layout:

```
Sidebar (56px) → Chat Panel (280–400px) → Content Area (flex) → Detail Panel (320–480px)
```

The receipt chat fills 100% width and 100% height of the Chat Panel.

### Internal Structure

```
┌─────────────────────────────────────────┐
│ .receipt-container                       │ ← flex column, 100% height
│ ┌─────────────────────────────────────┐ │
│ │ .receipt-scroll                     │ │ ← flex: 1, overflow-y: auto
│ │ ┌─────────────────────────────────┐ │ │
│ │ │ .receipt-tape                   │ │ │ ← padding: 20px, max-width: 720px
│ │ │                                 │ │ │
│ │ │ [Thread Header]                 │ │ │
│ │ │ [Messages...]                   │ │ │
│ │ │ [Thread Break]                  │ │ │
│ │ │ [Thread Header]                 │ │ │
│ │ │ [Messages...]                   │ │ │
│ │ │                                 │ │ │
│ │ └─────────────────────────────────┘ │ │
│ └─────────────────────────────────────┘ │
│ ┌─────────────────────────────────────┐ │
│ │ .input-area                         │ │ ← pinned to bottom, border-top
│ │ [Shortcut Chips]                    │ │
│ │ [› Input Field               ↵]    │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Key Layout CSS

```css
.receipt-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  width: 100%;
  background: var(--rc-paper);
  border-left: 1px solid #000;
  border-right: 1px solid #000;
  position: relative;
}
```

**Paper texture overlay:** A subtle noise texture via inline SVG applied as `::before` pseudo-element on `.receipt-container`. `opacity: 0.015`, `pointer-events: none`, `z-index: 0`.

**Scrollbar:** Hidden completely. `scrollbar-width: none` and `::-webkit-scrollbar { display: none; }`.

**Receipt tape max-width:** `720px` for readability. NOT centred — left-aligned within the scroll area. No max-width on the container itself.

---

## 4. Message Types

### 4a. User Message

```
│ ┃ › What's the status on the capital model?     │
│                                           08:17  │
```

- `border-left: 3px solid var(--rc-cyan)`
- `padding-left: 14px`
- `margin-left: 0`
- `color: var(--rc-text-primary)`
- Prompt character `›` prefixed before message text
  - `color: var(--rc-cyan)`, `opacity: 0.7`, `margin-right: 6px`, `font-weight: 500`
- Timestamp right-aligned: `font-size: 9.5px`, `color: var(--rc-text-muted)`, `opacity: 0.6`, `font-weight: 300`

### 4b. Machine Message

```
│ Good morning, Ali.                               │
│                                                   │
│ 3 tasks need attention today:                     │
│ • ICAAP Capital Model — blocked                   │
│ • Board Pack — due Thursday                       │
│                                           08:15  │
```

- No left border, no decoration
- `color: var(--rc-text-primary)`
- `padding-left: 0`
- Text uses `white-space: pre-wrap` and `word-break: break-word`
- Timestamp same style as user messages

### 4c. Thinking Disclosure (Collapsible)

```
│ ▶ Thinking…                                      │
```

Expanded:

```
│ ▼ Thinking…                                      │
│   │ The user is asking about the ICAAP capital    │
│   │ model status. Let me check the task board...  │
```

- Container: `margin: 4px 0`, `border-radius: 3px`
- Toggle button: `display: flex`, `align-items: center`, `gap: 6px`
  - `padding: 5px 8px`, `border-radius: 3px`
  - `background: rgba(255,255,255, 0.02)`
  - `font-size: 11.5px`, `color: var(--rc-text-dim)`
  - Hover: `background: rgba(255,255,255, 0.04)`
- Chevron: `▶` character, `font-size: 9px`, `color: var(--rc-text-muted)`
  - Rotates `90deg` when open, `transition: transform 0.2s`
- Label: `"Thinking…"`, `font-size: 11.5px`, `color: var(--rc-text-dim)`, `letter-spacing: 0.3px`
- Content (hidden by default, shown when open):
  - `padding: 6px 8px 8px 22px`
  - `font-size: 12px`, `color: var(--rc-text-muted)`, `line-height: 1.6`
  - `border-left: 1px solid #252a35`, `margin-left: 12px`, `margin-top: 2px`
- **Collapsed by default.** Clicking toggles open/closed.

### 4d. Tool Call Disclosure (Collapsible)

```
│ ▶ ⚙ Searched tasks                               │
```

Expanded:

```
│ ▼ ⚙ Searched tasks                               │
│   │ Query: project:ICAAP status:blocked           │
│   │ Found 1 task: "ICAAP Capital Model v3"        │
│   │   Status: Blocked                             │
```

- Identical structure to thinking disclosure
- Label includes tool icon: `⚙` character with `opacity: 0.6`, `margin-right: 3px`
- Label text is the tool action name (e.g. "Searched tasks", "Created task: Import Q4 actuals", "Sent message to Sarah", "Running capital model", "Updating board pack")
- **Collapsed by default.**

### 4e. Task Confirmation

```
│ ✓ Task created: Review strategic initiative proposals  │
```

- Inline display: `display: inline-flex`, `align-items: center`, `gap: 6px`
- Check icon `✓`: `color: var(--rc-green)`
- Task title is a clickable link:
  - `color: var(--rc-cyan)`
  - `text-decoration: underline`
  - `text-decoration-color: rgba(0, 212, 245, 0.3)`
  - `text-underline-offset: 2px`
  - Hover: `text-decoration-color: var(--rc-cyan)`

For running tasks:
- Icon `⟳`: `color: var(--rc-amber)`

### 4f. Error/Warning Message

```
│ ┃ ⚠ Google Calendar sync failed — token expired.     │
│ ┃                                                      │
│ ┃ This won't affect your work today.                   │
│                                                 14:58  │
```

- `border-left: 3px solid var(--rc-red)`
- `padding-left: 14px`
- `background: var(--rc-red-dim)` (subtle red tint: `rgba(248, 113, 113, 0.08)`)
- `border-radius: 0 3px 3px 0`
- Error icon `⚠`: `color: var(--rc-red)`, `margin-right: 6px`

### 4g. Bullet Lists / Status Lists

Within machine messages, lists use inline spans:

- Done items: `✓` prefix, `color: var(--rc-green)`
- Running items: `⟳` prefix, `color: var(--rc-amber)`
- Default items: `•` prefix, default text colour
- Indented with `padding-left: 2px`
- Each item: `padding: 1px 0`
- Highlight text (e.g. project names): `color: var(--rc-cyan)`, `opacity: 0.85`

---

## 5. Thread System

### Thread Header

Appears at the start of each thread (conversation session).

```
                    ⬡ DEEPFLOW
          ——— Mon, 23 Mar · 08:15 ———
```

- Centred text alignment
- **Brand line:**
  - `⬡` hexagon icon: `color: var(--rc-cyan)`, `opacity: 0.6`, `font-size: 11px`
  - `"DEEPFLOW"` text: NOT caps-locked text — use CSS `text-transform: uppercase`
  - `font-size: 10px`, `letter-spacing: 2.5px`, `color: var(--rc-text-muted)`, `font-weight: 300`
  - `margin-bottom: 10px`
- **Dateline:**
  - Format: `"Mon, 23 Mar · 08:15"` (abbreviated day, date, mid-dot, time)
  - `font-size: 10.5px`, `color: var(--rc-text-muted)`, `letter-spacing: 1px`, `font-weight: 300`
  - Decorative dashes on either side using `::before` and `::after` pseudo-elements:
    - `flex: 1`, `max-width: 60px`, `height: 1px`
    - `background: linear-gradient(to right, var(--rc-text-muted), transparent)` (left side)
    - `background: linear-gradient(to left, var(--rc-text-muted), transparent)` (right side)
    - `opacity: 0.3`
  - Container: `display: flex`, `align-items: center`, `justify-content: center`, `gap: 10px`
  - `padding: 4px 0 14px`

### Thread Break

The visual separator between threads. This is a multi-part element that creates the illusion of the receipt paper ending and a new one beginning.

**Structure (top to bottom):**

1. **Fade out** (48px): `linear-gradient(to bottom, var(--rc-paper), var(--rc-bg-dark))`
2. **Dark gap** (24px): solid `var(--rc-bg-dark)` background
3. **Dateline rule**: centred timestamp on a dark background with thin lines on each side
   - Background: `var(--rc-bg-dark)`
   - Lines: `::before` and `::after` with `flex: 1`, `height: 1px`, `background: #252a35`
   - Timestamp: `font-size: 10px`, `color: var(--rc-text-muted)`, `letter-spacing: 1px`, `opacity: 0.6`, `white-space: nowrap`
   - Container: `display: flex`, `align-items: center`, `gap: 14px`, `padding: 0 24px`
4. **Small gap** (16px): solid `var(--rc-bg-dark)` background
5. **Fade in** (48px): `linear-gradient(to bottom, var(--rc-bg-dark), var(--rc-paper))`

**Critical:** The thread break bleeds to the panel edges. Apply `margin: 0 -20px` to negate the receipt tape padding, so the gradients span the full container width.

After the fade-in, a new Thread Header appears (on the paper background).

### When Threads Break

- User explicitly starts a new thread (button/command)
- Significant time gap since last message (configurable threshold, default: 2+ hours)
- New login / session start

---

## 6. Input Area

### Structure

```
┌──────────────────────────────────────────┐
│ What's blocked?  Summarise  Run model    │ ← shortcut chips
│ Create task  Board pack  Next meeting    │
│                                          │
│ ›  Message DeepFlow…                  ↵  │ ← input row
└──────────────────────────────────────────┘
```

### Input Area Container

- Pinned to bottom of `.receipt-container`
- `position: relative`, `z-index: 2`
- `padding: 12px 20px 16px`
- `border-top: 1px solid #1e2430`
- `background: var(--rc-paper)`
- Safe area: `padding-bottom: env(safe-area-inset-bottom, 0)` on the container

### Shortcut Chips

Flat, border-left-only chips. No background, no pill shape, no rounded corners.

- Container: `display: flex`, `flex-wrap: wrap`, `gap: 6px 10px`, `margin-bottom: 10px`
- Each chip:
  - `font-family: var(--rc-font-mono)`, `font-size: 11px`
  - `padding: 4px 0 4px 10px`
  - `border: none` (no border around the chip)
  - `border-left: 2.5px solid <colour>`
  - `background: transparent`
  - `color: var(--rc-text-dim)`
  - `cursor: pointer`, `white-space: nowrap`, `letter-spacing: 0.2px`
  - Hover: `background: rgba(255,255,255, 0.03)`, `color: var(--rc-text-primary)`

**Chip colour variants:**

| Variant | Left border colour | Hover text colour | Usage |
|---------|-------------------|-------------------|-------|
| `cyan` | `var(--rc-cyan)` | `var(--rc-cyan)` | Status queries: "What's blocked?", "Summarise" |
| `amber` | `var(--rc-amber)` | `var(--rc-amber)` | Actions: "Run model", "Create task" |
| `grey` | `#555b65` | `var(--rc-text-primary)` | Navigation: "Board pack", "Next meeting" |

### Input Row

- `display: flex`, `align-items: center`, `gap: 10px`
- **Prompt character:** `›`
  - `color: var(--rc-cyan)`, `opacity: 0.5`, `font-size: 14px`, `font-weight: 500`, `flex-shrink: 0`
- **Input field:**
  - `flex: 1`
  - `font-family: var(--rc-font-mono)`, `font-size: 13px`
  - `color: var(--rc-text-primary)`
  - `background: transparent`
  - `border: none` except `border-bottom: 1px solid #252a35`
  - `padding: 6px 0`
  - `outline: none`, `caret-color: var(--rc-cyan)`
  - Focus: `border-bottom-color: var(--rc-cyan)` with `transition: border-color 0.2s`
  - Placeholder: `"Message DeepFlow…"`, `color: var(--rc-text-muted)`, `opacity: 0.5`
- **Send button:** `↵` character
  - `background: none`, `border: none`
  - `color: var(--rc-text-muted)`, `font-size: 16px`, `opacity: 0.4`
  - `padding: 4px`
  - Hover: `opacity: 1`, `color: var(--rc-cyan)`, `transition: 0.15s`

---

## 7. Animations

### Print-in Animation

Messages appear as if being printed — sliding in from slightly above with a fade.

```css
@keyframes printIn {
  from { opacity: 0; transform: translateY(-6px); }
  to { opacity: 1; transform: translateY(0); }
}
```

- Applied to each `.msg` element: `animation: printIn 0.3s ease-out both`
- Staggered: each subsequent message delays by `0.05s` (1st: 0s, 2nd: 0.05s, 3rd: 0.1s, etc.)
- Only apply stagger to initial load (first 5 messages). New messages arriving via WebSocket get the animation without stagger delay.

### Disclosure Toggle

- Chevron character `▶` rotates to `90deg` when disclosure opens
- `transition: transform 0.2s`

### Input Focus

- Input bottom border transitions from `#252a35` to `var(--rc-cyan)` on focus
- `transition: border-color 0.2s`

---

## 8. Responsive Behaviour

- **Full width at all panel sizes.** The receipt chat fills 100% of the panel. No responsive breakpoints within the component itself — the panel system handles sizing.
- **Content max-width: 720px** on `.receipt-tape` for text readability. Left-aligned, not centred.
- **Panel min-width: 280px** (from brief). At this width, the chat is still fully functional.
- **Mobile safe area:** `padding-bottom: env(safe-area-inset-bottom, 0)` on the input area.
- **Scrollbar hidden:** No visible scrollbar. Scroll via touch/trackpad/mousewheel.
- **Small viewport adjustments** (at `max-width: 480px` — for when panel is narrow or on mobile):
  - Receipt tape padding reduces to `16px` horizontal
  - Thread break margin adjusts to `-16px`
  - Thread break dateline padding reduces to `0 20px`
  - Input area padding reduces to `10px 16px 14px`
  - Base font size reduces to `12.5px`

---

## 9. Accessibility

### Disclosure Blocks

- Toggle must be a `<button>` element (not a div with click handler)
- `aria-expanded="false"` (collapsed) / `aria-expanded="true"` (open)
- `aria-controls="disclosure-content-{id}"` pointing to the content element
- Content element has `id="disclosure-content-{id}"`
- Content is hidden with `display: none` when collapsed (not just visually hidden)
- Chevron rotation is decorative — `aria-hidden="true"` on the chevron span

### Timestamps

- Use `<time datetime="2026-03-23T08:15:00">08:15</time>` elements
- Datelines also use `<time>` with full ISO datetime

### Colour Contrast

- Primary text (`#c8cdd5`) on paper (`#1a1f2a`): ratio ~7.2:1 (passes AAA)
- Dim text (`#8b919a`) on paper (`#1a1f2a`): ratio ~4.5:1 (passes AA)
- Muted text (`#555b65`) on paper (`#1a1f2a`): ratio ~2.5:1 — used for timestamps only (supplementary info). Pair with semantic context (position, format) for comprehension. Consider bumping to `#6b7280` (~3.5:1) if AA compliance is required on timestamps.
- Cyan (`#00D4F5`) on paper (`#1a1f2a`): ratio ~7.8:1 (passes AAA)
- Green (`#4ade80`) on paper: ~8.5:1 (passes AAA)
- Red (`#f87171`) on paper: ~6.7:1 (passes AA)

### Keyboard Navigation

- **Chips:** Arrow keys navigate between chips. Enter/Space activates.
- **Disclosures:** Tab focuses the toggle button. Enter/Space toggles open/closed.
- **Input field:** Tab focuses the input. Enter sends. Shift+Enter for newline.
- **Send button:** Tab-focusable. Enter/Space activates.
- **Links in messages:** Tab-focusable, standard link behaviour.

### Screen Reader

- Chat region: `role="log"`, `aria-label="Chat messages"`, `aria-live="polite"` for new messages
- Input area: `aria-label="Chat input"`
- Send button: `aria-label="Send message"`
- Chips: `role="toolbar"`, each chip is a `<button>` with descriptive text

---

## 10. Component API (React)

### Component Tree

```tsx
<ReceiptChat>
  <ReceiptScroll>
    <Thread>
      <ThreadHeader brand="DeepFlow" date="Mon, 23 Mar · 08:15" />
      <Message type="machine" timestamp="08:15">
        Good morning, Ali. 3 tasks need attention today...
      </Message>
      <MessageGap size="lg" />
      <Message type="user" timestamp="08:17">
        What's the status on the capital model?
      </Message>
      <MessageGap />
      <Message type="machine" timestamp="08:17">
        <ThinkingDisclosure>
          The user is asking about the ICAAP capital model status...
        </ThinkingDisclosure>
        <ToolCallDisclosure tool="Searched tasks">
          Query: project:ICAAP status:blocked...
        </ToolCallDisclosure>
        The capital model is blocked on two inputs...
      </Message>
      <MessageGap />
      <Message type="machine" variant="error" timestamp="14:58">
        <ErrorIcon /> Google Calendar sync failed...
      </Message>
    </Thread>
    <ThreadBreak timestamp="Mon, 23 Mar · 11:42" />
    <Thread>
      <ThreadHeader brand="DeepFlow" date="Mon, 23 Mar · 11:42" />
      ...
    </Thread>
  </ReceiptScroll>
  <InputArea>
    <ShortcutChips chips={chips} onChipClick={handleChip} />
    <ChatInput
      placeholder="Message DeepFlow…"
      onSend={handleSend}
    />
  </InputArea>
</ReceiptChat>
```

### Type Definitions

```typescript
// ── Message Types ──

type MessageType = 'user' | 'machine';
type MessageVariant = 'default' | 'error';

interface ChatMessage {
  id: string;
  type: MessageType;
  variant?: MessageVariant;
  content: string;              // Plain text or structured content
  timestamp: string;            // ISO 8601
  disclosures?: Disclosure[];   // Thinking blocks, tool calls
  taskConfirmations?: TaskConfirmation[];
  statusItems?: StatusItem[];
}

interface Disclosure {
  id: string;
  kind: 'thinking' | 'tool';
  label: string;                // "Thinking…" or "Searched tasks"
  content: string;              // Expanded content text
  defaultOpen?: boolean;        // Default: false
}

interface TaskConfirmation {
  id: string;
  title: string;
  url: string;
  status: 'complete' | 'running';
}

interface StatusItem {
  text: string;
  status: 'done' | 'running' | 'default';
}

// ── Chip Types ──

type ChipVariant = 'cyan' | 'amber' | 'grey';

interface ShortcutChip {
  id: string;
  label: string;
  variant: ChipVariant;
  action: string;               // The text to send when clicked
}

// ── Thread Types ──

interface ChatThread {
  id: string;
  startedAt: string;            // ISO 8601
  messages: ChatMessage[];
}

// ── Component Props ──

interface ReceiptChatProps {
  threads: ChatThread[];
  chips: ShortcutChip[];
  onSendMessage: (text: string) => void;
  onChipClick: (chip: ShortcutChip) => void;
  className?: string;
}

interface ReceiptScrollProps {
  children: React.ReactNode;
  onScrollTop?: () => void;     // Triggered when user scrolls to top (load older threads)
}

interface ThreadProps {
  children: React.ReactNode;
}

interface ThreadHeaderProps {
  brand: string;                // "DeepFlow"
  date: string;                 // Formatted date string: "Mon, 23 Mar · 08:15"
  dateTime: string;             // ISO 8601 for <time> element
}

interface ThreadBreakProps {
  timestamp: string;            // Formatted: "Mon, 23 Mar · 11:42"
  dateTime: string;             // ISO 8601
}

interface MessageProps {
  type: MessageType;
  variant?: MessageVariant;
  timestamp: string;            // Formatted time: "08:15"
  dateTime: string;             // ISO 8601 for <time> element
  children: React.ReactNode;    // Message content (text + disclosures)
}

interface MessageGapProps {
  size?: 'sm' | 'lg';          // Default: 'sm' (14px), 'lg' (22px)
}

interface ThinkingDisclosureProps {
  id: string;
  defaultOpen?: boolean;
  children: React.ReactNode;    // Thinking content
}

interface ToolCallDisclosureProps {
  id: string;
  tool: string;                 // Tool action name
  icon?: string;                // Default: "⚙"
  defaultOpen?: boolean;
  children: React.ReactNode;    // Tool call details
}

interface TaskConfirmProps {
  title: string;
  url: string;
  status: 'complete' | 'running';
}

interface InputAreaProps {
  children: React.ReactNode;
}

interface ShortcutChipsProps {
  chips: ShortcutChip[];
  onChipClick: (chip: ShortcutChip) => void;
}

interface ChatInputProps {
  placeholder?: string;
  onSend: (text: string) => void;
  disabled?: boolean;
}
```

### Component File Structure

```
src/components/chat/receipt/
├── ReceiptChat.tsx              ← Root container
├── ReceiptScroll.tsx            ← Scroll area with infinite scroll up
├── Thread.tsx                   ← Thread wrapper
├── ThreadHeader.tsx             ← ⬡ brand + dateline
├── ThreadBreak.tsx              ← Fade out → gap → dateline → fade in
├── Message.tsx                  ← User/machine message with variants
├── MessageGap.tsx               ← Spacing between messages
├── ThinkingDisclosure.tsx       ← Collapsible thinking block
├── ToolCallDisclosure.tsx       ← Collapsible tool call block
├── TaskConfirm.tsx              ← ✓/⟳ task link
├── StatusItem.tsx               ← ✓/⟳/• list item
├── InputArea.tsx                ← Bottom input section
├── ShortcutChips.tsx            ← Chip bar
├── ChatInput.tsx                ← Input field + send button
├── receipt-tokens.css           ← CSS custom properties
└── types.ts                     ← All TypeScript types above
```

---

## 11. Integration with Existing Brief

### Replaces Default Chat Panel

The receipt chat replaces the default `<ChatPanel />` component referenced in the brief's shell layout. The component occupies the same slot:

```
<ShellLayout>
  <Sidebar />
  <ReceiptChat />    ← was <ChatPanel />
  <div>
    <TopBar />
    <HubBar />
    <main>{children}</main>
  </div>
  <DetailPanel />
</ShellLayout>
```

### Panel Resize Integration

- Uses `react-resizable-panels` as specified in the brief
- Panel constraints: `minSize: 280px`, `maxSize: 400px`, `defaultSize: 320px`
- Double-click drag handle to collapse (collapses to 0px, icon in sidebar reactivates)
- Panel size persisted via Zustand `preferences` store to IndexedDB

### WebSocket Message Rendering

Messages arrive via WebSocket events (as per brief section 10):

```
WS event: chat.message { role: 'user'|'assistant', content, taskId, metadata }
```

Mapping to receipt chat:
- `role: 'user'` → `<Message type="user">`
- `role: 'assistant'` → `<Message type="machine">`
- `metadata.thinking` → `<ThinkingDisclosure>`
- `metadata.toolCalls[]` → `<ToolCallDisclosure>` for each tool call
- `metadata.error: true` → `<Message type="machine" variant="error">`
- `metadata.taskConfirmation` → `<TaskConfirm>`

New messages append to the current thread. The print-in animation plays. Auto-scroll to bottom on new message (unless user has scrolled up — then show a "new message" indicator).

### Chat History API Pagination

- On initial load: fetch the most recent thread (latest N messages)
- When user scrolls to the top of `<ReceiptScroll>`: trigger `onScrollTop` callback
- Callback fetches the previous thread/page from the chat history API
- Older threads are prepended above the current content
- Thread breaks are inserted based on time gaps between the last message of the older thread and the first message of the current thread
- Use TanStack Query for chat history (as specified in the brief): `staleTime: 5min`, `gcTime: 30min`

### Chat Context Scoping

As per the chat architecture spec, the chat panel's context changes based on the current view:

| View | Chat context | Placeholder text |
|------|-------------|-----------------|
| Dashboard | General | "Ask about your workflow…" |
| Graph View | Active project | "Ask about [Project Name]…" |
| Workspace (task) | Active task | "Ask about this task…" |
| List View | Active project | "Ask about [Project Name]…" |

The thread header and shortcut chips should update when context changes. A context change MAY trigger a thread break if the user navigates to a completely different project.

---

## 12. Font Loading

JetBrains Mono must be loaded for the receipt chat. Add via Google Fonts:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@300;400;500&display=swap" rel="stylesheet">
```

Weights needed: 300 (light — timestamps, datelines), 400 (regular — body text), 500 (medium — prompt character).

The rest of the app uses Euclid Circular B. JetBrains Mono is ONLY for the receipt chat panel. Do not apply it elsewhere.

---

## 13. Anti-Patterns (Do NOT Do These)

- **No chat bubbles.** Messages are not in rounded rectangles. They're flat text on paper.
- **No avatars in messages.** No user/bot avatar icons next to messages.
- **No alternating alignment.** Both user and machine messages are left-aligned. No right-aligned user messages.
- **No background colours on messages** (except error variant's subtle red tint).
- **No sans-serif fonts.** Everything in the chat panel is monospace.
- **No emoji reactions.** This is a receipt, not Slack.
- **No typing indicators with animated dots.** If needed, show a disclosure-style "Thinking…" that converts to actual content when the response arrives.
- **No message grouping with shared timestamps.** Each message gets its own timestamp.
