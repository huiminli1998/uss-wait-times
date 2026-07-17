# uss-wait-times

Hourly wait times for the two Battlestar Galactica roller coasters at Universal
Studios Singapore, collected by a Claude Code **cloud routine** and appended to
[`waits.csv`](waits.csv).

The routine runs on Anthropic's cloud infrastructure, so collection continues
whether or not any local machine is on.

## The data

`waits.csv` grows by two rows per run — one per ride, sharing one timestamp so a
run can be grouped:

```
fetched_at_sgt,ride,status,wait_time,last_updated
2026-07-16T17:25:28+08:00,HUMAN,OPERATING,50,2026-07-16T09:18:00.442Z
2026-07-16T17:25:28+08:00,CYLON,OPERATING,60,2026-07-16T08:42:29.622Z
```

| Column | Meaning |
| --- | --- |
| `fetched_at_sgt` | When the routine ran, Singapore time. Identical for both rows of a run. |
| `ride` | `HUMAN` or `CYLON` |
| `status` | `OPERATING`, `CLOSED`, `REFURBISHMENT`, or `FETCH_ERROR` |
| `wait_time` | Standby queue, minutes. **Empty** when the ride is closed or the fetch failed. |
| `last_updated` | When the upstream API last refreshed this ride, UTC. Lags `fetched_at_sgt`. |

`FETCH_ERROR` rows mean the run happened but could not read the API — the reason
is in that commit's message. They are deliberate: a visible gap beats a silently
missing hour. Filter them out before analysis.

Read it directly:

```
https://raw.githubusercontent.com/huiminli1998/uss-wait-times/main/waits.csv
```

## Schedule

Cron `23 2-11 * * *` — **note this is UTC**, which is SGT 10:23–19:23, once per
hour, 10 runs/day. That window tracks USS opening hours (~10:00–19:00 SGT);
outside it the park is closed and every row would just say `CLOSED`.

Routine cron expressions are always UTC even though the claude.ai UI shows local
times elsewhere. Getting this wrong shifts collection by 8 hours.

## Data source

[themeparks.wiki](https://themeparks.wiki) — free, no API key.

```
https://api.themeparks.wiki/v1/entity/f95d7f76-2024-4510-b799-26e122d0e448/live
```

| Entity | ID |
| --- | --- |
| Universal Studios Singapore (park) | `f95d7f76-2024-4510-b799-26e122d0e448` |
| Battlestar Galactica: HUMAN | `7b8c7a5b-a4fa-4c74-ac85-6ba72fd653bb` |
| Battlestar Galactica: CYLON | `ccab7e24-6848-4f71-9b2e-459bba5831a0` |

queue-times.com does not carry USS.

## The routine

[`routine/config.json`](routine/config.json) is the exact body the RemoteTrigger
API accepts, prompt included. It exists only in Anthropic's cloud, so this file
is the only way to rebuild it.

Print the prompt on its own:

```bash
jq -r '.job_config.ccr.events[0].data.message.content' routine/config.json
```

To edit the prompt, edit it inside `config.json` (it is the single source of
truth), then `POST` the file back to the routine.

- Routine: <https://claude.ai/code/routines/trig_01No25H7z8uAWyYs5PhTQc4D>
- Trigger id: `trig_01No25H7z8uAWyYs5PhTQc4D`
- Environment id: `env_019fwezJGJJ2oKV8S2FQhuNt`

## Recreating this from scratch

`config.json` alone is **not enough**. Three settings live outside the routine
body, all default to something that silently blocks this job, and none of them
report a useful error from the API — you only see the real reason in the run's
session log at claude.ai/code/routines.

**1. GitHub write access.** `sources.git_repository` only names the repo to
clone; it carries no credentials. Install <https://github.com/apps/claude> and
grant it this repo. Git traffic uses a proxy that is independent of the network
setting below.

**2. Branch push permission.** Cloud routines may only push to `claude/`-prefixed
branches by default, so `git push origin HEAD:main` is rejected. Open the routine
at claude.ai/code/routines → pencil icon → Permissions → check **Allow
unrestricted branch pushes**. Web UI only; the API has no field for it.

**3. Network access level.** The one that costs the most time to find. A cloud
environment defaults to **Trusted**, which permits only a built-in allowlist
(package registries, GitHub, cloud SDKs). Every third-party API returns 403. Fix:
claude.ai/code → click the cloud icon showing the environment name → hover the
environment → settings icon → set level to **Custom** → add `api.themeparks.wiki`
under *Allowed domains* → check *Also include default list of common package
managers*.

Levels are None / Trusted / Full / Custom.

### Dead ends — do not retry these

- **`.claude/settings.json` with `sandbox.network.allowedDomains` does nothing
  here.** Only local sandboxes read it; cloud routines ignore it. See
  [anthropics/claude-code#30112](https://github.com/anthropics/claude-code/issues/30112).
  A file like this was committed during setup and has been removed, because
  leaving it implies the network allowlist is version-controlled when it is not —
  it lives in the environment settings.
- **WebFetch does not bypass the sandbox network policy.** A domain blocked for
  `curl` is 403 for WebFetch too.
- **The Managed Agents API (`api.anthropic.com/v1/environments`) is a different
  product** from claude.ai/code environments. Its networking defaults to
  `unrestricted`; do not use its docs to reason about this routine's defaults.

## Design notes

**Append to one CSV, don't write a file per run.** Git handles append safely —
each run gets a fresh checkout, and runs are an hour apart, so they never race.
An earlier plan stored one JSON per run in Google Drive purely because the Drive
connector has no atomic append and concurrent writes could lose data. On GitHub
that concern disappears, and one CSV loads straight into pandas.

**Always record the run, even when the fetch fails.** The prompt requires a
`FETCH_ERROR` row plus the reason in the commit message. Every failure during
setup was diagnosed from git history alone because of this. Without it, a broken
routine looks identical to a routine that never ran.

**No fallback fetch path.** The prompt explicitly forbids working around a failed
`curl` with WebFetch or anything else. WebFetch answers through a summarizing
model, so a wrong number could be recorded as though it were real — worse than an
honest gap.
