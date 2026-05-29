# Cisco Unsaved Config Detection — Implementation Plan

Monitor whether a Cisco device has unsaved configuration changes by polling three
CISCO-CONFIG-MAN-MIB OIDs and computing a state flag that alert rules can query.

## OIDs

| OID name | OID number | Purpose |
|---|---|---|
| `ccmHistoryRunningLastChanged` | `1.3.6.1.4.1.9.9.43.1.1.1` | TimeTicks when running config last changed |
| `ccmHistoryRunningLastSaved` | `1.3.6.1.4.1.9.9.43.1.1.2` | TimeTicks when running config last saved to NVRAM |
| `ccmCLIHistoryCmdEntries` | `1.3.6.1.4.1.9.9.43.1.2.2` | Count of entries in the CLI history command table |

## Unsaved-change logic

```
$basic_unsaved    = $last_changed_ticks > $last_saved_ticks;
$cli_entries_grew = $current_cli_entries > $state->cli_cmd_entries;

$has_unsaved = $basic_unsaved && ($cli_entries_grew || $state->has_unsaved_changes);
```

- `basic_unsaved` is the primary condition (running config changed after last save).
- `cli_entries_grew` guards against the false positive where `conf t` + `exit` bumps
  `ccmHistoryRunningLastChanged` without any actual config commands being run.
- `$state->has_unsaved_changes` provides state persistence: once unsaved changes are
  detected, the flag stays `true` across subsequent polls until the config is saved.

---

## Tasks

### 1. Database migration

- [ ] Create `database/migrations/YYYY_MM_DD_000000_create_cisco_config_states_table.php`
  - [ ] `id` (PK, auto-increment)
  - [ ] `device_id` (UNIQUE FK → `devices.device_id`)
  - [ ] `running_last_changed` (BIGINT UNSIGNED, raw TimeTicks)
  - [ ] `running_last_saved` (BIGINT UNSIGNED, raw TimeTicks)
  - [ ] `cli_cmd_entries` (BIGINT UNSIGNED, stored for delta comparison across polls)
  - [ ] `has_unsaved_changes` (BOOLEAN, the computed alert flag)
  - [ ] `created_at`, `updated_at` timestamps

### 2. Eloquent model

- [ ] Create `app/Models/CiscoConfigState.php`
  - [ ] Extend `BaseModel`
  - [ ] `belongsTo(Device::class)`
  - [ ] Cast `has_unsaved_changes` to `bool`

### 3. Polling module

- [ ] Create `LibreNMS/Modules/CiscoConfigMgmt.php` implementing `LibreNMS\Interfaces\Module`
  - [ ] `dependencies()` — return `[]`
  - [ ] `shouldDiscover()` — `$status->isEnabledAndDeviceUp($device) && $os instanceof \LibreNMS\OS\Shared\Cisco` (restricts discovery to Cisco devices; avoids unnecessary SNMP probes on all other vendors)
  - [ ] `discover()` — fetch `ccmHistoryRunningLastChanged.0` via `SnmpQuery`; create row on success, delete any existing row on no response (handles devices where the MIB is later removed/unavailable)
  - [ ] `shouldPoll()` — `$status->isEnabledAndDeviceUp($device) && $this->dataExists($device)`
  - [ ] `poll()` — fetch all three OIDs in one SNMP call, compute `has_unsaved_changes`, save model
  - [ ] `poll()` — log a device event when state transitions (saved→unsaved or unsaved→saved)
  - [ ] `dataExists()` — `CiscoConfigState::where('device_id', …)->exists()`
  - [ ] `cleanup()` — delete the row, return count
  - [ ] `dump()` — return row as array (excluding transient fields) for OS module tests

### 4. Module registration

- [ ] Add `discovery_modules.cisco-config-mgmt` entry to `resources/definitions/config_definitions.json`
- [ ] Add `poller_modules.cisco-config-mgmt` entry to `resources/definitions/config_definitions.json`
  - Default: `true` — discovery is cheap (one OID probe, Cisco-only) and the feature is useful enough to be on by default for all Cisco devices
- [ ] Add English label/description to `resources/lang/en/settings.php`
- [ ] Run `./lnms translation:generate` to rebuild JS translation files

### 5. Test data

No real device is available, so test data will be hand-crafted. Two snmprec scenarios are
needed to exercise both branches of the detection logic.

snmprec OID types used:
- `67` = TimeTicks (for `ccmHistoryRunningLastChanged` and `ccmHistoryRunningLastSaved`)
- `66` = Gauge32 (for `ccmCLIHistoryCmdEntries`)

- [ ] Identify an existing Cisco OS variant with test data (e.g. `cisco-ios`) or create a
  minimal new variant specifically for this module
- [ ] Add three OID entries to `tests/snmpsim/<variant>.snmprec` representing the **saved**
  state (i.e. `last_changed <= last_saved`, so `has_unsaved_changes` should be `false`):
  ```
  1.3.6.1.4.1.9.9.43.1.1.1.0|67|<last_changed_ticks>
  1.3.6.1.4.1.9.9.43.1.1.2.0|67|<last_saved_ticks_higher>
  1.3.6.1.4.1.9.9.43.1.2.2.0|66|<entry_count>
  ```
- [ ] Add a second variant snmprec representing the **unsaved** state (i.e. `last_changed >
  last_saved` and `cli_cmd_entries` higher than the saved-state value, so `has_unsaved_changes`
  should be `true`)
- [ ] Run `./scripts/save-test-data.php -o <osname>` for each variant to generate the
  expected JSON
- [ ] Verify that `cisco_config_states` appears correctly in `tests/data/<variant>.json` with
  `has_unsaved_changes` set to the expected value for each scenario

### 6. Verification

- [ ] `./lnms dev:check unit -o <osname>` passes with the new module
- [ ] `./lnms dev:check` (full check) passes with no new lint/static-analysis errors
- [ ] Manual smoke test: `./lnms device:discover -vvv HOSTNAME` and `./lnms device:poll -vvv HOSTNAME` show the module running and the DB row being created/updated

### 7. Alert rule

- [ ] Decide whether to ship a default alert rule template in `misc/alert_rules.collection.json`
  or document it as a manual setup step
- [ ] Alert rule condition: `cisco_config_states.has_unsaved_changes = 1`
- [ ] Confirm the alert rule builder can reference `cisco_config_states` (may need to register
  the table in the alert query builder if it doesn't auto-discover Eloquent models)

### 8. PR preparation

- [ ] Review `tests/snmpsim/*.snmprec` and `tests/data/*.json` for any sensitive device data
- [ ] Run `./lnms dev:check` one final time
- [ ] Include a brief description of the feature and the alert rule setup in the PR body
- [ ] Reference the CISCO-CONFIG-MAN-MIB and note that the MIB file already exists at `mibs/cisco/CISCO-CONFIG-MAN-MIB`

---

## Open questions

- **Alert rule registration**: Does the LibreNMS alert builder auto-discover new Eloquent model
  tables, or does it need an explicit registration step? Investigate before starting task 7.
