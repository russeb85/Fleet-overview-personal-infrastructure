# Fleet overview — implementation and deployment plan

Status: planning only; implementation and deployment have not started.
Last updated: 2026-11-01.
Repository: https://github.com/russeb85/Fleet-overview-personal-infrastructure

## Resume here

Build the approved dark dashboard for four personal computers, hosted on Vercel, with Supabase as the recommended backend and a small native agent on each computer.

**Latest user requirement: alerting must be machine-specific.** Each machine owns its rules, thresholds, enabled states, durations, recovery settings, notification cooldowns, and maintenance mute. Editing one machine must never change another machine's rules. See the alerting section and acceptance tests before implementing settings.

Next step: establish the repository structure, Supabase migrations and authorization, and a minimal agent-to-backend ingestion path. Validate on Debian and one Mac before implementing the rest of the dashboard.

The user approved the visual direction and asked to save this plan for later. This document does not claim any resources, agents, bots, or deployments already exist.

## 1. Objective and scope

Provide a simple private dashboard showing:

- Current CPU utilization, historical average, and sampled peak.
- Current memory used, available, and total; historical average and minimum available.
- Storage used and remaining for the main disk and selected additional disks.
- Historical trends and accurate last-seen / stale states.
- Independently configurable alerts for each machine, delivered through Telegram.
- Responsive dark UI suitable for a desktop or phone.

Initial fleet:

| Machine | Role | Connectivity profile |
| --- | --- | --- |
| Debian server | Home automation | Always on |
| Mac Mini | Remote AI coding | Always on |
| Mac Studio | Local AI models | Always on, sustained compute expected |
| M5 MacBook | Everyday work and travel | Portable; sleep and offline periods expected |

Discover actual hardware capacities and OS/CPU architecture during enrollment. The mockup's capacities and measurements were illustrative.

Version one excludes process inspection, remote command execution, GPU-specific telemetry, Docker/service monitoring, and predictive capacity forecasting. These can be separate future additions.

## 2. Recommended architecture

| Component | Technology | Responsibility |
| --- | --- | --- |
| Dashboard | React + TypeScript + Vite on Vercel | Approved design, charts, machine settings |
| Managed backend | Supabase | Postgres, Auth, Edge Functions, scheduled jobs |
| Agent | Go executable using gopsutil | Native measurements and outbound HTTPS uploads |
| Service supervision | launchd on macOS; systemd on Debian | Start on boot, restart on failure |
| Notifications | Dedicated Telegram bot | Warning, escalation, reminder, recovery |

Data flow:

1. Each computer measures its own resources.
2. Agents send authenticated batches directly to a Supabase ingestion endpoint over HTTPS.
3. The backend validates and stores measurements and updates latest state.
4. The signed-in browser reads authorized summary/history data from Supabase.
5. A scheduled backend worker evaluates rules and missing heartbeats, then delivers queued Telegram notifications.

No inbound ports or VPN are required on monitored computers. The MacBook can report from any network allowing outbound HTTPS. The dashboard and alert checker do not depend on one of the monitored computers remaining online.

Use Supabase scheduling for minute-level checks: Vercel Hobby cron is limited to daily runs. No persistent WebSocket connection is necessary.

If all home machines stop reporting, report them as unreachable. Missing heartbeats do not prove whether the cause is a power outage, internet outage, or machine failure. Supabase or Telegram outages can themselves interrupt alerts; this system cannot guarantee notification of its own complete backend failure. An independent external healthcheck can be added later.

### Alternatives considered

- **Cloudflare Workers + D1:** inexpensive managed backend, but requires more work assembling authentication and administration. Paid Workers starts at $5/month at the time of planning.
- **Beszel:** existing monitoring product with macOS agents and Telegram support; fastest if its standard interface is acceptable. Preserving the approved custom dashboard requires integration and a separately hosted hub.
- **Chosen recommendation:** Vercel + Supabase, keeping authentication, storage, functions, and scheduling together.

## 3. Dashboard design to preserve

The approved mockup used the working name "signal" and a restrained dark charcoal theme with mint CPU accents, lavender memory lines, and amber storage warnings.

Preserve:

