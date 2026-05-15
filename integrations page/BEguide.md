# Integrations: backend handoff

This document describes what the current frontend expects from the backend for the `Integrations` area.

Source files used for this handoff:
- `src/pages/integrations/Integrations.tsx`
- `src/pages/integrations/index.tsx`
- `src/pages/integrations/types.ts`
- `src/services/integrations.ts`
- `src/services/properties.ts`

## 1. High-level behavior

The frontend has 2 macro states:

1. Onboarding / first setup
   - No active PMS connection exists.
   - The user picks a PMS, validates a token, maps PMS properties to Alfred properties, and confirms the connection.

2. Returning / post-onboarding
   - At least one active PMS connection exists.
   - The user sees connection status, can sync, re-authenticate, remap properties, or switch PMS.

Important frontend assumptions:

- All integration requests are authenticated with the same Bearer JWT used by the rest of the app.
- The UI is built around a single active PMS connection per account.
- If `GET /integrations/connections` returns more than one connection, the UI shows only the first one.
- Existing Alfred properties are loaded from the existing properties backend, not from the integrations backend.

## 2. Base URL and auth

Frontend base URL:

- `VITE_BACKEND_INTEGRATIONS_URL`

All requests are sent through the shared Axios instance, which automatically adds:

- `Authorization: Bearer <access_jwt_token>`

If the access token expires and the backend returns `401`, the frontend will try to refresh and replay the request.

## 3. PMS ids the frontend can send

Unless restricted manually later, the frontend exposes these PMS ids:

- `apaleo`
- `beds24`
- `bookingsync`
- `channex`
- `guesty`
- `guesty4hosts`
- `hostaway`
- `hostfully`
- `igms`
- `lodgify`
- `mews`
- `octorate`
- `ownerrez`
- `rentalsunited`
- `rms`
- `smoobu`
- `tokeet`
- `teamsystem`
- `magellano`

If the backend supports only a subset, frontend and backend must be aligned before release.

## 4. Existing non-integrations dependency

Before the mapping screen is shown, the page loads Alfred properties through the existing properties service:

- `GET ${VITE_BACKEND_PROPERTIES_URL}get_properties?current_page=1`

Current frontend expectation for that response:

```json
{
  "data": {
    "properties": [
      {
        "property_id": "uuid",
        "name": "Loft Trastevere"
      }
    ]
  }
}
```

Those properties populate the "In Alfred" select in the mapping screen.

## 5. Screen-by-screen flow

### Screen 0: page bootstrap

When `/dashboard/integrations` loads, the frontend immediately calls:

- `GET /integrations/connections`

Expected response body:

```json
[]
```

or

```json
[
  {
    "pmsId": "hostaway",
    "status": "connected",
    "lastSync": "2026-05-15T08:10:00.000Z",
    "mapped": 12,
    "total": 14
  }
]
```

Field notes:

- `pmsId` must match one of the PMS ids listed above.
- `status` must be one of `connected`, `syncing`, `error`.
- `lastSync` should preferably be an ISO string parseable by `new Date(...)`.
- `mapped` and `total` are used to show the synced/unmapped counters.

Routing logic:

- `[]` means first-time user -> show onboarding picker.
- non-empty array means returning user -> show returning screen.

Important:

- The frontend expects a direct array, not `{ data: [...] }`.
- A failure here is not handled gracefully by the UI. Prefer returning `200` with `[]` when there is no connection.

### Screen 1: PMS picker

No backend call is made here.

User action:

- User selects a PMS card.
- Frontend stores the chosen `pmsId` and moves to token validation.

### Screen 2: token validation

When the user clicks "Verifica", the frontend calls:

- `POST /integrations/:pmsId/validate`

Request body:

```json
{
  "token": "the-pms-access-token"
}
```

Expected response body:

```json
{
  "valid": true
}
```

Behavior:

- `valid: true` enables the "Connect" button.
- `valid: false` keeps the user on the same screen and shows an invalid state.
- Any rejected request is also treated as invalid by the UI.

Recommendation:

- Return `200` with `{ "valid": false }` for business invalidation.
- Reserve `4xx/5xx` for real server/network failures.

### Screen 3: property discovery

When the user clicks "Connetti <PMS>", the frontend calls:

- `POST /integrations/:pmsId/discover`

Request body during initial onboarding:

```json
{
  "token": "the-pms-access-token"
}
```

Expected response body:

```json
{
  "properties": [
    {
      "id": "pms-property-1",
      "name": "Rome Loft Navona",
      "beds": 4,
      "suggested": "alfred-property-uuid"
    },
    {
      "id": "pms-property-2",
      "name": "Milan Suite Duomo",
      "suggested": null
    }
  ]
}
```

Field notes:

- `id` is the PMS-side property identifier and is used as the key in the later mapping payload.
- `name` is shown in the UI.
- `beds` is currently optional and not rendered, but safe to include.
- `suggested` is optional. If present, the frontend preselects that Alfred property and marks the row as auto-matched.

Important:

- The `suggested` value must be an Alfred property id already present in the `get_properties` response.
- If you send an auto-matched row and the user does not change it, the later mapping payload may include `"auto": true`. The backend should ignore that field if it is not needed.

### Screen 4: property mapping save

After discovery, the user maps each PMS property to one of 3 states:

1. Skip
2. Link to existing Alfred property
3. Create a new Alfred property

When the user clicks "Salva mappatura", the frontend calls:

- `POST /integrations/:pmsId/connect`

Request body example:

```json
{
  "token": "the-pms-access-token",
  "mapping": {
    "pms-property-1": {
      "kind": "existing",
      "alfredId": "alfred-property-uuid",
      "auto": true
    },
    "pms-property-2": {
      "kind": "create",
      "newName": "Milan Suite Duomo"
    },
    "pms-property-3": {
      "kind": "skip"
    }
  }
}
```

Expected frontend outcome:

- Any `2xx` response is considered success.
- Any rejected request opens the generic "mapping failed" modal.

Backend responsibilities for this endpoint:

- Persist the PMS connection for the current Alfred account/user.
- Persist or update the stored PMS token when a non-empty token is provided.
- Save the mapping between PMS properties and Alfred properties.
- For rows with `kind: "create"`, create Alfred properties server-side.
- Make sure newly created properties are compatible with the existing Alfred property model and appear later in `get_properties`.
- Optionally trigger the initial data import/sync after mapping is saved.

Strong recommendation:

- Treat this endpoint as idempotent for the same account + PMS.
- Make the whole operation transactional if possible, especially when property creation and mapping persistence happen together.

### Screen 5: success

No backend call is made here.

User actions:

- "Torna alle integrazioni"
- "Apri Calendario" -> frontend redirects to `/dashboard/calendar`

Backend implication:

- If the connection triggers property creation or booking import, that data should already be available to the existing property/calendar APIs after success.

## 6. Returning view: post-onboarding behavior

The returning screen is driven by the same bootstrap call:

- `GET /integrations/connections`

The screen shows only the first returned connection.

### Action: sync now

Frontend call:

- `POST /integrations/:pmsId/sync`

Request body:

- none

Expected response:

- Any `2xx` response with no required body.

Frontend behavior:

- Shows a local "syncing" state while the request is pending.
- As soon as the request resolves, shows "just now" as last sync.

Important implication:

- The current UI behaves as if the sync is completed when the request resolves.
- If the backend only enqueues an async job and returns immediately, the UI will still look like the sync already happened.

Recommended backend behavior:

- Either complete the sync before returning, or align later with a frontend change that introduces polling/job status.

### Action: re-authentication

Step 1:

- `POST /integrations/:pmsId/validate`

Step 2, only if valid:

- `PATCH /integrations/:pmsId/token`

Request body:

```json
{
  "token": "new-pms-token"
}
```

Expected response:

- Any `2xx` response with no required body.

Backend responsibility:

- Replace the stored token without deleting mappings, sync history, or imported data.

### Action: remap properties

The frontend reuses the same discovery and save endpoints used during onboarding.

Step 1:

- `POST /integrations/:pmsId/discover`

Request body during remap:

```json
{
  "token": ""
}
```

Step 2:

- `POST /integrations/:pmsId/connect`

Request body during remap:

```json
{
  "token": "",
  "mapping": {
    "pms-property-1": {
      "kind": "existing",
      "alfredId": "alfred-property-uuid"
    }
  }
}
```

