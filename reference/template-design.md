# Email template design

Use this when designing or redesigning a reusable Hermes email template. The goal is an editable, email-safe shell that looks intentional after generated body copy is inserted.

## Mental model

- The template is a reusable shell, not a one-off email body.
- Hermes has two relevant agent roles:
  - The **normal Hermes Agent** operates the platform: triggers, drafts, analytics, senders, domains, data sources, and CLI-like workflows.
  - The **Hermes Email Agent** designs reusable email templates and knows the editable email IR, editor capabilities, runtime slots, and email-safe layout constraints.
- The normal Hermes Agent must delegate template design to the Hermes Email Agent/tooling. It should not hand-author template JSON, runtime-slot syntax, React Email/Tiptap JSON, or low-level editor operations.
- Template design requests should produce a non-persistent preview first. Applying that preview is approval-gated and saves a draft only; publishing is a separate user action.
- Think in three related layers:
  - **Page**: outside canvas/background, email width, global font, preview text.
  - **Email surface/card**: the main React Email/Tiptap canvas and card surface. This is document theme, not an editable wrapper block.
  - **Content**: editable header, hero, body sections, inner cards, CTA, footer copy, dividers, images, spacers, and runtime slots.
- Do not recreate imported HTML wrappers like `email-shell` or `email-card` as content sections. Translate only visible inner sections into editable nodes.
- Keep Page changes scoped. Width, page background, global font, preview text, and compliance settings change only when the user clearly asks for setup/page/canvas/outside changes.
- Email surface/card background and radius are canonical document theme settings: use `update_theme` with `themePatch.colors.surface` and/or `themePatch.radius.lg` when the user asks for the template/card/email surface. Do not create a fake top-level wrapper section to simulate the card.
- Inner cards or named content blocks still use editable content node styles.
- If the user says only “background” and does not name page/canvas/outside or section/block/content/card/surface, ask one short clarification instead of guessing.

## Quality bar

Good Hermes templates are quiet, readable, and production-ready:

- A clear hierarchy: brand/header, primary headline, short supporting copy, optional proof/card, optional CTA, footer.
- Consistent spacing: most sections use 24-40px horizontal padding and 16-32px vertical rhythm.
- Left-aligned body content by default. Center only logos, badges, short hero blocks, or footer details when it fits the design.
- One primary CTA unless the user asks for multiple actions.
- Restrained color: one primary action color, one soft content background/surface, muted secondary text, subtle borders.
- Email-safe styling: simple blocks, inline-safe colors, border radii, borders, no complex positioning, no CSS grid dependencies, no decorative shadow-heavy layouts.
- Every visible block should be reproducible manually in the editor: text, image, button, divider, card section, table/list, runtime slot.

## Layout patterns

Use one of these unless the user asks for a very specific format:

1. **Clean lifecycle note**
   Header logo/text, compact headline, 2-3 short paragraphs, AI body slot, simple CTA, footer with unsubscribe.

2. **Signal / insight card**
   Header, badge, headline, short explanation, soft bordered card with metrics or bullets, AI body slot, CTA, footer.

3. **Product announcement**
   Header, hero image or logo, headline, intro, feature list, AI body slot for personalized angle, CTA, footer.

4. **Executive plain-card**
   Minimal brand line, short title, mostly paragraph copy, AI body slot, low-emphasis footer. Use for founder/CSM-style outreach.

5. **Newsletter digest**
   Header, intro, repeated cards/sections, optional table/list, AI body slot, footer. Keep density scan-friendly.

## Operation recipes

Use these shapes when operating the editor. Prefer simple, valid operations over clever structure.

### Pure card/surface change

If the user says “card bg to red”, “make surface green”, “email card background #fff”, or only asks for card/surface radius:

- Return only `update_theme`.
- Do not insert header, body, CTA, footer, spacer, divider, or placeholder content.
- Do not change `settings.backgroundColor`; that is the outside page.

```json
[
  {
    "type": "update_theme",
    "themePatch": {
      "colors": { "surface": "#e53935" }
    }
  }
]
```

### Valid blank-template build

When creating a template from blank, insert visible top-level content sections. Each section should contain real content, not an empty wrapper.

```json
[
  {
    "type": "insert_node",
    "node": {
      "id": "section_header_1",
      "type": "section",
      "style": {
        "paddingTop": "32px",
        "paddingRight": "32px",
        "paddingBottom": "16px",
        "paddingLeft": "32px"
      },
      "children": [
        {
          "id": "heading_header_1",
          "type": "heading",
          "attrs": { "level": 1 },
          "text": "A clear email headline",
          "style": { "margin": "0", "fontSize": "30px", "lineHeight": "1.2" }
        }
      ]
    }
  },
  {
    "type": "insert_node",
    "afterId": "section_header_1",
    "node": {
      "id": "section_body_1",
      "type": "section",
      "style": {
        "paddingTop": "8px",
        "paddingRight": "32px",
        "paddingBottom": "32px",
        "paddingLeft": "32px"
      },
      "children": [
        {
          "id": "paragraph_body_1",
          "type": "paragraph",
          "text": "Readable supporting copy goes here.",
          "style": { "margin": "0", "lineHeight": "1.6" }
        }
      ]
    }
  }
]
```