- "Fleet overview" header, online / attention counts, history range controls.
- Four machine cards containing CPU, memory, and storage readings.
- CPU current, average, and peak; memory current, average, available, and total.
- Storage used / capacity and free space; clear warning state.
- Clicking a machine selects its resource-history chart.
- Telegram message preview and notification history.
- Initial ranges: 1 hour, 24 hours, 7 days; add 30 and 90 days for capacity trends.
- Mobile layout that stacks cards without hiding essential values.
- Loading, empty, error, stale, and offline states; remove sample-data badges in production.

**Required change from the mockup:** the Alert rules view must have an explicit machine selector and heading such as "Alert rules — Mac Studio". Show only that machine's rules and muted state. The initial mockup's shared "Default rules for all four machines" wording is superseded.

A card's alert-settings action opens that specific machine's rules. Show unsaved changes and an explicit Save action. An optional "Copy rules from..." action may copy values into another machine only after an explicit user action; it must not create ongoing inheritance.

The refresh selector controls browser polling, independently of agent upload frequency.

## 4. Measurements and correct interpretation

| Resource | Current | History |
| --- | --- | --- |
| CPU | Whole-machine utilization, normalized 0–100% | Time-weighted average and highest sampled utilization |
| Memory | Total, available, derived used, swap used | Average used and minimum available |
| Storage | Capacity, used, available bytes, percentage used per selected disk/pool | Usage and remaining-space trends |
| Agent health | Last fresh contact, uptime, version | Reporting gaps and connectivity incidents |

Implementation details:

- Sample CPU from counter deltas; discard the initial unprimed measurement.
- CPU peak means the highest five-second measurement, not an instantaneous peak.
- Define memory used consistently as total minus estimated available; retain underlying values and document OS differences.
- High macOS RAM use alone is not proof of pressure. Cached/compressed memory and swap require careful interpretation. Validate against Activity Monitor; add native pressure information if reliable without broad privileges.
- On Linux validate memory against native available-memory reporting.
- Use writable available disk capacity for headroom. Document reserved-space differences and consistent units.
- Select the macOS Data volume and intended local drives; deduplicate APFS shared capacity instead of summing related volumes.
- Exclude temporary, virtual, and network filesystems by default. Make additional local drives opt-in.
- Never convert missing readings into zeros or treat absent measurements as healthy.
- Keep both sample time and receipt time. Reject implausible timestamps; preserve gaps around suspend/reboot.
- Compute averages over observed duration; retain counts/coverage, minima, maxima, and peaks through aggregation.
- Unsupported fields remain unavailable, never fabricated.

## 5. Collection, freshness, retention

| Operation | Default |
| --- | --- |
| Local CPU / memory sample | Every 5 seconds |
| Local disk capacity sample | Every 60 seconds |
| Upload batch | Every 30 seconds |
| Dashboard poll while visible | Every 30 seconds |
| Alert and heartbeat worker | Every minute |
| Mark readings stale | After 90 seconds without fresh telemetry |
| Always-on missing-machine alert | After 5 minutes, independently configurable per machine |

Expected ordinary dashboard lag is around 30–60 seconds. Disk data may be older because it is sampled once per minute. Display relevant timestamps rather than implying every metric was sampled simultaneously.

Poll less or pause while the browser is hidden. Fetch bounded chart summaries rather than all raw history.

| Stored resolution | Retention |
| --- | --- |
| 30-second summaries | 7 days |
| 5-minute summaries | 90 days |
| Hourly summaries | 1 year |
| Alert history | 1 year |

Create aggregates successfully before deleting their source data. Use idempotent rollup jobs, indexed queries, bounded cleanup, and short retention for verbose job logs. Keep enough lateness allowance to incorporate buffered uploads and recompute affected recent summaries.

## 6. Agent installation and lifecycle

Target a single executable per OS/architecture, with no Python or Node dependency on monitored computers. Build macOS arm64 and the Debian architecture actually discovered; add macOS amd64 if needed.

Enrollment flow:

1. Sign in and select Add machine.
2. Enter a display name and choose Always on or Portable.
3. Download a versioned installer and enroll using a short-lived, single-use code.
4. Install the binary, protected configuration, and native background service.
5. Exchange the enrollment code for a unique machine credential.
6. Send a first measurement and show the machine online.
7. Choose extra local disks and review that machine's initial alert settings.

