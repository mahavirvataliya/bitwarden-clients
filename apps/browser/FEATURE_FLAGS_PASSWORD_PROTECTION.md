# Password Field Protection and Auto-Submit Feature Flags

This document describes the password field protection and auto-submit feature flags added to the Bitwarden browser extension.

## Features

### 1. Password Field Protection (`preventPasswordInspection`)

This feature prevents password fields from being exposed via browser DevTools inspect element functionality.

**How it works:**
- After Bitwarden autofills a password field, a MutationObserver is attached to monitor the field
- If someone attempts to change the field type from `password` to `text` (e.g., via DevTools), the observer detects it
- The password value is immediately cleared from the field
- Input and change events are triggered to notify the page

**Security benefit:**
- Prevents attackers from using browser DevTools to reveal autofilled passwords
- Protects against shoulder surfing where someone might try to inspect and reveal passwords
- Adds an extra layer of security for password-protected sites

### 2. Auto-Submit Control (`enableAutoSubmit`)

This feature allows control over the auto-submit functionality after autofill.

**How it works:**
- When enabled (default), forms are automatically submitted after autofill if the site supports it
- When disabled, forms are filled but not automatically submitted
- Users must manually click the submit button

## Feature Flag Configuration

### For Development

Both flags default to **enabled** (true) when not explicitly set.

To configure the flags during development, set them in your environment:

```javascript
// In your build configuration or environment
process.env.FLAGS = JSON.stringify({
  preventPasswordInspection: true,  // Enable password protection
  enableAutoSubmit: true             // Enable auto-submit
});
```

### For Production Builds

The flags can be set during the build process:

```bash
# Enable both features
FLAGS='{"preventPasswordInspection":true,"enableAutoSubmit":true}' npm run build:chrome

# Disable password protection
FLAGS='{"preventPasswordInspection":false,"enableAutoSubmit":true}' npm run build:chrome

# Disable auto-submit
FLAGS='{"preventPasswordInspection":true,"enableAutoSubmit":false}' npm run build:chrome
```

## Implementation Details

### Files Modified

1. **`libs/common/src/platform/misc/flags.ts`**
   - Added `preventPasswordInspection?: boolean` to `SharedFlags`
   - Added `enableAutoSubmit?: boolean` to `SharedFlags`

2. **`apps/browser/src/autofill/models/autofill-script.ts`**
   - Added feature flag properties to `AutofillScript` class

3. **`apps/browser/src/autofill/services/autofill.service.ts`**
   - Reads feature flags using `flagEnabled()` function
   - Sets flags on `AutofillScript` during fill script generation

4. **`apps/browser/src/autofill/services/insert-autofill-content.service.ts`**
   - Implements password field protection using MutationObserver
   - Monitors password fields after autofill
   - Clears value when type attribute changes

5. **`apps/browser/src/autofill/content/auto-submit-login.ts`**
   - Checks `enableAutoSubmit` flag before submitting forms
   - Respects user preference for auto-submit behavior

### Architecture

```
┌─────────────────────────────────────────────────────┐
│ Environment / Build Configuration                    │
│ FLAGS={ preventPasswordInspection: true, ... }      │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ autofill.service.ts                                 │
│ - Reads flags via flagEnabled()                     │
│ - Sets flags on AutofillScript                      │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────┐
│ Content Scripts                                      │
│ - insert-autofill-content.service.ts                │
│   ├─ Reads preventPasswordInspection from script    │
│   └─ Sets up MutationObserver on password fields    │
│ - auto-submit-login.ts                              │
│   ├─ Reads enableAutoSubmit from script             │
│   └─ Controls form submission behavior              │
└─────────────────────────────────────────────────────┘
```

## Testing

### Unit Tests

Tests are located in:
- `apps/browser/src/autofill/services/insert-autofill-content.service.spec.ts`
- `apps/browser/src/autofill/content/auto-submit-login.spec.ts`

Run tests:
```bash
npm test
```

### Manual Testing

1. Open `apps/browser/test-password-protection.html` in your browser
2. Install the Bitwarden extension with the feature flags enabled
3. Save credentials for the test page
4. Use Bitwarden to autofill the forms
5. Try to change password field types using DevTools
6. Verify that password values are cleared

### Testing Password Protection

1. Autofill a password field using Bitwarden
2. Open DevTools (F12)
3. Inspect the password field
4. Change `type="password"` to `type="text"`
5. **Expected:** Password value is immediately cleared
6. **Expected:** Input and change events are fired

### Testing Auto-Submit Control

1. Set `enableAutoSubmit: false` in FLAGS
2. Rebuild the extension
3. Autofill a login form
4. **Expected:** Form is filled but not submitted
5. User must manually click submit button

## Security Considerations

### Password Protection
- Only monitors fields that were autofilled by Bitwarden
- Does not prevent legitimate user interactions
- Clears password on any type change, not just to "text"
- Observer cleanup when field is removed from DOM

### Auto-Submit Control
- Respects user preference for submission behavior
- Defaults to enabled for backward compatibility
- Can be disabled for security-sensitive environments

## Future Enhancements

Potential improvements for future versions:

1. **UI Controls:** Add settings page for users to toggle flags
2. **Site-Specific Rules:** Allow per-site configuration
3. **Enhanced Detection:** Monitor additional attack vectors
4. **Telemetry:** Track protection activation frequency
5. **Grace Period:** Optional delay before clearing password

## Troubleshooting

### Password keeps getting cleared
- Ensure you're not legitimately changing the field type
- Check if other extensions are modifying the field
- Verify the feature flag is correctly set

### Auto-submit not working
- Check the `enableAutoSubmit` flag is set to true
- Verify the site supports auto-submit
- Check browser console for error messages

### Observer not activating
- Ensure Bitwarden is autofilling (not manual entry)
- Verify `preventPasswordInspection` is enabled
- Check that the field type is "password"

## Related Documentation

- [Feature Flags Documentation](../../libs/common/src/platform/misc/flags.ts)
- [Autofill Service Architecture](./src/autofill/README.md)
- [Content Scripts Guide](./src/autofill/content/README.md)
