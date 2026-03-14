---
name: accessibility-interaction-patterns
description: Explains accessible interaction patterns in web applications including keyboard navigation, focus management, semantic controls, form feedback, announcements, and motion preferences. Use when building interactive UI that must work well for keyboard, screen reader, and assistive technology users.
---

# Accessibility Interaction Patterns

## Overview

Accessible interaction design is about making interface behavior usable beyond the default mouse-and-vision path.

In modern web apps, interaction bugs often come from custom components that break:

- keyboard access
- focus order
- screen reader announcements
- visible state feedback
- motion and timing expectations

Accessibility is not a separate polish step. It is part of how interactive components are designed.

## Start with Native HTML

The most reliable accessibility pattern is to use the correct built-in element first.

| User Intent | Prefer | Avoid |
|-------------|--------|-------|
| Trigger an action | `<button>` | Clickable `<div>` |
| Navigate to another location | `<a href>` | Button that changes URL |
| Input text | `<input>` / `<textarea>` | `contenteditable` without strong reason |
| Choose from options | `<select>`, radios, checkboxes | Custom listbox by default |
| Expand/collapse content | `<button>` + hidden region | Clickable heading only |

Native elements already handle many things correctly:

- keyboard support
- focusability
- semantics
- disabled states
- browser and assistive technology expectations

If you replace them with generic elements, you must rebuild all of that behavior yourself.

## Keyboard Interaction

Every interactive feature should work without a mouse.

### Minimum Expectations

- `Tab` moves through interactive controls in a sensible order
- `Shift+Tab` moves backward
- `Enter` and `Space` activate controls where appropriate
- Arrow keys work for widgets that expect them
- Focus remains visible at all times

### Anti-Pattern

```jsx
<div onClick={save}>Save</div>
```

Problems:

- Not keyboard-focusable by default
- No button semantics
- No default keyboard activation

### Better

```jsx
<button type="button" onClick={save}>
  Save
</button>
```

## Focus Management

Focus is the user's current interaction location.

In SPAs, focus does not reset automatically the way it often does on full page loads, so developers need to manage it intentionally.

### When Focus Should Move

- Opening a modal
- Closing a modal
- After client-side navigation
- Showing inline validation for a failed submit
- Revealing new interactive content

### Modal Example

When a modal opens:

1. Move focus into the modal
2. Keep tab focus inside while open
3. Restore focus to the triggering element when closed

```javascript
const trigger = document.activeElement;
openModal();
modalRef.current.focus();

function close() {
  closeModal();
  trigger?.focus();
}
```

### Route Change Example

After client-side navigation, move focus to the main heading or main landmark so screen reader and keyboard users know the page changed.

```javascript
router.navigate('/settings');
requestAnimationFrame(() => {
  document.querySelector('main h1')?.focus();
});
```

This usually requires the target element to be programmatically focusable:

```jsx
<h1 tabIndex="-1">Settings</h1>
```

## Visible Focus

If a user tabs to an element, they must be able to see where they are.

Avoid removing focus outlines unless you replace them with an equally visible style.

```css
:focus-visible {
  outline: 3px solid #1e66f5;
  outline-offset: 2px;
}
```

If your design system suppresses focus rings globally, it is introducing an accessibility bug.

## Semantics and ARIA

Use ARIA to enhance semantics, not to compensate for missing structure when native HTML would work.

### Good Uses

- `aria-expanded` on a disclosure button
- `aria-controls` to describe relationship
- `aria-current="page"` for active navigation link
- `aria-busy="true"` while content is updating
- `aria-live` for dynamic announcements

### Warning

No ARIA pattern should be used as an excuse to skip proper keyboard and focus behavior.

For example:

```jsx
<div role="button" tabIndex="0">Open</div>
```

This is still usually worse than:

```jsx
<button type="button">Open</button>
```

because you still need to implement keyboard activation and other expected behavior correctly.

## Form Interaction Patterns

Forms fail accessibility when labels, errors, or submission feedback are unclear.

### Label Every Control