Requirements:

- Service installation may require administrator privileges; run ongoing collection with minimal privileges.
- Automatic start when the OS is available and restart after crashes.
- Sleep normally on laptops; never prevent sleep to keep reporting.
- Retry network failures with backoff and jitter.
- Bounded local queue: up to 24 hours with an additional byte cap.
- Prioritize fresh telemetry on reconnect; upload historical batches separately.
- Identify batches by machine, boot/session ID, and sequence to prevent duplicate insertion.
- Backfilled measurements update history but never create historical alerts or falsely refresh current liveness.
- Bound HTTP timeouts and payload size; report dropped data visibly in diagnostics.
- Include status, diagnostics, update, and uninstall commands.
- Versioned release artifacts with checksums; preserve configuration during updates and support rollback.
- Avoid tokens in logs and shell history.

## 7. Machine-specific alerting

### Ownership and settings

Every rule belongs to exactly one machine. Storage rules may additionally target a disk/pool belonging to that machine.

Each rule stores:

- Machine ID and optional disk/pool ID.
- Metric, comparison, severity, threshold, and enabled state.
- Required breach duration.
- Recovery threshold and recovery duration.
- Reminder interval and notification destination.
- Rule version / updated time.

Machine settings additionally contain connectivity profile, maintenance mute, and preferred Telegram destination. The same private Telegram chat can be used for all machines while all alert evaluation and incident state remain independent.

Initial settings are copied into independent machine records at enrollment. There is no live global threshold inheritance. Changing or disabling Mac Studio CPU alerts must not change CPU alerts on any other machine.

### Proposed initial values

These are editable starting values, not verified optimal thresholds for the hardware.

| Machine | CPU warning | Memory warning | Storage warning / critical | Missing heartbeat |
| --- | --- | --- | --- | --- |
| Debian server | Above 90% for 10 min; enabled | Available below 10% for 5 min; enabled | Above 85% for 10 min / above 95% OR below 20 GB for 2 min; enabled per monitored disk | 5 min; enabled |
| Mac Mini | Above 90% for 10 min; enabled | Available below 10% for 5 min; enabled | Above 85% for 10 min / above 95% OR below 20 GB for 2 min; enabled per monitored disk | 5 min; enabled |
| Mac Studio | Above 90% for 10 min; **disabled initially** because long model workloads are expected | Available below 10% for 5 min; enabled, tune during pilot | Above 85% for 10 min / above 95% OR below 20 GB for 2 min; enabled per monitored disk | 5 min; enabled |
| M5 MacBook | Above 90% for 10 min; enabled | Available below 10% for 5 min; enabled | Above 85% for 10 min / above 95% OR below 20 GB for 2 min; enabled per monitored disk | **Disabled initially** |

Use the storage critical condition as an OR rule; recovery requires both contributing conditions to clear. Store and compare exact byte values, and label displayed GB/GiB consistently.

Suggested recovery defaults, separately editable per machine:

- CPU below 80% for 5 minutes.
- Memory available above 15% for 5 minutes.
- Storage warning below 82% used for 5 minutes.
- Storage critical below 92% used AND above 25 GB available for 5 minutes.
- Missing heartbeat recovers when a valid fresh report arrives.

An unreachable portable computer is labeled "Not reporting — last seen..." rather than claiming it is asleep. No heartbeat cannot distinguish sleep, shutdown, or lost connectivity.

### Evaluation and notification behavior

- Evaluate on the backend even when the dashboard is closed.
- Use fresh window averages for sustained CPU alerts, not peaks.
- Require continuous qualifying coverage; missing or stale windows interrupt pending breach durations.
- Key incident state by machine, rule, and resource. One machine's breach cannot start, clear, or remind another's incident.
- Send a warning when the duration qualifies, escalate on critical conditions, remind at most hourly, and send recovery after stable clearance.
- Coordinate storage warning/critical rules per machine and disk to avoid duplicate simultaneous notifications.
- Never send a recovery solely because the agent stopped reporting.
- While muted, continue collecting data; suppress notifications for only that machine.
- Changes to a rule reset pending evaluation for that rule. Disabling a rule closes it as disabled, not as recovered.
- Persist an outbox and delivery status; claim work atomically, retry failed deliveries with backoff, and avoid duplicate scheduling from overlapping workers.
- A network timeout after Telegram accepts a message can still produce a duplicate on retry; do not promise exactly-once delivery.
- Persist sent, pending, and failed statuses for troubleshooting.

