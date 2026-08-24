[← Roomba+](../README.md)

# Roomba 900-series startup timeout investigation

---

## Status

**Field hypothesis confirmed; production design pending.** A tagged proof of
concept based on stable v3.5.2 changes only the outer setup deadline from 16 s
to 45 s. It is deployed on the field Home Assistant instance; no upstream
runtime change or pull request has been proposed yet.

This document is the evidence and decision log for a possible Roomba 900-series
startup fix. It intentionally separates what has been observed from what is
inferred, and it records the remaining gaps before a production pull request is
proposed.

---

## Problem statement

A stock Roomba 980 can accept locally retrieved credentials and establish a
local MQTT session, yet both the Home Assistant Core integration and Roomba+
can reject setup because the robot has not published the state used as the
readiness signal before a hard deadline.

The question is not simply whether a larger timeout makes one robot work. The
question is:

> Does Roomba+ classify a slow but valid 900-series MQTT startup as a failed
> connection, and can that false negative be removed without weakening actual
> authentication or setup failures?

The field experiment confirms that the deadline causes Roomba+'s specific
failure: the unchanged config entry completed several clean loads between 19 s
and 26 s with only the outer ceiling raised to 45 s. A later rapid-reconnect
attempt also exceeded 45 s before automatic retry recovered, so the concept is
confirmed but the experimental ceiling is not production-ready. The separate
official-app failure remains consistent with a stale cloud/product record and
does not describe the robot's local health. Exact per-stage MQTT timings are
still required before choosing a production design.

---

## Field environment

| Component | Value |
|---|---|
| Home Assistant Core | 2026.8.2 |
| Roomba+ | 3.5.2 |
| `roombapy` | 1.9.1 |
| Robot | Roomba 980 / SKU family R98---- |
| Robot firmware | stock `v2.4.17-138` |
| Transport | local MQTT over TCP 8883 |

Network addresses, hardware identifiers, and credentials are deliberately
omitted. No credential was printed or written to this repository.

---

## Evidence

### Field reproduction — 2026-08-24

1. Home Assistant Core retrieved a local password and created an entry, but
   setup retried. Re-entering the retained password failed inside Core's 10 s
   validator.
2. An independent `roombapy==1.9.1` client, allowed to wait 20 s, accepted the
   same locally retrieved credentials and returned the robot's live state and
   name. The password was then discarded.
3. Retrying while the robot was idle, actively cleaning, and after a safe
   reboot did not make Core complete setup.
4. Roomba+ retrieved pairing credentials and created an entry, then repeatedly
   spent approximately its 16 s setup window in `setup_in_progress` before
   returning to `setup_retry` with `Cannot connect`.
5. Parallel TCP probes found port 8883 open while the entry was in
   `setup_retry`, closed while Roomba+ was in `setup_in_progress`, and open
   again immediately after the timeout. This repeated across several cycles.
   The timing is consistent with Roomba+ owning the robot's single local MQTT
   slot during setup, then releasing it on timeout; it is not consistent with
   an unreachable host or a firewall rejecting the connection.
6. Disabling the entry returned it to `not_loaded`, and TCP 8883 remained open.
7. The official iRobot app still recognizes the existing product but displays
   it as offline with connectivity code C510: `Connection to Wi-Fi or cloud was
   lost when its battery ran out.` The app shows a stale yellow battery state
   while the physical robot is awake. This is evidence for a broader cloud
   reachability, account-registration, lifecycle-support, or robot-side issue
   and prevents the timeout from being treated as the complete explanation.
8. At the time of investigation, the network controller showed the robot
   associated and authorized on 2.4 GHz with -34 dBm signal, -95 dBm noise,
   zero reported transmit drops, and 99% client satisfaction. The cumulative
   transmit retry rate was 11.1%. This makes basic Wi-Fi reachability unlikely
   to explain the official-app failure, but it cannot verify iRobot cloud or
   account registration.
