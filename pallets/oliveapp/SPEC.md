# oliveapp — ingredients-checking app: APK + API surface research

> Research pallet: obtain the oliveapp APK and map its API endpoints. The
> deliverable is an endpoint report, not a clone. A cloud agent runs this
> entirely in its own sandbox.

## 1. Mission

1. Identify the exact "oliveapp" (ingredients-checking app) the user found —
   its store listing/source URL is unknown; the first sub-task is to pin down
   which app it is (search by name + ingredients-checking description, confirm
   with the user before heavy work).
2. Obtain its **APK** from a legitimate source (store, publisher site, or a
   public APK mirror) — record the SHA-256 + version.
3. **Map the API endpoints**: decompile/static-analyze (apktool / jadx / jadx-gui
   or equivalent) to find base URLs, endpoint paths, auth scheme, headers, and
   sample request/response payloads. If a network probe is feasible from the
   sandbox (mitm/ssl-pinning is usually the blocker), sniff a live session;
   otherwise document what static analysis yields.
4. Produce an **endpoint report**: base URL(s), endpoint table (method, path,
   purpose, auth), payload examples, and a note on what a client would need to
   call it.

## 2. What to deliver

- `APK.md` — download source, version, SHA-256, size.
- `endpoints.md` — the full endpoint map (method/path/purpose/auth/payloads).
- `feasibility.md` — what was possible in a sandbox vs what needs a device/
  mitm (e.g. SSL pinning, geo/region gates, login walls).

## 3. Guardrails

- Public/legitimate sources only; no scraping behind auth or bypassing app
  protections.
- Don't leak credentials; redact anything sensitive from the report.
- Don't ship the APK itself in this repo (too large) — ship hashes + report.

## 4. Definition of done

- [ ] App identified + confirmed with the user
- [ ] APK obtained, hashed, documented
- [ ] Endpoints extracted (static; dynamic if feasible) and mapped
- [ ] Report committed to the pallet + pushed