Every message includes the machine name, resource, severity, observed value, threshold, duration, and dashboard link. Notification previews and test messages must show the selected machine.

### Telegram setup

1. Create a dedicated bot through BotFather.
2. Start a private conversation with it.
3. Store its token and destination chat ID as backend secrets.
4. Associate the destination with each machine's notification settings.
5. Explicitly send and verify a labeled test notification during deployment.

Outbound Telegram messages use its HTTPS API; no continuously running bot process is needed. Keep tokens out of Git, the browser bundle, agent configuration, and logs.

## 8. Authentication and backend design

Use Supabase Auth with GitHub sign-in as the proposed default. Restrict dashboard/data access to the owner's verified account; logging in as any GitHub user must not grant access.

Enforce ownership through database row-level security and authenticated endpoints, including settings writes. Keep privileged functions tightly scoped. Only the project URL and publishable client key belong in the frontend; never expose an administrative key.

Each agent has a separate high-entropy revocable credential, stored as a hash server-side and protected on the machine. The ingestion endpoint resolves machine identity from the credential instead of trusting a submitted machine ID. Agents may upload only their own measurements and cannot read fleet history or edit settings.

Logical records:

| Records | Purpose |
| --- | --- |
| Machines, disks, enrollment codes, credentials | Ownership, lifecycle, device identity |
| Recent measurements and latest state | Resource history and current values |
| Rollups | Efficient long-term charts and summaries |
| Machine-owned alert rules | Independent thresholds and behavior |
| Incidents | Pending, active, critical, unknown, recovered, disabled state |
| Notification outbox and history | Delivery, retries, deduplication |

Ingestion must validate payload sizes, numeric bounds, schema versions, timestamps, device ownership, and rate. Persist accepted measurements and latest-state updates transactionally. Out-of-order history must not replace the current snapshot.

The scheduled worker handles alert evaluation and delivery retries; rollups and cleanup may use separate short scheduled SQL jobs. Protect scheduler-only endpoints with backend credentials. Keep ordinary dashboard reads on the database API to avoid unnecessary Edge Function calls.

## 9. Costs and capacity

Pricing was checked during planning on 2026-11-01 and should be rechecked before deployment.

- Vercel Hobby: $0 for eligible personal/noncommercial use.
- Supabase Free: 500 MB database and 500,000 monthly Edge Function invocations; useful for a pilot.
- Supabase Pro: starts at $25/month, includes one Micro instance through compute credits and daily backups with seven-day retention.
- Free Supabase projects may pause after inactivity.
- Optional domain, taxes, paid Vercel requirements, and overages are additional.

For four continuously reporting computers, per 30-day month:

- 345,600 ingestion calls at one upload per machine every 30 seconds.
- 43,200 worker calls at one every minute.
- Approximately 388,800 calls before retries, enrollment, and administrative requests.

The free allowance has limited headroom; additional machines or faster uploads can change the calculation significantly. Use direct authorized database reads for the dashboard. Measure database/index size, egress, worker usage, and cleanup logs during the pilot rather than assuming retention fits.

Recommendation: pilot at $0/month if eligible; budget around $25/month for Supabase once relied upon. Enable available spend controls and usage notifications. For the free pilot, document a manual database/configuration export before changes; for production, verify backups and perform a restore exercise.

## 10. Delivery milestones

Suggested repository structure:

- apps/dashboard — React/TypeScript UI.
- agent — Go collector and native service integration.
- supabase/migrations — schema, authorization, rollups, scheduler setup.
- supabase/functions — ingestion, enrollment, alert/delivery worker.
- scripts — installation and release tooling.
- docs — setup, operations, metric definitions, troubleshooting.