9. The app's product record displayed a cached SSID different from the SSID to
   which the network controller showed the robot actually associated. Both
   SSIDs map to the same unisolated LAN. On an Android phone, joining the
   robot's live SSID, force-stopping the app, and reopening it left the cached
   SSID unchanged and the Product Wi-Fi Details control disabled. This is
   consistent with a stale product/cloud record that the app cannot repair
   through its same-network gate.
10. A proof-of-concept package was built from the exact v3.5.2 tag. Its only
    runtime diff changes the outer `asyncio.timeout(16)` to
    `asyncio.timeout(45)`; firmware, credentials, connection mode, readiness
    predicates, and entity behavior are unchanged.
11. HACS installed the tagged package and Home Assistant restarted once while
    the existing config entry remained disabled. The same entry and stored
    credential then completed three clean loads in 26 s, 19 s, and 22 s. Every
    success exceeds the original 16 s ceiling.
12. After load, the integration registered 109 entities, reported the vacuum
    idle and battery at 68%, held the robot's one local MQTT slot, and produced
    no Roomba repair issues. This validates the stored credential and confirms
    that the official app's yellow battery display was not current robot state.
13. One intermediate enable request outlasted the diagnostic client's
    unrelated 30 s WebSocket timeout. The client disconnected while setup was
    still in progress; Home Assistant retried and loaded. The final controlled
    run used a 75 s observation timeout and completed normally in 22 s. This
    harness artifact is excluded from the three clean success timings.
14. A later rapid-reconnect sequence produced two consecutive 22 s loads, then
    reached the full 45 s POC deadline and returned to `setup_retry`. Home
    Assistant's automatic retry subsequently loaded the entry. This does not
    weaken the false-timeout diagnosis, but it shows that 45 s is not yet a
    justified production ceiling and that five-second reconnect cadence may
    trigger additional robot-side delay or backoff.

These observations confirm that Roomba+ misclassifies a valid slow startup as a
connection failure under the 16 s ceiling. They do not yet identify whether
the excess time is spent in transport establishment, first reported state, or
reported `name`; production instrumentation must preserve that distinction.

### Repository history

