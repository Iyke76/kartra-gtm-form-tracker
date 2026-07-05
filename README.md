# Kartra GTM Form Tracker

Confirmation-render-based form tracking for Kartra opt-in forms in Google Tag Manager.

This standalone tracker fires only when a Kartra form shows an observable success signal, renders an error state, or redirects after a valid submit attempt. It is designed for GTM, GA4, server-side GTM, Meta CAPI, and Google Enhanced Conversions workflows where click-based form tracking is too noisy.

## Why this exists

Most form tracking is wired to a submit button click or a thank-you page view.

Click tracking counts validation failures, repeated clicks, and rage clicks. Thank-you-page tracking can count refreshes, direct visits, and bookmarked URLs. This script watches Kartra's actual form behavior so the dataLayer event is closer to a real form outcome.

## Files

| File | Use case |
| --- | --- |
| `kartra-form-tracker.js` | Raw JavaScript for GitHub, CDN, or custom hosting. Do not wrap this file in another `<script>` tag when loading by URL. |
| `kartra-form-tracker.gtm.html` | GTM Custom HTML paste-ready version. Includes the `<script>` wrapper. |

## Events

The tracker pushes these events to `window.dataLayer`:

```text
kartra_form_success
kartra_form_failed
ec_form_interaction
ec_tracker_initialized
```

### Success event

`kartra_form_success` fires when the script detects one of these success paths:

- Inline confirmation render
- Form removal or replacement after submit
- Page redirect via `pagehide` within the submit window
- Same-domain thank-you URL recall when recent stored form data exists

### Failure event

`kartra_form_failed` fires when Kartra renders an observable validation failure after the user has interacted with the form.

### Diagnostic events

`ec_form_interaction` fires once per form after the first real user input. It contains metadata only, not PII.

`ec_tracker_initialized` fires once per page load so you can confirm the script loaded even before a test submission.

## Payload example

```js
{
  event: "kartra_form_success",
  ec_event_id: "uuid-v4",
  ec_detection_method: "confirmation_render",
  ec_tracker_version: "3.0.1",
  ec_name: "Jane Doe",
  ec_email: "jane@example.com",
  ec_phone: "+15551234567",
  form_url: "https://example.com/landing-page",
  form_cta: "Get Access",
  form_platform: "kartra",
  form_position: "top",
  form_index: 1,
  form_id: "abc123",
  ec_raw_fields: {
    email: {
      value: "jane@example.com",
      type: "email",
      _bucket: "email"
    }
  }
}
```

When PII consent is denied, PII fields are removed:

```js
{
  event: "kartra_form_success",
  ec_event_id: "uuid-v4",
  ec_detection_method: "confirmation_render",
  ec_tracker_version: "3.0.1",
  form_url: "https://example.com/landing-page",
  form_platform: "kartra",
  form_position: "top",
  form_index: 1,
  form_id: "abc123",
  ec_pii_redacted: true
}
```

## Quick start in GTM

1. Open Google Tag Manager.
2. Create a new **Custom HTML** tag.
3. Paste the contents of `kartra-form-tracker.gtm.html`.
4. Trigger it on **All Pages - Page View** or **Consent Initialization** if your consent setup needs the tracker available early.
5. Preview the container.
6. Submit a real Kartra test form.
7. Confirm `kartra_form_success` or `kartra_form_failed` appears in the dataLayer.
8. Create GA4, server-side GTM, Meta CAPI, or Enhanced Conversions tags that listen for the events.

## Loading as raw JavaScript

If you host `kartra-form-tracker.js` yourself, load it like this:

```html
<script src="https://your-domain.example/kartra-form-tracker.js"></script>
```

Do not use `kartra-form-tracker.gtm.html` as a hosted JavaScript file. It is only for GTM Custom HTML paste-in use.

## Configuration

Define `window.EC_FORM_CONFIG` before loading the tracker:

```html
<script>
window.EC_FORM_CONFIG = {
  debug: false,
  hashPii: false,
  dedupe: true,
  allowRecoverySuccess: true,
  submitWindowMs: 15000,
  ttlMs: 600000,
  defaultCountryCode: "1",
  extraThankYouPatterns: []
};
</script>
```