```jsx
<label htmlFor="email">Email</label>
<input id="email" name="email" type="email" />
```

Placeholder text is not a label.

### Connect Errors to Inputs

```jsx
<label htmlFor="email">Email</label>
<input
  id="email"
  aria-invalid="true"
  aria-describedby="email-error"
/>
<p id="email-error">Enter a valid email address.</p>
```

### Error Handling Rules

- Show errors near the field
- Move focus to the first invalid field or error summary after submit
- Keep error text persistent long enough to be read
- Do not rely on color alone to indicate error state

## Async Interaction Feedback

Interactive systems often change after a delay: saving, loading, optimistic mutation, search, upload.

Users need feedback about those transitions.

### Useful States

- Loading
- Saving
- Success
- Error
- Pending / syncing

### Example

```jsx
<button type="submit" aria-busy={saving} disabled={saving}>
  {saving ? 'Saving...' : 'Save'}
</button>
```

### Announcements

When an important status changes without moving focus, use a live region.

```jsx
<div aria-live="polite">{statusMessage}</div>
```

Use polite announcements for normal updates and assertive ones sparingly for urgent interruption-worthy feedback.

## Common Widgets

Custom widgets are where accessibility bugs multiply.

### Modal / Dialog

Needs:

- Focus moved inside on open
- Focus trapped while open
- Escape closes when appropriate
- Background content not interactable
- Clear label and description

### Dropdown Menu

Needs:

- Button trigger
- Clear open/closed state
- Keyboard navigation between items
- Escape to close
- Focus return to trigger

### Tabs

Needs:

- Tab list semantics
- Active tab indication
- Keyboard movement between tabs
- Correct relationship between tab and panel

### Accordion / Disclosure

Usually simplest when built from:

- a `<button>`
- a region that is shown/hidden
- `aria-expanded` reflecting current state

## Pointer, Touch, and Drag

Accessibility is not only about screen readers.

Also consider:

- Touch target size
- Hover-only behavior
- Drag-and-drop alternatives
- Fine motor limitations

### Rules of Thumb

- Do not hide critical actions behind hover only
- Ensure controls are large enough to activate reliably
- Provide a non-drag fallback for reordering
- Avoid gestures that require precision with no alternative

If drag-and-drop exists, offer keyboard buttons like "Move up" and "Move down" when practical.

## Motion and Timing

Some interactions become unusable when they rely on animation or time pressure.

### Prefer Reduced Motion

Honor user motion preferences for large transitions, parallax, and animated feedback.

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

### Timing Concerns

Be careful with:

- Toasts that disappear too quickly
- Session warnings without enough response time
- Content that changes automatically while being read

## SPA-Specific Pitfalls

Client-rendered apps introduce recurring accessibility problems:

| Problem | Typical Cause |
|---------|---------------|
| No announcement after navigation | URL changed, but focus stayed on old trigger |
| Keyboard trap | Modal or drawer lacks escape/focus handling |
| Hidden content still focusable | CSS-only hiding without interaction control |
| Screen reader misses status change | Async update has no live region |
| Focus lost after rerender | Node unmounted or replaced unexpectedly |

## Testing Checklist

At minimum, test these flows:

1. Navigate the page with only keyboard
2. Confirm focus is always visible
3. Open and close dialogs without losing context
4. Submit a form with errors and verify error feedback is announced
5. Trigger loading or save states and verify feedback is perceivable
6. Try a screen reader pass for the main flows
7. Check reduced-motion behavior for major transitions

If a component is too hard to test with keyboard only, it is usually too complex.

## Rules of Thumb

- Prefer native controls over custom ones
- Do not use click-only interactions for critical UI
- Manage focus deliberately after major UI changes
- Keep visible focus indicators
- Use ARIA to clarify behavior, not to fake semantics
- Announce async status changes when focus does not move
- Treat accessibility regressions as interaction bugs, not visual polish issues

## Related Skills

- `routing-patterns` for navigation behavior in SPAs
- `hydration-patterns` for interactive islands and client activation
- `optimistic-ui-patterns` for accessible pending and error states during mutations