This is a very important frontend assumption:

- During remap, the frontend sends an empty token.
- The backend must therefore use the token already stored for that PMS connection.

If the backend requires a token on every remap request, the current frontend will not work.

### Action: disconnect / switch PMS

Planned service endpoint:

- `DELETE /integrations/:pmsId`

Expected semantic behavior from the UI copy:

- Remove the active PMS connection
- Remove mappings
- Remove sync history
- Remove imported bookings/data related to that PMS

Important current frontend gap:

- The service method exists, but the current UI does not actually call `DELETE` yet.
- Today, when the user confirms "Disconnetti e cambia", the UI only returns to the picker screen.

What this means for backend integration:

Option A:

- Patch the frontend later so it really calls `DELETE /integrations/:pmsId` before switching.

Option B:

- Make `POST /integrations/:newPmsId/connect` replace any previous active PMS connection atomically on the backend.

Given the current UX copy, one of these 2 behaviors must exist before release.

## 7. Endpoint summary

### Status matrix: already specified vs to create

| Method | Path | Frontend status | Backend action | Notes |
| --- | --- | --- | --- | --- |
| `GET` | `/integrations/connections` | Already called by the page on load | Must exist | Drives first-time vs returning flow |
| `POST` | `/integrations/:pmsId/validate` | Already called | Must exist | Used in onboarding and re-auth |
| `POST` | `/integrations/:pmsId/discover` | Already called | Must exist | Used in onboarding and remap |
| `POST` | `/integrations/:pmsId/connect` | Already called | Must exist | Saves connection + mapping, and should handle `create` rows |
| `PATCH` | `/integrations/:pmsId/token` | Already called | Must exist | Used after successful re-validation |
| `POST` | `/integrations/:pmsId/sync` | Already called | Must exist | FE assumes sync is done when request resolves |
| `DELETE` | `/integrations/:pmsId` | Specified in service, but not actually triggered by the current UI flow | Should exist before release, or the backend must replace old PMS on next `connect` | Current switch-PMS flow has a frontend gap |
| `GET` | `/get_properties?current_page=1` | Already called by existing FE | Must keep existing behavior | Not an integrations endpoint, but required for mapping |

So, in short:

- The frontend already specifies all the integrations endpoints it expects.
- The backend still has to implement those endpoints if they do not exist yet.
- There is no additional mandatory integrations endpoint beyond the list above.
- The only partial case is `DELETE /integrations/:pmsId`: it is already specified in code, but the current UI does not invoke it yet.

### Already expected by the current frontend

| Method | Path | Used in |
| --- | --- | --- |
| `GET` | `/integrations/connections` | page bootstrap, returning screen |
| `POST` | `/integrations/:pmsId/validate` | token validation, re-auth |
| `POST` | `/integrations/:pmsId/discover` | initial connect, remap |
| `POST` | `/integrations/:pmsId/connect` | initial save, remap save |
| `PATCH` | `/integrations/:pmsId/token` | re-auth |
| `POST` | `/integrations/:pmsId/sync` | sync now |
| `DELETE` | `/integrations/:pmsId` | intended for disconnect, not yet wired in UI |

### Existing non-integrations dependency

| Method | Path | Used in |
| --- | --- | --- |
| `GET` | `/get_properties?current_page=1` | mapping dropdown, calendar property dropdown |

## 8. Button and sensitive-action matrix

This section is the most operational one for the backend developer: button by button, it tells what the frontend does and what the backend must support.

