# Outlook OAuth Integration — Status Notes

## Current State
OAuth flow works end-to-end (PKCE, token exchange, redirect) but Microsoft 365
work accounts require an admin to grant consent for the `Calendars.Read` permission
before users can connect.

## What's Built
- Full OAuth 2.0 PKCE flow via Microsoft Graph API
- Ephemeral token stored in localStorage (with refresh token)
- Events fetched from `/me/calendarView` and overlaid on weekly view
- Toggle button in cal nav bar shows/hides overlay
- Tenant ID field in modal for work accounts (uses tenant-specific endpoint)
- Error messages from Microsoft surfaced as toasts

## Blocker: Admin Consent Required
Work accounts (Microsoft 365 / Azure AD) require an admin to grant consent for
`Calendars.Read` on behalf of the organization before individual users can
authorize the app.

### How to Fix (IT Admin — 5 minutes)
1. Azure Portal → Enterprise applications → find "VoiceMap" (or the Client ID)
2. Permissions → Grant admin consent for [organization]
3. Confirm — after this, users can connect without admin prompt

### Alternative: Request Individual Consent
In Azure Portal → App registration → API permissions → `Calendars.Read` →
click "Grant admin consent". Requires Global Admin or Application Admin role.

## Azure App Settings That Must Match
- Platform: Single-page application (SPA)
- Redirect URI: exact URL of index.html as served
- Tenant ID: Directory (tenant) ID from app Overview page
- Supported account types: "Accounts in this organizational directory only"
- Permissions: Microsoft Graph → Delegated → Calendars.Read

## Code Location
All Outlook code is in `index.html` under:
`// ─── OUTLOOK CALENDAR ────────────────────────────────────────────────────────`

Key functions: `_olConnect`, `_olHandleRedirect`, `_olFetchEvents`, `_olRefreshToken`
Key constants: `_OL_TOKEN_KEY`, `_OL_CLIENT_KEY`, `_OL_TENANT_KEY`
Modal HTML: `id="outlookModal"`
