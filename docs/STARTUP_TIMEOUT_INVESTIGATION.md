[← Roomba+](../README.md)

# Roomba 900-series startup timeout investigation

---

## Status

**Hypothesis validation only.** No runtime behavior has been changed. The field
config entry remains disabled while the evidence is collected.

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

The current evidence makes this a sound problem to investigate. However, the
owner also reports that re-pairing through the official iRobot app has failed
repeatedly for some time. That lowers confidence that the integration timeout
is the robot's only problem, even though it remains a plausible explanation for
Roomba+'s specific failure. The evidence does not yet prove the exact time at
which the field robot publishes `name`, so a controlled A/B experiment is still
required before choosing a production design.

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
7. The owner reports that recent attempts to pair the same robot with the
   official iRobot app have also been unsuccessful. The exact failing stage and
   app error have not yet been captured. This is evidence for a broader
   provisioning, account-registration, or robot-side issue and prevents the
   timeout from being treated as the complete explanation.
8. At the time of investigation, the network controller showed the robot
   associated and authorized on 2.4 GHz with -34 dBm signal, -95 dBm noise,
   zero reported transmit drops, and 99% client satisfaction. The cumulative
   transmit retry rate was 11.1%. This makes basic Wi-Fi reachability unlikely
   to explain the official-app failure, but it cannot verify iRobot cloud or
   account registration.

These observations strongly support a late-readiness hypothesis. They do not,
by themselves, prove that the password stored in the current Roomba+ entry is
identical to the independently validated password; Home Assistant correctly
redacts it from public APIs.

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

Together, the repository and upstream history provide a plausible mechanism,
an exact-model precedent, and a field reproduction. The missing item is a
timestamped trace of the current field robot using the same credential before
and after the candidate timeout change.

---

## Alternative hypotheses

| Hypothesis | Status | Evidence and remaining gap |
|---|---|---|
| Incorrect pairing procedure | **Strongly disfavored** | The robot chimed in response to the documented button sequence, password retrieval completed, and an independently retrieved credential established a live session. |
| Incorrect password in the current Roomba+ entry | **Not fully eliminated** | The standard DHCP path skips `validate_input()` when discovery already supplied a name, and the stored secret is intentionally inaccessible. The A/B test must use one freshly retrieved credential for both cases. |
| Another client occupies the one MQTT slot | **Strongly disfavored** | Port 8883 is available outside setup, becomes occupied exactly during Roomba+ setup, and returns immediately after Roomba+ times out. No Core Roomba entry is loaded. A debug trace can make this conclusive. |
| Robot must be awake or cleaning | **Disfavored** | The same failure occurred while idle, actively cleaning, and after reboot. Upstream reports also include active-cleaning retries. |
| LAN, routing, or firewall failure | **Eliminated for the observed session** | Direct TCP access works, the integration acquires the MQTT slot, and an independent local client returned state. |
| Unsupported or custom firmware | **Disfavored** | The device runs stock firmware, Roomba+ lists the Roomba 980 as tested, and the same firmware appears in the upstream late-CONNACK report. |
| Robot-side provisioning or cloud-registration fault | **Plausible and uncharacterized** | Repeated official-app pairing failures broaden the fault beyond Home Assistant. The robot is nevertheless stably associated to Wi-Fi and serves local MQTT, so the failing official-app stage must be captured before attributing it to LAN connectivity, cloud registration, stale account state, or the robot itself. |
| Valid startup exceeds the readiness deadline | **Leading explanation for the Roomba+ symptom, not necessarily the whole device fault** | Explains the independent-client success, exact timeout transitions, code history, and same-model upstream trace. Exact local stage timings remain to be captured, and official-app pairing failure may represent an additional fault. |

---

## Decision log

| ID | Date | Decision | Reason |
|---|---|---|---|
| D-001 | 2026-08-24 | Keep the robot on stock firmware. | The failure is reproducible in the integration boundary, while the stock robot can serve MQTT state to an independent client. Reflashing would add risk and destroy the clean control case. |
| D-002 | 2026-08-24 | Do not change live behavior during the investigation phase. | Establish the problem and the falsification criteria before applying a fix. The field entry remains disabled. |
| D-003 | 2026-08-24 | Use a one-variable 80/20 experiment: retain readiness semantics and temporarily raise only the outer setup deadline from 16 s to 45 s. | This isolates whether the deadline causes the false negative. Refactoring readiness and changing the timeout together would make the result ambiguous. |
| D-004 | 2026-08-24 | Do not remove the `name`, initial-snapshot, or map waits in the proof-of-concept. | Entity and capability setup assumes initial state hydration. Changing that contract is more invasive and requires broader tests. |
| D-005 | 2026-08-24 | Treat 45 s as an experimental ceiling, not the proposed production value. | A production value or staged-deadline design should follow measured distributions and explicit failure behavior. |
| D-006 | 2026-08-24 | Do not open an upstream PR from the proof-of-concept alone. | The PR should add focused tests, stage-specific diagnostics, a bounded rationale, and documentation after the A/B result is known. |
| D-007 | 2026-08-24 | Characterize the official-app pairing failure before attributing all behavior to Roomba+'s deadline. | A second independent provisioning path now reportedly fails. The timeout experiment remains valid for Roomba+, but it cannot establish overall robot health or explain an app/cloud failure. |

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

- The unmodified implementation reaches its 16 s deadline without an
  authentication rejection.
- The timeout-only implementation reaches reported `name` and completes setup
  on three consecutive attempts.
- The successful trace shows which stage exceeded the original budget.
- No credentials or private network identifiers appear in logs or commits.
- No firmware, readiness predicate, connection mode, entity behavior, or cloud
  behavior changes between A and B.

If the 45 s build fails with the same credential, the timeout hypothesis is
falsified for this field case and the next step is protocol-level logging, not a
still-larger timeout.

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