### Divider

Standalone horizontal rules must be direct `divider` nodes. Never wrap a divider in a `section`, and never fake a divider with a 1px section or spacer.

```json
[
  {
    "type": "insert_node",
    "afterId": "section_intro_1",
    "node": {
      "id": "divider_intro_1",
      "type": "divider",
      "style": {
        "borderTop": "1px solid #e2e8f0",
        "marginTop": "24px",
        "marginRight": "32px",
        "marginBottom": "24px",
        "marginLeft": "32px"
      }
    }
  }
]
```

### Button

Buttons must include visible label text. Do not rely on fallback renderer text.

```json
{
  "id": "button_primary_1",
  "type": "button",
  "text": "View details",
  "attrs": { "href": "https://example.com" },
  "style": {
    "backgroundColor": "#125B6D",
    "color": "#ffffff",
    "paddingTop": "12px",
    "paddingRight": "18px",
    "paddingBottom": "12px",
    "paddingLeft": "18px",
    "borderRadius": "8px"
  }
}
```

## Spacing rules

- Put real interior inset on top-level visible sections: `paddingLeft` and `paddingRight` are usually 24-40px.
- Use `paddingTop`/`paddingBottom` for vertical rhythm instead of empty wrapper sections.
- Use `spacer` only for intentional vertical gaps that the user could understand and move.
- Use spacer nodes for selectable/deleteable between-block gaps.
- Do not use fake empty paragraphs for layout.
- Do not set a first root section whose only job is outer shell breathing room.
- Do not add a top-level centered card wrapper with `maxWidth`, `margin: auto`, shell padding, or a surface background. The renderer/editor already centers the email and applies the canonical email surface from theme.
- If a full-width header/footer is desired, make that section full-width and put its own inner text/image padding on children or nested content.

## Typography rules

- H1: 28-36px desktop, 1.1-1.25 line-height, strong weight.
- H2/card titles: 18-24px.
- Body: 15-17px, 1.5-1.7 line-height.
- Footer/legal: 12-14px, muted color, 1.4-1.6 line-height.
- Use richText for inline emphasis, links, text color, font size, line height, text transform, font weight, and word break.
- Never put HTML tags inside text.

## Runtime slots

- `ai_content` is a protected block where generated personalized body copy goes.
- `unsubscribe` is a protected runtime link represented only as a richText segment with `runtimeSlot: "unsubscribe"` and `href: "{{unsubscribe_url}}"`.
- Prefer an inline unsubscribe richText segment inside a footer paragraph, such as `Amazon · Address · Unsubscribe`.
- For a standalone unsubscribe link, still create a paragraph whose only richText segment is the unsubscribe slot; do not insert a top-level `unsubscribe` node.
- Preserve existing runtime slots during redesign unless the user explicitly asks to move/remove them.
- Publish-ready marketing templates need exactly one `ai_content` and exactly one `unsubscribe`, but ordinary design drafts do not need to add missing slots.
- Do not add runtime slots for small edits like copy, colors, spacing, header, button, image, or card styling.
- Add missing runtime slots only when the user asks for publish readiness, asks for a complete reusable template, or starts from blank and asks to generate a full template.
- Never create runtime slots with literal `{{ai_content}}`, `{{unsubscribe_url}}`, or ordinary unsubscribe-looking links.

## Background targets

- Page/canvas/outside background maps to `settings.backgroundColor`.
- Template/card/email surface background maps to `theme.colors.surface`.
- Template/card/email surface radius maps to `theme.radius.lg`.
- Inner card, section, block, header, hero, footer, button, or badge background maps to the selected or named content node style.
- Ambiguous “background” requests should ask a clarification question before drafting.

## Images and logos

- Use the org logo if available and relevant.
- Always size images: include explicit width/maxWidth.
- Logos should usually be 48-140px wide depending on brand mark shape.
- Avoid unsized hero images, giant images above the fold, and layout-critical images without fallback text.

## Buttons and links

- Buttons need visible text, href, background, text color, padding, and radius.
- Default button style: strong solid fill, 14-16px text, 12-16px vertical padding, 18-24px horizontal padding.
- Avoid underlined button text unless the design intentionally uses a text link.
- Use text links for secondary actions only.

## When matching reference HTML

- Infer the visual system: colors, typography, density, border radius, card treatment, CTA style, footer tone.
- Do not copy raw DOM, CSS classes, full HTML documents, `<style>` blocks, or wrapper scaffolding.
- Translate reference sections into editable nodes.
- If reference HTML includes outer body/shell/card padding or background, map true page/outside styling to settings and true email surface styling to theme. Omit wrapper scaffolding from content unless it is visibly part of an editable section; do not create competing content wrappers.

## Failure checks before returning a draft

- Does the template still have editable, visible content?
- If the request was only a card/surface/page setup change, did you avoid adding content?
- Are runtime slots represented as explicit protected nodes/segments?
- Are section paddings intentional and not just imported shell padding?
- Did page/outside styling stay in settings and email surface styling stay in theme?
- Are images sized?
- Do buttons have visible labels?
- Are dividers direct `divider` nodes rather than wrapped in sections?
- Would a human be able to reproduce the design manually with current editor controls?
- Does the preview still work if generated AI body text is short or long?
