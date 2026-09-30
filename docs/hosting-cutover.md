# Marketing Hosting cutover

Prepared on 2026-09-30 for Firebase project `worldofbeacons`, site
`worldofbeacons-web`, target `www`. Domain records below were read from Firebase
Console and its Hosting API for this site, not inferred from generic examples.

## DNS plan at one.com

Preserve all MX, SPF, DKIM, existing `firebase=worldofbeacons` TXT, and other
unrelated records. one.com accepts a minimum TTL of 600 seconds.

Preparation records (safe while production still uses GitHub Pages):

| Type | Host in one.com | Value | TTL |
| --- | --- | --- | --- |
| TXT | empty (apex) | `hosting-site=worldofbeacons-web` | 600 |
| TXT | `_acme-challenge` | `cOcFsl5tvh9cQQdIcCmByWQSbXvFxWgLoql2WAbQNVI` | 600 |
| TXT | `_acme-challenge.www` | `en3pTaqw-KMiEYaPRtZ6hN1ADs2JhvvOsdE0VlBmnB8` | 600 |

The ACME challenge can change. Reopen the apex domain's Advanced setup in Firebase
Console and compare the value before applying this plan later. Keep validation
records until Firebase says certificate setup is complete.

Traffic changes, only after a successful marketing deploy and TLS preparation:

| Action | Type | Host | Value |
| --- | --- | --- | --- |
| Add/replace | A | empty (apex) | `199.36.158.100` |
| Remove | A | empty (apex) | `185.199.108.153` |
| Remove | A | empty (apex) | `185.199.109.153` |
| Remove | A | empty (apex) | `185.199.110.153` |
| Remove | A | empty (apex) | `185.199.111.153` |
| Remove | AAAA | empty (apex) | `2606:50c0:8000::153` |
| Remove | AAAA | empty (apex) | `2606:50c0:8001::153` |
| Remove | AAAA | empty (apex) | `2606:50c0:8002::153` |
| Remove | AAAA | empty (apex) | `2606:50c0:8003::153` |
| Replace | CNAME | `www` | `worldofbeacons-web.web.app` (was `worldofbeacons.github.io`) |

Firebase did not request a replacement apex AAAA record. Use the current Console
instructions if they change. Preserve a copy of the original traffic records for
rollback, and leave GitHub Pages active through the verification window.

## Verification and progress

- [x] Dedicated Firebase site created; existing default and app sites preserved.
- [x] Apex and www registered on the dedicated site.
- [x] Static config/workflow prepared; local homepage, routes and cache headers checked.
- [x] CLI upload manifest checked: 42 static files, excluding Git metadata and config.
- [x] Preparation TXT records confirmed at one.com's authoritative nameserver.
- [x] Firebase confirms certificate preparation.
- [x] Manual marketing deploy succeeds on `https://worldofbeacons-web.web.app`.
- [x] Deployment service account JSON installed as the repository secret.
- [x] [PR #1](https://github.com/WorldOfBeacons/web/pull/1) merged to main;
  [first live GitHub Actions run](https://github.com/WorldOfBeacons/web/actions/runs/36754060335)
  is green (40 seconds).
- [x] Firebase apex certificate prepared via Advanced setup.
- [x] www CNAME changed at one.com to `worldofbeacons-web.web.app` (TTL 600).
- [x] Apex traffic records updated at one.com; www certificate provisioned.
- [x] Both hostnames show Connected and valid TLS with marketing content.
- [x] Pages disabled and `CNAME` removed after cutover.

At 19:56 Europe/Stockholm on 2026-09-30, the www CNAME was confirmed on both
`ns01.one.com` and `ns02.one.com`. Google and Cloudflare resolvers also saw the
apex ownership TXT. Firebase's ownership checks still reported cached older
records. The apex certificate was `CERT_PROPAGATING`; www was `CERT_VALIDATING`.
A direct apex TLS request to Firebase passed hostname validation but returned
Firebase's 404, so apex traffic remains on Pages pending ownership activation.
Pages and its `CNAME` must remain active through this wait.

Hourly check at 21:40 Europe/Stockholm on 2026-09-30: www now returns HTTP 200
over valid HTTPS, and its homepage SHA-256 matches the export. Firebase reports
`HOST_ACTIVE`, `OWNERSHIP_ACTIVE`, and `CERT_ACTIVE` for www. Apex ownership and
certificate are also active, but its traffic still uses GitHub Pages
(`HOST_MISMATCH`). A direct HTTPS request to `199.36.158.100` with apex SNI passes
certificate validation but still returns HTTP 404. The scheduled preflight
requires correct apex content before changing traffic, so no apex DNS change or
Pages cleanup was performed. Both public hostnames currently serve the correct
homepage; the apex direct-to-Firebase content check remains outstanding.

At 23:40 Europe/Stockholm on 2026-09-30 the direct apex Firebase preflight
returned HTTP 200 with valid TLS and the exact export SHA-256. The apex ownership
and certificate were active. At 23:43 the apex A records were replaced with
`199.36.158.100` (TTL 600), and all four old GitHub Pages AAAA records removed.
Both one.com authoritative nameservers confirmed the new A value; Cloudflare
also saw it, while Google and the local resolver still cached GitHub addresses.
Both public domains remained HTTP 200 with valid TLS. Keep Pages and repository
`CNAME` active until cached old A/AAAA records have cleared and both domains are
verified on Firebase. Screenshot: `/private/tmp/wob-apex-dns-cutover.jpg`.

Final verification at 00:42 Europe/Stockholm on 2026-10-01: both domains report
`HOST_ACTIVE`, `OWNERSHIP_ACTIVE`, and `CERT_ACTIVE`. Both return HTTP 200 with
valid HTTPS and the exact exported homepage SHA-256. Cloudflare and Google
resolvers return only `199.36.158.100` for apex A and no apex AAAA records. The
public apex response now carries Firebase Hosting headers. GitHub Pages was
disabled after these checks; this cleanup removes the old repository `CNAME`.
Retain ownership and certificate TXT records. To roll back in future, first
re-enable Pages and restore its `CNAME`, verify its content and TLS, then restore
the old traffic records above; do not point DNS at an unverified fallback.

The default site's last release remained 2026-09-29 12:39:27 UTC (version
`9ce6cc9b53689fec`); `worldofbeacons-app` still had no releases. Both migration
deploys affected only `worldofbeacons-web`.

Baseline before migration: apex returned HTTP 200 from GitHub Pages; www failed
hostname verification. Do not treat that as successful Firebase verification.

```sh
dig +short worldofbeacons.com A
dig +short worldofbeacons.com AAAA
dig +short www.worldofbeacons.com CNAME
dig +short worldofbeacons.com TXT
dig +short _acme-challenge.worldofbeacons.com TXT
dig +short _acme-challenge.www.worldofbeacons.com TXT
curl --fail --head https://worldofbeacons-web.web.app/
curl --fail --head https://worldofbeacons.com/
curl --fail --head https://www.worldofbeacons.com/
```

Inspect the peer certificate subject/SAN and issuer with SNI for each hostname,
without disabling validation. Compare the served homepage to the deployed export.
After DNS propagation, confirm the old GitHub IPs are absent for both IPv4 and IPv6.

If validation fails after a traffic change, restore only the old traffic records
listed above; keep Pages available until migration is verified. This restores the
prior apex service but does not repair the pre-existing www certificate failure.