| Screen | Sensitive element | Current frontend behavior | Endpoint involved | What backend must do |
| --- | --- | --- | --- | --- |
| Bootstrap | Page open `/dashboard/integrations` | Immediately loads current connections | `GET /integrations/connections` | Return `[]` for first-time users, or the current active connection list |
| Picker | PMS card click | Saves selected PMS locally and moves to token screen | none | Nothing backend-side |
| Picker | "Richiedi un'integrazione" link | Currently placeholder only (`href="#"`) | none | Nothing yet; no backend integration currently tied to this control |
| Token screen | "Indietro" | Local navigation only | none | Nothing backend-side |
| Token screen | "Verifica" | Calls token validation | `POST /integrations/:pmsId/validate` | Verify token and return `{ valid: boolean }` |
| Token screen | "Connetti {PMS}" | Calls property discovery after successful validation | `POST /integrations/:pmsId/discover` | Use provided token, fetch PMS properties, optionally auto-match them to Alfred properties |
| Mapping screen | Property select row | Updates mapping only in frontend state | none | Nothing immediately; final state is sent on save |
| Mapping screen | New property name input | Updates mapping only in frontend state | none | Nothing immediately; final state is sent on save |
| Mapping screen | "Indietro" | Local navigation back to token screen | none | Nothing backend-side |
| Mapping screen | "Salva mappatura" | Sends the complete mapping object | `POST /integrations/:pmsId/connect` | Persist connection, persist/update token if present, save mapping, create Alfred properties for `kind: "create"` rows, optionally trigger first sync |
| Success screen | "Torna alle integrazioni" | Local navigation only | none | Nothing backend-side |
| Success screen | "Apri Calendario" | Navigates to `/dashboard/calendar` | none | Data imported by the integration should already be readable by existing calendar/property APIs |
| Returning screen | "Gestisci" menu button | Opens local dropdown menu | none | Nothing backend-side |
| Returning screen | "Sincronizza ora" | Calls sync and shows local loading state | `POST /integrations/:pmsId/sync` | Run or trigger sync; current FE assumes the sync is effectively done when the request resolves |
| Returning screen | "Ri-autenticazione" open | Opens modal locally | none | Nothing backend-side |
| Reauth modal | "Aggiorna token" | First validates, then updates token | `POST /integrations/:pmsId/validate` then `PATCH /integrations/:pmsId/token` | Validate new token, then replace stored token without touching mappings/history |
| Returning screen | "Modifica mappatura proprietà" | Reopens discovery/mapping flow using stored connection | `POST /integrations/:pmsId/discover` then `POST /integrations/:pmsId/connect` | Support remap flow even when FE sends `token: ""`, using stored token server-side |
| Returning screen | "Disconnetti" menu item | Opens local switch warning modal | none directly | Nothing yet at menu-open time |
| Switch warning modal | "Disconnetti e cambia" | Currently only returns user to picker; does not call backend delete yet | intended `DELETE /integrations/:pmsId`, but not actually called today | Before release, either implement FE delete call or make next `connect` replace the previous PMS atomically |
| Returning screen | Status cards / last sync info | Purely presentational | driven by `GET /integrations/connections` | Keep `status`, `lastSync`, `mapped`, `total` coherent so the cards render correctly |

In practical terms:

- Every sensitive action that actually hits the backend is already listed above.
- Controls not associated with an endpoint are local-only UI controls.
- The only intentionally incomplete sensitive flow today is disconnect/switch PMS.

## 9. Recommended backend checklist

- Enforce or support only one active PMS connection per Alfred account.
- Return a direct array from `GET /integrations/connections`.
- Support `discover` and `connect` both with a fresh token and with an empty token during remap.
- Ignore optional frontend-only fields such as `auto` inside mapping rows.
- Create Alfred properties server-side for `kind: "create"` mappings.
- Make sure new properties appear in the existing properties endpoints used by the app.
- Decide clearly how switching PMS works: either real `DELETE` before reconnect, or automatic replacement on the next `connect`.
- Decide whether `sync` is synchronous or async job based, because the current UI assumes "done when request resolves".

## 10. Open points worth aligning before backend work starts

- Should switching PMS delete old imported bookings immediately, or archive them first?
- Should `connect` trigger a full initial sync immediately, or only save the mapping?
- Should `discover` return only active/listable properties, or also archived/inactive ones?
- If `discover` returns zero properties, do we want the backend to allow that, or should the frontend get a dedicated empty-state response later?
- Do we want to expose all PMS ids already present in the picker, or temporarily limit the supported set?

## 11. Current frontend caveats to keep in mind

- After the first successful connection, the success screen does not refresh `GET /integrations/connections`. If the user clicks "Torna alle integrazioni" without reloading the page, they can land back on the picker instead of the returning screen.
- After `sync` and `reauth`, the frontend does not refetch connection status. It only updates local UI state.
- The disconnect service exists, but the current switch-PMS confirmation does not call it yet.
