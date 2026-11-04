# SmartApply RW bug review

Review date: 4 November 2026. Scope: the local project in `SmartApply-RW`, including all application HTML, JavaScript, CSS, image assets, the development server, and build/deployment configuration.

**Original review: six issues identified — three medium priority and three low priority. All six fixes were implemented locally on 4 November 2026.** No critical or high priority issue was found in the completed checks. Browser verification remains outstanding; this does not establish that the app is free of other bugs.

The visitor flow is: browse university guidance in Arabic or English → select a service/university or complete the enquiry form → open a prepared WhatsApp conversation. There is no application backend, database, account system, or payment flow in this project.

## Fix status

| Finding | Change | Verification |
| --- | --- | --- |
| 1. Spaces-only enquiry fields | Reject whitespace-only names/messages with Arabic/English native validation messages; clear errors on input; trim outgoing values. | Focused DOM tests pass in both languages for spaces, tabs/newlines, Unicode whitespace, correction, language switching, valid messages, and optional fields. |
| 2. Insufficient text contrast | Use a darker secondary text color in the contact section and review disclaimer. | Calculated contrast is now 4.89:1 on the contact background, 5.04:1 on the review background, and 5.48:1 on the white form background. |
| 3. Unnamed credits dialog | Connect the dialog to its existing translated heading through `aria-labelledby`. | DOM checks confirm the heading exists inside the dialog and supplies the expected title in both languages. |
| 4. FAQ closes on language change | Preserve expanded question indices when translating the FAQ. | Repeated language switches preserve both expanded and collapsed questions while translating their text. |
| 5. Generic welcome-package enquiry | Supply package-specific WhatsApp messages in both languages. | Decoded link text includes the selected package context and retains the correct destination number. |
| 6. Untranslated metadata/accessible labels | Translate the description and language/menu/review/close button labels, including saved-language startup. | DOM checks pass in both languages and on initial load with a saved English preference. |

After the fixes, `npm run check`, `npm run build`, all 16 DOM regression groups, and the focused bilingual form tests passed. The remaining browser checks listed below still apply. Changes have not been deployed, and no WhatsApp messages were sent.

## Original findings and reproduction evidence

### 1. Medium: required enquiry fields accept spaces