| Evidence | Observation | Implication |
|---|---|---|
| [Initial release `b4a91c8`](https://github.com/johnnyh1975/ha_roomba_plus/commit/b4a91c89a86b311e8e5220af467c6c192d1bb99b) | `async_connect_or_timeout()` began with a 10 s outer timeout, waited for both `roomba_connected` and reported `name`, then slept another 2 s for the initial state snapshot. | The readiness gate and budget were inherited architecture, not introduced for a measured 900-series regression. |
| [Smart Map fix `0081542`](https://github.com/johnnyh1975/ha_roomba_plus/commit/008154288b782de2b5f1f5391ab3ad08e1dc98f2) | The outer timeout changed from 10 s to 16 s in the same patch that added a wait of up to 6 s for `pmaps`. The commit message and comments discuss Smart Map capability detection, not connection latency. | The most direct reading is that 16 s was computed as the previous 10 s budget plus a new 6 s map budget. No repository evidence was found that 16 s was measured against slow 900-series startup. |
| [Current connection helper](https://github.com/johnnyh1975/ha_roomba_plus/blob/28876c003f1056956899c8a0cb2536a290d42f21/custom_components/roomba_plus/__init__.py#L3181-L3210) | One 16 s scope includes the blocking `connect()` call, polling for `name`, an unconditional 2 s snapshot delay, and a conditional wait of up to 6 s for map data. | A single deadline conflates transport establishment, minimum identity readiness, and richer capability hydration. The optional waits reduce the budget available to earlier stages. |
| [Current config flow](https://github.com/johnnyh1975/ha_roomba_plus/blob/28876c003f1056956899c8a0cb2536a290d42f21/custom_components/roomba_plus/config_flow.py#L352-L389) | DHCP discovery already provides `self.name`; after push-button password retrieval, `validate_input()` runs only when `self.name` is missing. | A normally discovered entry can be created without testing the new credentials. Entry creation is therefore not evidence that the MQTT credentials were validated. This is a separate observability/validation issue, not a user setup error. |
| Current tests | No test directly exercises `async_connect_or_timeout()` or delayed arrival of `roomba_connected`, `name`, capability state, or `pmaps`. | The timing contract can regress without a test failure. Existing config-flow tests do not establish an acceptable startup-latency envelope. |
| [`quality_scale.yaml`](https://github.com/johnnyh1975/ha_roomba_plus/blob/28876c003f1056956899c8a0cb2536a290d42f21/custom_components/roomba_plus/quality_scale.yaml#L74-L79) | The integration records `test-before-setup` as complete and says local MQTT succeeds or raises `ConfigEntryNotReady`. | A valid robot that is rejected only because its first state is late conflicts with the documented setup contract. |

The 3.5.2 tag and current `main` retain the same 16 s helper behavior, so the
field result is relevant to the current development branch.

### Upstream corroboration

- [Home Assistant Core issue #117071](https://github.com/home-assistant/core/issues/117071)
  reports 900-series setup failures at the `name` readiness wait. Reports include
  Roomba 960/980 devices and retries while actively cleaning.
- [Home Assistant Core PR #129230](https://github.com/home-assistant/core/pull/129230)
  changed `roombapy` setup to `continuous=True` and restored the default
  connection delay. Roomba+ already uses `continuous=True`, so repeating that
  fix is not the missing change here. The PR did not change or test the hard
  readiness timeout.
- [`roombapy` issue #322](https://github.com/pschmitt/roombapy/issues/322)
  contains a log from the same Roomba 980 SKU and firmware family. Its 10 s
  helper deadline expires, then MQTT returns CONNACK roughly 1.2 s later and
  state messages follow. A separate wrong-password run reports explicit
  `Not authorised`, distinguishing late success from authentication rejection.
- [`roombapy` issue #265](https://github.com/pschmitt/roombapy/issues/265)
  records broader discovery/connection fragility when packets are missed and
  asks for better logging and integration coverage.
- [iRobot's current connectivity-error documentation](https://answers.irobot.com/knowledge/15431)
  identifies C510 as a charging/Wi-Fi/internet/reboot path and explicitly warns
  that 900-series robots manufactured from 2015 through 2018 may no longer be
  supported. This makes loss of the official cloud path plausible even while
  the local MQTT service remains usable.

Together, the repository history, exact-model precedent, and field A/B result
establish the false-timeout mechanism. The remaining evidence gap is
stage-specific timing, not whether a longer bounded deadline permits a valid
connection.

---

## Alternative hypotheses

| Hypothesis | Status | Evidence and remaining gap |
|---|---|---|
| Incorrect pairing procedure | **Strongly disfavored** | The robot chimed in response to the documented button sequence, password retrieval completed, and an independently retrieved credential established a live session. |
| Incorrect password in the current Roomba+ entry | **Eliminated** | The unchanged stored credential completed three MQTT setups when only the outer timeout changed. The DHCP validation bypass remains a separate design weakness but did not cause this field failure. |
| Another client occupies the one MQTT slot | **Strongly disfavored** | Port 8883 is available outside setup, becomes occupied exactly during Roomba+ setup, and returns immediately after Roomba+ times out. No Core Roomba entry is loaded. A debug trace can make this conclusive. |
| Robot must be awake or cleaning | **Disfavored** | The same failure occurred while idle, actively cleaning, and after reboot. Upstream reports also include active-cleaning retries. |
| LAN, routing, or firewall failure | **Eliminated for the observed session** | Direct TCP access works, the integration acquires the MQTT slot, and an independent local client returned state. |
| Unsupported or custom firmware | **Disfavored** | The device runs stock firmware, Roomba+ lists the Roomba 980 as tested, and the same firmware appears in the upstream late-CONNACK report. |
| Robot-side provisioning, cloud-registration, or official lifecycle-support fault | **Plausible** | The app retains the product but reports C510 and a stale battery state. The robot is nevertheless stably associated to Wi-Fi and serves local MQTT. iRobot warns that 2015–2018 900-series units may no longer be supported, so LAN connectivity, cloud registration, stale account state, and service retirement must remain distinct hypotheses. |
| Valid startup exceeds the readiness deadline | **Confirmed for the Roomba+ symptom** | With only the deadline raised, the unchanged entry loaded in 26 s, 19 s, and 22 s. Exact local stage timings remain to be captured, and the official-app stale-record failure is a separate fault. |

---

## Decision log

| ID | Date | Decision | Reason |
|---|---|---|---|
| D-001 | 2026-08-24 | Keep the robot on stock firmware. | The failure is reproducible in the integration boundary, while the stock robot can serve MQTT state to an independent client. Reflashing would add risk and destroy the clean control case. |
| D-002 | 2026-08-24 | Do not change live behavior during the investigation phase. | Establish the problem and the falsification criteria before applying a fix. The field entry was kept disabled until the controlled proof-of-concept. |
| D-003 | 2026-08-24 | Use a one-variable 80/20 experiment: retain readiness semantics and temporarily raise only the outer setup deadline from 16 s to 45 s. | This isolates whether the deadline causes the false negative. Refactoring readiness and changing the timeout together would make the result ambiguous. |
| D-004 | 2026-08-24 | Do not remove the `name`, initial-snapshot, or map waits in the proof-of-concept. | Entity and capability setup assumes initial state hydration. Changing that contract is more invasive and requires broader tests. |
| D-005 | 2026-08-24 | Treat 45 s as an experimental ceiling, not the proposed production value. | A production value or staged-deadline design should follow measured distributions and explicit failure behavior. |
| D-006 | 2026-08-24 | Do not open an upstream PR from the proof-of-concept alone. | The PR should add focused tests, stage-specific diagnostics, a bounded rationale, and documentation after the A/B result is known. |
| D-007 | 2026-08-24 | Characterize the official-app pairing failure before attributing all behavior to Roomba+'s deadline. | A second independent provisioning path now reportedly fails. The timeout experiment remains valid for Roomba+, but it cannot establish overall robot health or explain an app/cloud failure. |
| D-008 | 2026-08-24 | Do not factory-reset or remove the robot from the iRobot account during diagnosis. | The robot currently retains working LAN association and local MQTT, while iRobot warns that some units of this age may no longer be supported. Destroying the working local state could make recovery impossible and is unnecessary for the timeout A/B test. |
| D-009 | 2026-08-24 | Stop the official-app recovery path after the non-destructive SSID/cache test and proceed with the local-only timeout experiment. | The app retains a stale SSID despite the Android phone joining the robot's live SSID and restarting the app. Local MQTT remains healthy, so further app recovery is outside the Roomba+ compatibility experiment and risks the working local state. |
| D-010 | 2026-08-24 | Accept the timeout hypothesis and leave the successful field POC loaded while production work remains separate. | Multiple clean loads succeeded beyond the old deadline with the same entry and credential, while one rapid-reconnect run also exceeded 45 s before automatic retry recovered. Reverting immediately would restore a known false failure; claiming 45 s as production-ready would overstate the evidence. |

---

## Controlled experiment

### Instrumentation

Record monotonic elapsed time for these milestones without logging BLID,
password, IP address, or raw state payloads:

1. helper entered;
2. `roomba.connect()` returned;
3. `roomba_connected` became true;
4. first reported-state message arrived;
5. reported `name` arrived;
6. initial 2 s hydration wait completed;
7. `pmaps` arrived or its optional wait ended;
8. setup succeeded, timed out, or raised an authentication/transport error.

### A/B procedure

1. Retrieve one fresh credential during a single pairing window and retain it
   only in Home Assistant's config entry storage.
2. Record the official app's exact failing step and user-visible error during
   the same pairing window; do not factory-reset the robot solely for this
   experiment.
3. On the unmodified 16 s implementation, capture a baseline failure and its
   stage timings.
4. Change only the outer deadline to 45 s and repeat with the same credential,
   robot state, network, and `continuous=True` setting.
5. Run at least three setup attempts, including one after a robot reboot if it
   can be done without changing any other condition.
6. Disable the entry after the test if the integration would otherwise retain
   the single local MQTT slot.

### Acceptance criteria for the proof-of-concept

- **Passed:** the unmodified implementation reached its 16 s deadline without
  an authentication rejection.
- **Partially passed:** the timeout-only implementation completed clean setups
  in 26 s, 19 s, 22 s, and 22 s with the unchanged entry and credential. A
  later rapid-reconnect run exceeded 45 s, so the original three-consecutive
  reliability criterion is not satisfied and 45 s remains experimental.
- **Pending production instrumentation:** total setup time proves the original
  budget was exceeded, but the current helper does not expose which internal
  stage consumed it.
- **Passed:** no credentials or private network identifiers appear in logs or
  commits.
- **Passed:** no firmware, readiness predicate, connection mode, entity
  behavior, or cloud behavior changed between A and B.

The observed 45 s failure does not undo the successful A/B result; it shows that
the chosen experimental ceiling and reconnect cadence are not yet a reliable
production contract. The next step is stage-specific timing with realistic
cooldown, not blindly selecting a still-larger timeout.

---

## Production follow-up if the experiment succeeds

The production proposal should be selected from evidence rather than assumed
in advance. At minimum it should:

- name and centralize the setup deadline;
- distinguish authentication, transport, and state-hydration failures in logs;
- add unit tests for delayed connection and delayed `name` arrival, plus a
  boundary case just beyond the chosen limit;
- preserve timely failure for unreachable hosts and rejected credentials;
- decide explicitly whether the credential-validation bypass in the normal
  DHCP path belongs in this PR or a separate issue;
- explain the 900-series field data and compatibility scope in the PR without
  including private device identifiers.

A staged readiness design may ultimately be better than one large outer
timeout, but it is deliberately deferred until the one-variable experiment
shows which stage is slow.

---

## Change log

| Date | Change |
|---|---|
| 2026-08-24 | Forked current upstream `main` at `28876c0` for investigation; no runtime changes. |
| 2026-08-24 | Reproduced the setup-retry/TCP-slot timing pattern against a stock Roomba 980. |
| 2026-08-24 | Traced the 10 s → 16 s timeout history and found no timing-specific test or measurement rationale. |
| 2026-08-24 | Recorded upstream same-model evidence, alternative hypotheses, decision points, and the controlled A/B protocol. |
| 2026-08-24 | Added the owner's history of unsuccessful official-app pairing and a contemporaneous, privacy-scrubbed network-health observation; lowered confidence that the timeout is the robot's only fault. |
| 2026-08-24 | Captured the official app's C510 offline/stale-battery state, manufacturer lifecycle warning, and the decision to preserve the robot's still-working local configuration. |
| 2026-08-24 | Confirmed the app's cached SSID differs from the live robot association and survives an Android force-stop/reopen on the live SSID; stopped before any reset or reprovisioning. |
| 2026-08-24 | Built and installed stable-based `v3.5.2-chatika-poc.1`; the sole runtime change is the 16 s → 45 s outer setup deadline. |
| 2026-08-24 | Confirmed three clean loads at 26 s, 19 s, and 22 s with the unchanged stored credential; verified 109 entities, idle state, 68% battery, and no Roomba repair issues. |
| 2026-08-24 | Extended the reliability run: another 22 s load succeeded, one rapid-reconnect attempt exceeded 45 s, and HA's automatic retry recovered. Retained 45 s as a field POC only, not a proposed production value. |