| Option | Default | What it does |
| --- | --- | --- |
| `debug` | `false` | Enables `[EC]` console logs. Use in GTM Preview/testing only. |
| `hashPii` | `false` | Adds `ec_email_sha256` and `ec_phone_sha256` when possible. |
| `dedupe` | `true` | Suppresses duplicate success events for the same form/email/phone within 5 minutes. |
| `allowRecoverySuccess` | `true` | Allows a failed attempt followed by a corrected resubmit to fire both failure and success. |
| `submitWindowMs` | `15000` | Time window after submit click where render/redirect signals are considered related. |
| `ttlMs` | `600000` | Freshness window for stored form data used by same-domain thank-you recall. |
| `defaultCountryCode` | `"1"` | Country code used for best-effort phone normalization. Set `""` to disable. |
| `extraThankYouPatterns` | `[]` | Additional URL substrings that identify thank-you pages. |

## Consent

The tracker checks consent before firing events or writing storage.

Auto-detected consent systems:

- Google Consent Mode v2
- OneTrust
- Cookiebot

You can override consent with `window.EC_FORM_CONSENT_CHECK` before the tracker loads:

```html
<script>
window.EC_FORM_CONSENT_CHECK = function () {
  return {
    analytics: true,
    pii: false
  };
};
</script>
```

Consent behavior:

| Consent state | Behavior |
| --- | --- |
| `analytics: false` | No events fire and no storage is written. |
| `pii: false` | Events can fire, but raw PII and hashes are removed. Storage and recall data are redacted. |

If no CMP or override is detected, the tracker defaults to full consent. You are responsible for matching this behavior to your site and legal requirements.

## PII warning

The success and failure payloads can include:

- `ec_email`
- `ec_phone`
- `ec_name`
- `ec_raw_fields`

Do not send raw PII to GA4. GA4 terms prohibit PII in event parameters.

Recommended approaches:

- Use server-side GTM and hash values before forwarding to ad platforms.
- Enable `hashPii: true` and send only hashed fields where appropriate.
- Explicitly exclude raw PII fields from GA4 Event tags.
- Use raw fields only for first-party CRM workflows where you are allowed to process the data.

## Detection methods

Every success/failure event includes `ec_detection_method`.

Common success methods:

```text
confirmation_render
form_removed
stored_recall
pagehide
```

Common failure methods:

```text
error_class
error_render
```

`pagehide` is the weakest success method. It exists because Kartra often redirects after submission, but it can fire if someone clicks submit and leaves the page within the submit window. Segment or QA those events separately if you use them for paid-media optimization.

## Debugging

Console logs are disabled by default for privacy. To enable logs in GTM Preview:

```html
<script>
window.EC_FORM_CONFIG = {
  debug: true
};
</script>
```

You can also inspect tracker status in the browser console:

```js
window._ecFormTrackers.kartra.status()
```

The status output includes:

- Tracker version
- Consent state
- Forms attached
- Last event
- Secure-context status
- Runtime config

## Browser support

The script uses standard browser APIs including:

- `MutationObserver`
- `localStorage`
- `sessionStorage`
- `history.pushState`
- `crypto.randomUUID` with fallback
- `crypto.subtle` for optional hashing

Optional SHA-256 hashing requires a secure context, usually HTTPS.

## Known limitations

- `pagehide` is a fallback, not a perfect confirmation signal.
- Cross-domain redirects cannot use same-domain stored recall after navigation.
- Bot or spam submissions that pass Kartra validation can still count as success.
- Phone normalization is best-effort. Raw values remain in `ec_raw_fields` when PII consent allows them.
- Kartra markup may change. Always test on a real page before publishing GTM changes.

## Suggested GTM triggers

Success trigger:

```text
Event name equals kartra_form_success
```

Failure trigger:

```text
Event name equals kartra_form_failed
```

Interaction trigger:

```text
Event name equals ec_form_interaction
```

Diagnostic trigger:

```text
Event name equals ec_tracker_initialized
```

## Version

Current version: `3.0.1`

## License

MIT. See `LICENSE`.

## Author

Iyke Abel  
LinkedIn: https://www.linkedin.com/in/iykeabel/  
Email: abeliyke05@gmail.com
