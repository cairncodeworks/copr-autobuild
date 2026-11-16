# copr-autobuild

> Keeps COPR packages in sync with upstream GitHub releases ÔÇö automatically.

[![check-releases](https://github.com/cairncodeworks/copr-autobuild/actions/workflows/check-releases.yml/badge.svg)](https://github.com/cairncodeworks/copr-autobuild/actions/workflows/check-releases.yml)
[![license: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

COPR can rebuild a package on a cron schedule or when its own webhook is called, but it has no idea when an upstream project publishes a new **GitHub release**. This repository closes that gap.

A GitHub Actions workflow polls a short list of upstream projects. When it finds a version that has not been built in COPR yet, it calls the COPR package webhook to start a build ÔÇö no polling loop on your side, no manual button pressing.

---

## How it works

```
  cron '17 * * * *'  ÔöÇÔöÇÔöÉ
  manual dispatch    ÔöÇÔöÇÔö┤ÔöÇÔöÇÔûÂ  check-releases.yml
                                 Ôöé
                                 Ôû╝
                     ÔöîÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÉ
                     Ôöé  for each app in      Ôöé
                     Ôöé  apps.json            Ôöé
                     ÔööÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔö¼ÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöİ
                                 Ôöé
              ÔöîÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔö┤ÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÉ
              Ôû╝                                     Ôû╝
   GitHub API /releases/latest              COPR API latest build
   ÔåÆ upstream tag                           ÔåÆ built version
              ÔööÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔö¼ÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöİ
                                 Ôû╝
                 tag != state/  OR  tag not in COPR build?
                                 Ôöé
                    ÔöîÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔö┤ÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÇÔöÉ
                   yes                        no
                    Ôöé                         Ôöé
                    Ôû╝                         Ôû╝
        POST COPR webhook            "no change"
        update state/<app>.txt
                    Ôöé
                    Ôû╝
        commit state/ back to the repo
```

The check is deliberately **belt and braces**:

- `state/<app>.txt` remembers the last version we saw ÔÇö cheap, but it can drift.
- The COPR API tells us what is *actually built*.

Because both are consulted, a missed webhook, a lost state file, or a failed run is recovered automatically on the next pass. The state file is a hint; COPR is the source of truth.

---

## Repository layout

| Path | Purpose |
| --- | --- |
| `.github/workflows/check-releases.yml` | The entire logic ÔÇö schedule, polling, comparison, webhook call, state commit |
| `apps.json` | The watch list: upstream `repo` + the `name` used as the COPR package name |
| `state/<app>.txt` | Last upstream tag seen per application |
| `state/.last-check` | Timestamp of the last successful run (heartbeat) |

---

## Setup

### 1. Secrets

Add these under **Settings ÔåÆ Secrets and variables ÔåÆ Actions**:

| Secret | Description |
| --- | --- |
| `COPR_OWNER` | COPR account or group that owns the project |
| `COPR_PROJECT` | COPR project name |
| `COPR_WEBHOOKS` | JSON map of `package name ÔåÆ webhook URL` |

`COPR_WEBHOOKS` is a single-line JSON object:

```json
{
  "heroic-games-launcher": "https://copr.fedorainfracloud.org/webhooks/custom/0000/1111/heroic-games-launcher/",
  "protonup-qt": "https://copr.fedorainfracloud.org/webhooks/custom/0000/1111/protonup-qt/",
  "zen-browser": "https://copr.fedorainfracloud.org/webhooks/custom/0000/1111/zen-browser/"
}
```

Each key **must match** the `name` field in `apps.json` and the package name in COPR.

### 2. Prepare the COPR side

1. Create the package in your COPR project with the same name as in `apps.json`.
2. Open the package settings in COPR and copy its **custom webhook URL**.
3. Put that URL into `COPR_WEBHOOKS` under the matching key.

### 3. Add the app to the watch list

```json
{ "name": "my-app", "repo": "owner/my-app" }
```

- `name` ÔÇö COPR package name (also the state file name and webhook key).
- `repo` ÔÇö GitHub `owner/repo` whose `/releases/latest` is polled.

### 4. Seed the state (optional)

If you leave `state/my-app.txt` out, the first run treats the app as never seen and triggers one build. To avoid that, create the file with the current tag:

```bash
echo "v1.2.3" > state/my-app.txt
```

---

## Adding a new application ÔÇö checklist

- [ ] Package exists in COPR with the right name
- [ ] Webhook URL copied from COPR
- [ ] Entry added to `apps.json`
- [ ] Key added to the `COPR_WEBHOOKS` secret
- [ ] `state/<name>.txt` created with the current tag (optional, prevents a duplicate build)

---

## Running it manually

**Actions ÔåÆ COPR surumlerini kontrol et ve derlemeleri tetikle ÔåÆ Run workflow**

A manual run is also the quickest fix after a long pause: it re-checks every app and triggers anything that was missed.

---

## Known limitations

- **Unauthenticated GitHub API.** The workflow calls `api.github.com` without a token, which is limited to 60 requests per hour per IP. On shared runner IPs this can be exhausted; the run then logs a warning and skips the app. Passing `GITHUB_TOKEN` raises the limit to 1000/hour.
- **`cron` is best effort.** GitHub does not guarantee hourly execution; in practice this repository runs roughly 4ÔÇô6 times a day, so expect a few hours of latency between an upstream release and the rebuild.
- **Scheduled workflows can be paused.** Long periods without human activity may cause GitHub to disable the schedule. If `state/.last-check` looks old, trigger a manual run and re-enable the workflow.
- **No concurrency guard.** Overlapping runs can race on `git push`. It is rare but has happened once.
- **Version matching is a substring check.** Very short tags (e.g. `1.2`) could false-positive against a longer COPR version (e.g. `1.22.3`).

---

## License

[MIT](LICENSE) ┬® 2026 CairnCodeWorks