# Marketing Hosting cutover

Prepared on 2026-09-30 for Firebase project `worldofbeacons`, site
`worldofbeacons-web`, target `www`. Domain records below were read from Firebase
Console for this site, not inferred from generic examples.

## DNS plan at one.com

Preserve all MX, SPF, DKIM, existing `firebase=worldofbeacons` TXT, and other
unrelated records. one.com accepts a minimum TTL of 600 seconds.

Preparation records (safe while production still uses GitHub Pages):

| Type | Host in one.com | Value | TTL |
| --- | --- | --- | --- |
| TXT | empty (apex) | `hosting-site=worldofbeacons-web` | 600 |
| TXT | `_acme-challenge` | `cOcFsl5tvh9cQQdIcCmByWQSbXvFxWgLoql2WAbQNVI` | 600 |

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
- [ ] Firebase confirms certificate preparation (asynchronous validation pending).
- [x] Manual marketing deploy succeeds on `https://worldofbeacons-web.web.app`.
- [x] Deployment service account JSON installed as the repository secret.
- [ ] PR merged to main; first live GitHub Actions run is green.
- [ ] Firebase apex certificate prepared via Advanced setup.
- [ ] Traffic records updated at one.com; www certificate provisioned.
- [ ] Both hostnames show Connected and valid TLS with marketing content.
- [ ] Pages disabled and `CNAME` removed after cutover.

Baseline before migration: apex returned HTTP 200 from GitHub Pages; www failed
hostname verification. Do not treat that as successful Firebase verification.

```sh
dig +short worldofbeacons.com A
dig +short worldofbeacons.com AAAA
dig +short www.worldofbeacons.com CNAME
dig +short worldofbeacons.com TXT
dig +short _acme-challenge.worldofbeacons.com TXT
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
