# Email design patterns

Use this with `reference/template-design.md` when the user asks the Hermes Email Agent to create, redesign, or polish a reusable email template.

These are Hermes-native patterns. They are not raw HTML examples and they are not rigid templates. Choose a pattern route, adapt it to the user's prompt and brand context, then generate editable Hermes email IR using supported blocks.

## How To Use Patterns

1. Pick the route that best matches the user job.
2. Make the result feel distinct from other routes. A welcome email, follow-up, announcement, digest, and founder note should not share the same skeleton.
3. Keep the message body area native. Style the parent section so generated body copy inherits font, color, line-height, and paragraph rhythm.
4. Preserve editability. Use sections, headings, paragraphs, buttons, dividers, spacers, images with explicit size, `ai_content`, and richText unsubscribe segments.
5. If the reference asks for a complex visual, produce an editable approximation and explain what was simplified for email safety.

## Premium Welcome

Use for onboarding, first touch, account created, or workspace created emails.

Structure:

1. Compact brand/header row.
2. Strong welcome headline.
3. Short expectation-setting intro.
4. Message body area after the intro.
5. One first-step CTA.
6. Quiet footer with inline unsubscribe.

Design recipe:

- Tone: warm, competent, restrained.
- H1: 30-34px, 1.12-1.2 line height, strong weight.
- Body: 15-17px, 1.55-1.7 line height.
- Footer: 12-14px, muted, 1.45-1.6 line height.
- Horizontal section padding: 32-40px.
- Hero top padding: 32-44px.
- Intro to message body: 24-32px.
- CTA: outcome label such as "Start setup", "Open workspace", or "View dashboard"; solid fill; 14-16px vertical padding; 20-24px horizontal padding; 8-12px radius.

Avoid:

- Generic fixed copy such as "we help folks do stuff".
- Body slot as a decorative placeholder card.
- CTA too close to body text.
- Browser-default-looking footer links.
- Centered body paragraphs unless explicitly requested.

## Follow-Up Letter

Use for lifecycle nudges, customer-success check-ins, incomplete setup reminders, and sales follow-ups.

Structure:

1. Minimal brand line or small header.
2. Letter-like opener with short context.
3. Message body area near the top.
4. Optional low-emphasis reminder/note box.
5. Restrained CTA or text link.
6. Quiet footer.

Design recipe:

- Tone: direct, personal, low-pressure.
- Title: optional, 22-28px.
- Body: 16px, 1.55-1.7 line height.
- Horizontal section padding: 32-40px.
- Use fewer decorative sections than welcome or announcement.
- CTA should appear after the personalized reason is clear.

Avoid:

- Over-designed campaign-card layout.
- CTA before relevance is established.
- Footer that is too loud for a personal note.
- Runtime body style that differs from fixed letter copy.

## Product Announcement

Use for feature launches, product updates, release notes, and change announcements.

Structure:

1. Header with brand mark or text.
2. Optional badge/eyebrow such as "New" or "Update".
3. Benefit-led headline.
4. Short intro.
5. Feature/detail section with bullets, table, or soft cards.
6. Message body area for personalized angle.
7. Primary CTA.
8. Footer.

Design recipe:

- Tone: confident, clear, useful.
- H1: 30-36px, 1.1-1.2 line height.
- Body: 15-17px, 1.5-1.65 line height.
- Feature titles: 16-18px, strong weight.
- Horizontal padding: 32-40px.
- Feature rhythm: 16-24px between rows/cards.
- CTA labels: "Explore the update", "See what's new", "Open feature".

Avoid:

- Same structure as welcome with different words.
- Too much fixed copy before the message body.
- Unsized hero images.
- Generic CTA labels for every feature.

## Signal Insight Card

Use when Hermes found a meaningful customer signal, metric, account event, or insight.

Structure:

1. Header or compact brand row.
2. Badge/eyebrow naming the signal.
3. Headline interpreting the signal.
4. Soft bordered insight card with metric, observation, or bullets.
5. Message body area for personalized explanation.
6. CTA or reply prompt.
7. Footer.

Design recipe:

- Tone: useful, analytical, calm.
- H1: 28-34px, 1.15-1.25 line height.
- Metric: 24-32px only if there is a real number.
- Body: 15-16px, 1.55-1.65 line height.
- Insight card padding: 20-28px.
- Section padding: 32-40px horizontal.
- CTA labels: "Review account", "View activity", "Open report".

Avoid:

- Decorative signal cards with no useful signal.
- Too many competing metric cards.
- Message body disconnected from the insight.

## Digest Update

Use for newsletters, weekly digests, recurring updates, and multi-item summaries.

Structure:

1. Header with brand and optional date/edition.
2. Compact headline and intro.
3. Repeated update sections or cards with short titles and summaries.
4. Message body area for personalized intro or recommendation.
5. Optional CTA after the lead item.
6. Footer.

Design recipe:

- Tone: organized, editorial, scan-friendly.
- H1: 28-34px.
- Section titles: 17-20px.
- Body summaries: 14-16px, 1.5-1.65 line height.
- Repeated item spacing: 18-24px.
- Use dividers or subtle backgrounds for separation.
- Prefer one primary CTA or several low-emphasis text links, not many identical filled buttons.

Avoid:

- Turning every item into an oversized card.
- No clear lead item.
- Awkward message body placement in a dense layout.

## Founder Note

Use for executive notes, personal announcements, and team notes where restraint is the premium move.

Structure:

1. Minimal brand or sender line.
2. Optional short title.
3. Message body area near the top or after a one-line opener.
4. Short fixed close or signature.
5. Low-emphasis CTA if needed.
6. Quiet footer.

Design recipe:

- Tone: human, concise, credible, plain.
- Title: optional, 22-28px.
- Body: 16px, 1.6-1.75 line height.
- Horizontal padding: 36-44px.
- Avoid heavy backgrounds and card stacks.
- Use text links or restrained buttons only when there is a concrete next step.

Avoid:

- Marketing-like hero treatment.
- Too many visual sections.
- Fake executive voice.
- Runtime body that does not blend with fixed opener/close.

## Pattern Evaluation

When checking a generated template, compare it against the selected pattern:

- Does the structure match the user job?
- Does it look distinct from other pattern routes?
- Is hierarchy stronger than a generic header/body/button/footer shell?
- Does the message body area feel native?
- Does the CTA have enough padding, contrast, and contextual label?
- Does the footer look intentional and compliant?
- Would short and long generated body copy both look good?
- Can a human reproduce and edit the design with Hermes editor controls?