Location: [contact form markup](dist/index.html#L13), [submit handler](dist/app.js#L21).

Reproduction:

1. Enter three spaces in both the required name and message fields.
2. Submit the form.

The form passes HTML validation and the handler prepares a WhatsApp message with an effectively empty name and message. This was reproduced in the DOM test: `form.checkValidity()` returned `true`, and the intercepted `window.open` received the blank enquiry. No message was sent.

Expected: required fields contain meaningful text before WhatsApp opens.

Suggested fix: trim field values, reject required values that become empty, and show a localized validation message at the relevant field.

### 2. Medium: some informational text has insufficient contrast

Location: [base color, contact, and review styles](dist/style.css#L1).

Calculated from the explicit foreground/background colors in the stylesheet:

| Text | Foreground | Background | Contrast |
| --- | --- | --- | --- |
| Contact introduction, contact labels, and contact fine print outside the white form | `#64778a` | `#eaf3fc` | 4.12:1 |
| Disclaimer explaining that reviews are illustrative | `#60768c` | `#f1f6fa` | 4.32:1 |

These are normal-size text and fall below the 4.5:1 minimum in [WCAG 2.2 contrast guidance](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html). The calculation uses stylesheet colors; it is not a screenshot or browser accessibility scan.

Impact: the enquiry information and illustrative-review disclaimer are harder to read for visitors with reduced contrast sensitivity.

Suggested fix: darken the affected text colors enough to reach at least 4.5:1 on their respective backgrounds, then confirm the computed styles in a browser.

### 3. Medium: the image-credits dialog has no accessible name

Location: [credits dialog markup](dist/index.html#L14).

The native `<dialog id="credits">` has neither `aria-label` nor `aria-labelledby`. Its visible heading has no ID connecting it to the dialog.

Impact: assistive technology lacks a programmatically assigned title for the dialog. The visible heading alone does not establish that relationship. This is a markup finding; actual screen-reader announcements were not tested.

Expected: opening the dialog exposes the translated “Image credits” title as its accessible name, following the [W3C modal-dialog labeling guidance](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/).

Suggested fix: give the existing translated heading an ID and reference it using the dialog's `aria-labelledby` attribute. Retain the native dialog behavior.

### 4. Low: changing languages closes an open FAQ

Location: [FAQ rendering](dist/app.js#L20), [language-switch handler](dist/app.js#L21).

Reproduction:

1. Expand an FAQ answer.
2. Switch languages.

`render()` replaces the complete FAQ markup. The previously expanded question is recreated without its `open` attribute. The DOM test confirmed `open === false` after switching.

Impact: a visitor trying to translate the answer they are reading must find and reopen it.

Suggested fix: retain expanded question indices or stable IDs across rendering, or translate existing FAQ elements without replacing them.

### 5. Low: the welcome-package enquiry loses the selected context

Location: [arrival/welcome-package CTA](dist/index.html#L9), [WhatsApp URL and link binding](dist/app.js#L17).

Reproduction:

1. Use “Ask about the welcome package” in the arrival section.
2. Inspect the prepared WhatsApp message.

The link has no `data-message`, so it receives the generic message about studying in Rwanda, services, and pricing. It does not mention the welcome package. This was reproduced by decoding the generated link. The corresponding welcome-package service-card link does include service context.

Expected: the arrival CTA prepares an enquiry specifically about the welcome package in the selected language.

Suggested fix: bind a localized package-specific message to this CTA and refresh it when the language changes.

### 6. Low: language switching leaves metadata and some accessible labels untranslated

Location: [description metadata](dist/index.html#L1), [language button](dist/index.html#L3), [credits close button](dist/index.html#L14), [translation renderer](dist/app.js#L20).

Reproduction and evidence:

- Switch to English: `html.lang`, direction, title, and visible translated content change, but the meta description remains Arabic.
- In Arabic mode, the language button's accessible name remains `Switch language`, and the dialog close button remains `Close`.

The renderer translates elements carrying `data-ar`/`data-en` and explicitly updates the title and language state; it does not translate these metadata or accessibility attributes.

Impact: the selected language is applied inconsistently, especially for assistive-technology users. This is not a claim about how a particular search engine will display a snippet.

Suggested fix: update the meta description and applicable accessible labels in the same language-rendering operation. Include the target language in the language-switch label.

## Checks that passed

`npm run check` and `npm run build` both exited successfully. Both commands currently run only `node --check dist/app.js`; they do not check HTML, layout, links, or interaction behavior.

Eleven temporary DOM test groups passed against the actual, unmodified `dist/app.js`:

1. Default Arabic language, RTL direction, and page title.
2. Rendering of six services, six universities, four steps, three review cards, and six FAQs.
3. Existence of all referenced local image, script, and stylesheet resources after rendering.
4. Resolution of every internal hash link to an existing element.
5. Correct destination number and URL properties for all generated WhatsApp anchors.
6. Correct contextual messages for service-card and university-card WhatsApp links.
7. Review next/previous behavior and wrapping.
8. Menu toggling, `aria-expanded` updates, and closing after selecting a navigation link.
9. Language switching, English content rendering, direction changes, and persistence.
10. Saved English preference on startup and usable language switching when local storage throws.
11. Form message encoding of Arabic, punctuation, and newlines with the correct WhatsApp destination.

Additional static checks found no duplicate HTML IDs, missing paired `data-ar`/`data-en` translations, or unreadable JPEG assets. The configured business phone and email match `AGENTS.md`. Demonstration reviews are explicitly identified in both languages.

External link spot checks returned page content for all six university destinations:

- [UNILAK](https://site.unilak.ac.rw/)
- [ALU](https://www.alueducation.com/)
- [University of Rwanda](https://ur.ac.rw/)
- [Mount Kigali University](https://mkur.ac.rw/)
- [ULK](https://ulk.ac.rw/)
- [INES–Ruhengeri](https://www.ines.ac.rw/)

The photo-credit destinations also returned content: UNILAK above, [ALU campus information](https://www.alueducation.com/home/study-at-alu/), [UR College of Education](https://ce.ur.ac.rw/), [The New Times article](https://www.newtimes.co.rw/article/7311/news/education/featured-mount-kenya-university-rwanda-to-participate-in-hec-exhibition), [ULK campuses](https://ulk.ac.rw/our-campuses/), [INES intercultural-day article](https://www.ines.ac.rw/news/ines-ruhengeri-celebrates-5th-intercultural-day-in-a-spectacular-display-of-unity-in-diversity/), and [UNILAK Rwamagana campus](https://site.unilak.ac.rw/rwamagana-campus/). These are retrieval checks, not a guarantee of availability on every visitor's network or an image-rights review.

## Improvements outside the six findings

- The mobile menu closes through its toggle or a selected navigation link. It does not close on Escape or an outside click. Adding those behaviors and `aria-controls="nav"` would improve usability; their absence does not make an ordinary navigation disclosure unusable.
- There is no skip-to-content link. Adding one would reduce repeated keyboard navigation.
- Most visible text is generated from empty translation elements by JavaScript. With JavaScript unavailable, the main heading, navigation labels, and service/university content are empty, with no `noscript` explanation. Static initial text would improve resilience.
- Add interaction checks beyond JavaScript syntax to the project's validation workflow.

## Verification limits and remaining work

The environment rejected the local server's bind to `0.0.0.0:3077` with `EPERM`. Chrome also exited during launch with an operation-not-permitted error. These are environment restrictions, not confirmed application bugs. No permission override or deployment was attempted.

As a result, the following remain unverified:

- Desktop and mobile layout, clipping, spacing, horizontal overflow, and browser zoom.
- Actual image decoding/rendering in a browser and external font loading.
- Native FAQ keyboard interaction and dialog opening, closing, focus trapping, and focus return.
- Real browser validation UI and WhatsApp handoff on desktop/mobile devices.
- Screen-reader behavior, full accessibility coverage, and measured performance.
- HTTP behavior of the local server, production hosting, and deployment configuration in a live Vercel project.

The interaction checks used jsdom in a temporary directory. jsdom executes DOM logic but has no layout engine and cannot replace browser verification. All WhatsApp window-opening calls were intercepted; no enquiries or other messages were sent. The initial review changed no application source files. The subsequent fixes changed `dist/app.js`, `dist/index.html`, and `dist/style.css`, as recorded in the fix-status table above.
