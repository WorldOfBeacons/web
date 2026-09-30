# World of Beacons marketing site

This repository contains the deployable Next.js static export. No build step is
needed here; application source is maintained separately in `WorldOfBeacons/web-source`.

## Firebase Hosting

- Project: `worldofbeacons`
- Dedicated marketing site: `worldofbeacons-web`
- Deploy target: `www`
- Default URL: https://worldofbeacons-web.web.app
- Custom domains: `worldofbeacons.com` and `www.worldofbeacons.com`
- DNS provider: one.com
- Canonical URL: https://worldofbeacons.com (already declared in the exported HTML
  and sitemap). Both hostnames serve the same content; there is no domain redirect
  in `firebase.json`, so the `web.app` URL stays independently testable.

Run from this repository only:

```sh
firebase login # use eddie.ch3ng@gmail.com
firebase deploy --project worldofbeacons --only hosting:www
```

`.firebaserc` maps `www` exclusively to `worldofbeacons-web`. Never change it to
`worldofbeacons` or `worldofbeacons-app`. The `WorldOfBeacons/cloud` deployment,
its `public/` directory, `wob-app` targets, functions and rules are separate.

HTML and route data revalidate on every request. Hashed `/_next/static/**` assets
are immutable for one year; images and video cache for one hour. Repository
metadata, workflows, documentation, local dependencies and credential files are
excluded from uploads. Do not place secrets anywhere in this public repository.

## Automatic deployments

`.github/workflows/deploy-firebase-hosting.yml` deploys the existing static export
on every push/merge to `main`, and supports manual dispatch from `main`.
It has only `contents: read` GitHub permission and explicitly uses target `www`.
Before deployment it checks the project, site mapping and Hosting-only config.

Repository secret: `FIREBASE_SERVICE_ACCOUNT_WORLDOFBEACONS`.

If credentials have not been configured, use a dedicated service account rather
than Firebase Admin SDK credentials or an Owner/Editor account. From an
authenticated Google Cloud CLI session for `eddie.ch3ng@gmail.com`:

```sh
gcloud iam service-accounts create github-web-hosting \
  --project worldofbeacons --display-name='GitHub web Hosting deployer'

gcloud projects add-iam-policy-binding worldofbeacons \
  --member='serviceAccount:github-web-hosting@worldofbeacons.iam.gserviceaccount.com' \
  --role='roles/firebasehosting.admin'

gcloud projects add-iam-policy-binding worldofbeacons \
  --member='serviceAccount:github-web-hosting@worldofbeacons.iam.gserviceaccount.com' \
  --role='roles/serviceusage.apiKeysViewer'

# Store the key outside this repository; never print or commit it.
umask 077
gcloud iam service-accounts keys create /tmp/wob-web-hosting-service-account.json \
  --project worldofbeacons \
  --iam-account='github-web-hosting@worldofbeacons.iam.gserviceaccount.com'

gh secret set FIREBASE_SERVICE_ACCOUNT_WORLDOFBEACONS \
  --repo WorldOfBeacons/web < /tmp/wob-web-hosting-service-account.json
rm /tmp/wob-web-hosting-service-account.json
```

These are project-level Hosting permissions, so the workflow target guard is
essential. No Auth Admin, Functions, Cloud Run, Editor, Owner or Service Account
User role is needed for this static deployment. Firebase documents the additional
[API Keys Viewer requirement](https://firebase.google.com/docs/projects/iam/roles-predefined-product#hosting).
If a service account already exists, inspect its permissions before reusing it;
do not create duplicate accounts or overwrite an existing secret blindly.

After merging the PR, verify the live deploy in
https://github.com/WorldOfBeacons/web/actions and test the `web.app` homepage,
static assets, and an unknown path (404) before changing production DNS.

## Domain cutover

The exact records and current progress are in [docs/hosting-cutover.md](docs/hosting-cutover.md).
Use Firebase's Advanced setup for the existing apex domain so the certificate can
be prepared while Pages continues serving traffic. Certificate provisioning is
asynchronous; see [Firebase's domain setup guide](https://firebase.google.com/docs/hosting/custom-domain).

Keep `CNAME` until DNS cutover and HTTPS verification have succeeded. It is excluded
from Firebase uploads. After both hostnames have valid Firebase TLS and correct
content, disable Pages for this repo, delete `CNAME` in a follow-up PR, and merge
that cleanup. Do not disable Pages while DNS still points at GitHub.
