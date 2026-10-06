# Device Verification Page Customization

Bring `/device` in line with how `/authorize` is customized, so the device code flow can ship as a
PR without being the one page consumers cannot brand.

## Problem

The login page and the device page are customized in different ways:

| | Login (`/authorize`) | Device (`/device`) |
|---|---|---|
| Template source | `/config/login.html`, then `templates/default.html`, then inline | inline string only |
| Shipped styled template | `templates/default.html` | none (styling lives in the Go const) |
| Helm override | `customLogin.enabled` / `customLogin.html` | none |
| Branding used | `ServiceName`, `LogoURL`, `PrimaryColor`, `Title`, `Subtitle` | `ServiceName`, `PrimaryColor` |
| Dev-mode affordances | `123456` notice and prefill, quick sign-in | none |
| README | "Custom Login Template" section | none |

A consumer with a custom login page (for example a light, image-backed brand page) gets sent from
their branded login to a generic dark panel on `/device`, and has no way to change it.

There is also a structural bug: `New` returns early through `NewFromFile` as soon as a login file
exists, and `NewFromFile` always hardwires the inline device template. The two templates must
resolve independently.

## Goals

- `/device` resolves its template the same way login does: mounted file, then shipped file, then
  inline fallback.
- A styled `templates/device.html` ships in the image, visually consistent with
  `templates/default.html`.
- The chart can mount a custom device template the same way it mounts a custom login template.
- The template data contract is documented in the README.

## Non-goals

- Changing the device flow protocol, status codes, or the `/oauth/device/code` and token
  responses.
- Rewriting the login page's error handling. Login renders errors with `http.Error`; see
  [Decision](#decision-in-page-errors) for whether the device page should differ.
- Org-level branding (`OrganizationBranding`). Login does not use it either.

## Design

### Template resolution (`pkg/templates/loader.go`)

Resolve each template independently through one helper:

```go
// loadTemplate parses the first candidate file that exists, else the inline fallback.
func loadTemplate(name string, candidates []string, inline string) (*template.Template, error)
```

| Template | Candidates | Inline fallback |
|---|---|---|
| login | `/config/login.html`, `templates/default.html` | `defaultTemplate` |
| device | `/config/device.html`, `templates/device.html` | `defaultDeviceTemplate` |

- `New` calls `loadTemplate` for both, so overriding one never pins the other to its fallback.
- An unexported `newWithPaths(loginCandidates, deviceCandidates)` backs `New`, so tests can point at
  temp files without `chdir`.
- `NewFromFile(path)` keeps its current meaning (login from `path`, device resolved normally) so
  existing callers and `TestShippedTemplateParsesAndRenders` are untouched.
- File templates are named after the file's base name, as `NewFromFile` already does, because
  `ParseFiles` associates by base name.
- Both templates share `templateFuncs`.

### Shipped template (`templates/device.html`)

A new file styled like `templates/default.html`:

- Same structure and classes: `auth-container`, `brand-header` with logo initial (`substr`),
  `Branding.Title` / `Branding.Subtitle`, `primary-button` using `Branding.PrimaryColor` with the
  same gradient fallback.
- The same `mock-notice` with `123456` prefilled in the verification code field, as login does.
- Three states keyed off the data contract below: enter code, confirm and approve/deny, result
  message.
- No JavaScript required. The device flow is a plain form round trip.

The inline `defaultDeviceTemplate` shrinks to a bare, unstyled form matching the style of the
inline login fallback. It stays because `pkg/server` tests run where no template file is found,
and they post the same field names.

### Template data contract

Documented in the README and pinned by tests:

| Field | Present when | Notes |
|---|---|---|
| `.Branding` | always | `config.Branding`: `ServiceName`, `LogoURL`, `PrimaryColor`, `Title`, `Subtitle` |
| `.UserCode` | confirm step | formatted `XXXX-XXXX` |
| `.ClientName` | confirm step | client `name`, else `client_id` |
| `.Audience` | confirm step | |
| `.Scope` | confirm step | space-separated |
| `.Message` | result step | approved / denied text |

Required form fields:

- Entry step: `GET /device` with `user_code`.
- Confirm step: `POST /device` with `user_code` (hidden), `confirm_user_code`, `identifier`, `code`,
  and a submit named `decision` with value `approve` or `deny`.

### Helm chart

Mirror `customLogin` exactly:

- `values.yaml`: `customDevice: { enabled: false, html: "" }`.
- `configmap.yaml`: a `-device` ConfigMap with a `device.html` key when enabled.
- `deployment.yaml`: mount it at `/config/device.html` via `subPath: device.html`.

### Server (`pkg/server/device.go`)

No behavior change. `handleDeviceVerification` and `renderDeviceVerification` already pass
`Branding` as the full struct. The only edit is dropping the unused `message` parameter from
`renderDeviceVerification` (its only caller passes `""`).

## Decision: in-page errors

Device verification errors (wrong code, unknown user, expired, rate-limited) currently go out as
plain-text `http.Error`, the same as login.

- **Option A, match login (chosen):** keep `http.Error`. The goal of this PR is consistency with
  the rest of the mock, and login does the same.
- **Option B, render in-page:** add `.Error` to the contract and render the template with the same
  status code. This is closer to real Auth0's hosted device page, but it makes device diverge from
  login. If we want it, do it for both pages in a follow-up.

## Tests

Following AGENTS.md: drive real behavior, and prove each test catches its bug by reverting the fix.

`pkg/templates/loader_test.go`:

- `TestShippedDeviceTemplateParsesAndRenders`: parse `../../templates/device.html` and render all
  three states. Assert the user code, client name, `Branding.Title`, the `substr` logo initial, and
  each required form field name are present.
- `TestDeviceTemplateResolvesIndependently`: with `newWithPaths`, a custom login file plus no device
  file yields the inline device fallback; a custom device file plus no login file yields the custom
  device page. This catches the current early-return bug.
- `TestNewFallsBackToInlineTemplate`: extend to assert the device fallback renders and carries the
  required field names.

`pkg/config/chart_test.go`:

- With `customDevice.enabled=true`, the rendered manifests contain the `-device` ConfigMap and a
  `/config/device.html` mount. With it disabled, neither appears.

`pkg/server/device_test.go`:

- Existing tests continue to pass against the inline fallback, unchanged.

## Docs

- README: add "Custom Device Template" next to "Custom Login Template", with the chart snippet, the
  data contract and the required form fields.
- AGENTS.md Notes: extend the `templates/default.html` note to cover `templates/device.html` and
  `TestShippedDeviceTemplateParsesAndRenders`.

## Verification

```bash
go test -race ./...
golangci-lint run ./...
just ci
just docker   # walk /device end to end with the shipped template
```

Then repeat with a mounted `/config/device.html` to confirm the override wins and login is
unaffected.

## Steps

1. Refactor `loader.go` to `loadTemplate` / `newWithPaths`, fixing the independent-resolution bug.
   Add the resolution test first and watch it fail.
2. Add `templates/device.html` and its shipped-template test. Trim the inline fallback.
3. Chart: values, ConfigMap, mount, chart test.
4. Remove the unused `message` parameter from `renderDeviceVerification`.
5. README and AGENTS.md updates.
6. Full verification, then the PR.