| Phase | Deliverable | Done when |
| --- | --- | --- |
| 1. Foundation | Repo structure, Supabase schema/Auth, machine enrollment | Unauthorized access rejected; enrolled machine can upload |
| 2. Agent | macOS and Debian builds, services, buffering | Readings validated on one Mac and Debian; restart works |
| 3. Dashboard | Approved design wired to real data | Cards, chart selection, history, freshness, errors work |
| 4. Machine-specific alerts | Per-machine editor, evaluator, Telegram outbox | Isolation, warning, escalation, mute, recovery, retry pass |
| 5. Deployment | Vercel app, all four agents, retention jobs | Fleet reports automatically and owner can access privately |
| 6. Pilot | 24–48 hours observation and tuning | No alert spam; laptop behavior, measurements, costs checked |

Planning estimate: 3–5 focused implementation days plus the observation period. Actual effort depends on macOS/APFS validation and account provisioning.

## 11. Acceptance criteria

- [ ] All four machines report actual capacities and resource usage.
- [ ] Measurements agree with native tools within documented sampling/definition differences.
- [ ] CPU peaks survive aggregation; averages use observed durations.
- [ ] Charts preserve gaps; stale values are visibly marked.
- [ ] MacBook sleep/wake does not cause unwanted offline or historical resource alerts.
- [ ] Network loss buffers history; reconnect prioritizes fresh data and safely backfills.
- [ ] Agents survive restart and reboot; uninstall and credential revocation work.
- [ ] Per-machine rule changes persist and survive reload/redeploy.
- [ ] Editing Debian CPU threshold changes no Mac threshold.
- [ ] Disabling Mac Studio CPU alerts leaves all other CPU alerts intact.
- [ ] Muting Mac Mini does not suppress Debian, Studio, or MacBook alerts.
- [ ] Same metric breaches on two machines create two correctly attributed incidents.
- [ ] Recovering one machine does not recover another machine's incident.
- [ ] Storage rules target only disks belonging to their selected machine.
- [ ] Portable offline alerts remain off until explicitly enabled for that machine.
- [ ] Rule-copy action, if implemented, creates independent values rather than inheritance.
- [ ] Threshold duration, missing-data behavior, escalation, hysteresis, reminders, and recovery work.
- [ ] Telegram messages identify the correct machine and resource.
- [ ] Failed delivery retries are visible and overlapping jobs do not schedule duplicates.
- [ ] Alerts work with the dashboard closed.
- [ ] Anonymous users and other signed-in accounts cannot access data.
- [ ] An agent cannot impersonate another machine or modify alert settings.
- [ ] No administrative or Telegram secrets appear in client assets, Git, or logs.
- [ ] Retention jobs, backups/exports, and restore procedures are verified.
- [ ] Test alerts use synthetic data or temporary thresholds rather than filling real disks.

## 12. Deployment inputs still needed

- Vercel project/team and optional domain.
- Supabase project, chosen region, and free-pilot versus paid plan.
- Owner's login identity and OAuth configuration.
- Telegram bot token and private destination, supplied through secret configuration.
- Access to install agents on the four machines.
- Actual Debian architecture and any additional disks/model volumes to monitor.
- Review of proposed independent alert defaults during enrollment.

No secrets, private network addresses, or real telemetry belong in this public planning repository.

## 13. Sources

- [Supabase scheduling](https://supabase.com/docs/guides/functions/schedule-functions)
- [Vercel cron limits](https://vercel.com/docs/cron-jobs/usage-and-pricing)
- [Vercel Hobby eligibility](https://vercel.com/docs/plans/hobby)
- [Supabase pricing](https://supabase.com/pricing)
- [Supabase Edge Function allowances](https://supabase.com/docs/guides/platform/manage-your-usage/edge-function-invocations)
- [Supabase GitHub login](https://supabase.com/docs/guides/auth/social-sign-in/auth-github)
- [Supabase row-level security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [gopsutil platforms and metrics](https://github.com/shirou/gopsutil)
- [Apple memory interpretation](https://support.apple.com/en-nz/guide/activity-monitor/actmntr1004/10.14/mac/15.0)
- [Telegram bot setup](https://core.telegram.org/bots/tutorial)
- [Telegram sendMessage API](https://core.telegram.org/bots/api#sendmessage)
- [Cloudflare Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Beszel agent installation](https://beszel.dev/guide/agent-installation)
- [Beszel notifications](https://beszel.dev/guide/notifications/)
