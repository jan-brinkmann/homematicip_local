# HmIP-HAP JS1 system button illumination

## Purpose

Expose channel 0 system button illumination as a pure on/off Home Assistant light.

## Problem

The reported device has a writable MASTER float named `SIGNAL_BRIGHTNESS` with
range 0.0–1.0. Its configuration value was available through the Device
Configuration panel but had no dedicated light entity. A generic number or dimmer
would expose brightness controls that are outside the requested behavior.

## Technical Background

The checkout started clean on `main` at
`fb1af859bb2181cee09d982e843fe80f0d4a9bda`. Work is on the local branch
`feature/hmip-hap-system-button-light`, created from that commit.
All changes remain unstaged and uncommitted. No fetch, pull, push, merge, rebase,
tag, PR, hook installation or Git configuration change was performed.

The implementation was based on the local development checkouts, not on the
production installation (homematicip_local 2.11.1, aiohomematic 2026.9.2,
OpenCCU 3.89.11.20260919, HAP firmware 3.0.18).

The local integration is 2.11.2 (unreleased), pins aiohomematic 2026.10.2,
requires Python 3.14 and Home Assistant 2026.9 or newer, and also supports the
separate openccu-loom backend. The library is 2026.10.2. Local historical tags
for the production versions were unavailable, so an exact historical code diff
could not be made without fetching. The local changelogs document relevant
initial-cache and reconnect fixes and the change from pydevccu to godevccu.

Discovery uses device descriptions and cached paramset descriptions. The generic
resolver normally maps a readable/writable FLOAT to a number. MASTER parameters
must also pass visibility rules. Custom profiles combine registered generic data
points and subscribe to their existing state events.

MASTER reads use the device value cache and client `get_paramset`, ending at
XML-RPC `getParamset`. The central bulk VALUES snapshot is not the source of this
MASTER value. Writes use client `put_paramset`, ending at XML-RPC `putParamset`;
there is no `setParamset` call.

## Architecture Decision

Device recognition and MASTER state/command semantics belong in aiohomematic.
Home Assistant adds a separate light wrapper for this data point, using normal
entity identity, device assignment, subscriptions and availability.

The profile is registered for `HmIP-HAP JS1`, channel 0. Because the registry
also supports prefix matching, creation additionally requires the exact model
(case insensitive), channel type `MAINTENANCE`, and a readable/writable
`DpFloat` from MASTER. No serial number participates in recognition.

## Alternatives Considered

- An HA-only special case would duplicate device recognition and bypass the
  backend's data point state and subscription architecture.
- A global mapping of `SIGNAL_BRIGHTNESS` to light would affect unrelated models.
- A dimmer subclass would carry brightness conversions, timers and capabilities
  into a pure on/off device.
- Generic `send_value` applies optimistic state with a confirmation timeout.
  HmIP MASTER does not emit the matching VALUES callback, so this could later
  roll back an accepted write.
- A new polling loop would add permanent traffic for a configuration parameter.
  The existing initialization, configuration refresh and standard reload paths
  are sufficient for the requested implementation.

## Changed Files

- `custom_components/homematicip_local/light.py`: separate on/off wrapper, dispatch
  and direct HA translation of the entity name.
- `custom_components/homematicip_local/backend_types.py`: typed direct-backend
  recognition beside the existing backend class checks.
- `custom_components/homematicip_local/strings.json`: English entity name.
- `custom_components/homematicip_local/translations/en.json`: English translation.
- `custom_components/homematicip_local/translations/de.json`: German translation.
- `tests/test_light.py`: capabilities, state, commands and normal constructor/dispatch.
- `tests/test_websocket_api.py`: panel API preserves and returns the intermediate
  MASTER value 0.5.
- `changelog.md`: feature note in the existing unreleased section.
- `docs/naming.md`: dedicated system button naming behavior and registry identity.
- `UPDATE.md`: implementation and validation record.

## Implementation

The LIGHT platform dispatches the new backend class to
`AioHomematicSystemButtonLight`. Existing dimmers keep their existing wrapper.
The new wrapper inherits the ordinary generic entity, delegates availability,
identity, device info and state subscriptions, and forwards only on/off commands.
A `LightEntityDescription` supplies the configuration category and translated name.
The wrapper does not subclass the restore/dimmer entity.

The name property uses HA's existing `_name_internal` translation mechanism
directly, following the pattern already used by the generic entity for dedicated
entity translations. The generic custom-data-point naming path replaces the raw
parameter token inside the backend name; that replacement does not match an
already localized name such as "Helligkeit der Systemtaste". The dedicated wrapper
therefore resolves its own entity label from the HA catalog. This preserves normal
HA user-name overrides and does not change existing unique IDs or entity IDs.

## Entity Behavior

| Device value | Light state |
| --- | --- |
| 0.0 | OFF |
| 1.0 | ON |
| 0.5, or another valid positive value | ON |
| Missing/uninitialized value | unknown |
| Unavailable device/client | unavailable through the normal entity base |

`turn_on` requests exactly 1.0; `turn_off` requests exactly 0.0.
A confirmed, valid, certain endpoint already equal to the target skips the write
and the command readback. A positive intermediate value is not equivalent to 1.0
for command deduplication: an explicit ON command still writes 1.0.
Initialization and ordinary refresh do not normalize intermediate values.

The HA wrapper advertises only `ColorMode.ONOFF`, supported features 0, and no
brightness, color, effect or transition support. It is a configuration entity,
enabled by default, with translation key `system_button_light`. The names are
"System button" and "Systemtaste". State is not
restored as a fabricated MASTER value.

## Backend Interaction

Commands stay inside the existing client layer, including its throttling and
retry policy. The client logs failed or superseded writes and returns an empty
set of sent keys. The custom data point then leaves the confirmed state unchanged
and skips command readback. It does not add an optimistic value.

After a successful write, the existing data point loader performs one direct
MASTER read, bypassing the short-lived value cache. A read failure follows the
existing uncertainty behavior and retains the last confirmed value; it does not
claim that the command has already been confirmed.

## MASTER Paramset Handling

The model-specific visibility rule admits only channel 0
`SIGNAL_BRIGHTNESS` from MASTER. The custom profile hides the underlying
generic number by default. Its `allow_undefined_generic_data_points` setting
preserves the existing diagnostic and other generic data points of the HAP.

The Device Configuration panel remains on the existing configuration coordinator:
raw FLOAT metadata and validation retain arbitrary valid values from 0.0 to 1.0.
A panel write of 0.5 remains 0.5; the light reports ON after refresh. The feature
does not alter panel code or validation.

## State Synchronization

Initialization loads the registered MASTER data point through the existing
device value cache. Changes publish the existing data point state event to the
custom light, then the HA entity subscription.

HmIP MASTER has no direct value push callback in this architecture: XML-RPC
parameter events are routed to VALUES data points. The existing
`CONFIG_PENDING` true-to-false path calls `Device.on_config_changed`,
which reloads channel metadata and MASTER values. Manual device configuration
reload and existing scheduled data refresh can also update the state.
No new polling, timer or synthetic MASTER event was introduced.

After local commands the immediate confirmed readback updates the entity.
External changes are visible after a supported configuration/reload event or
standard refresh, not necessarily immediately after a panel or WebUI write.

## Compatibility

Existing dimmer/color light wrappers and generic number mappings are preserved.
The feature adds one new custom data point type rather than changing existing
light capabilities globally. Standard unique ID generation uses the device
address and channel; no serial literal is included. HA prefixes that ID with the
integration domain and uses its normal device identifier/configuration URL.

This implementation targets direct aiohomematic connections. The current pinned
openccu-loom client/daemon do not provide this new custom data point. No loom twin,
dependency pin or daemon capability is invented here.

## Regression Considerations

The isolation tests reject other models (including the registry's prefix case),
wrong channels, wrong channel types, VALUES-only metadata, missing parameters
and read-only metadata. A second synthetic address proves serial-independent
recognition. Existing diagnostic data points are retained.
Existing dimmer tests and Device Configuration panel tests are part of validation.

## Tests

The existing light test module now covers unknown/ON/OFF forwarding, ONOFF-only
capabilities, no brightness/effects/colors/transitions, normal availability,
configuration category, translation key, default enabling, command delegation,
platform dispatch, unique ID and device assignment.
The name test loads the real English and German HA translation catalogs.
It now covers both raw and already localized backend names in both languages,
including the reported German "Helligkeit der Systemtaste". The wrapper must
return "Systemtaste" or "System button" in all four cases. Before platform
translations are attached, it preserves HA's undefined-name behavior.
The panel WebSocket test verifies that 0.5 is passed to the configuration facade
unchanged and returned unchanged by the read API.
Backend tests validate physical values and the actual MASTER RPC routing.
Existing panel and general light tests provide regression coverage.

## Validation

Validation used Python 3.14.4, the pinned Home Assistant test environment
(Home Assistant 2026.10.0b0 and pytest-homeassistant-custom-component 1.0.57),
and the local editable backend. Commands ran from this repository.

```sh
env PATH=/tmp/hmip-hap-ha/bin:/usr/bin:/bin \
  PREK_HOME=/tmp/hmip-hap-prek make check PYTEST_ARGS=-n0
```

The initial implementation's complete gate passed: all repository hooks, including Mypy, Pylint,
Ruff, translations, spelling and backend-type contracts, followed by
**1,042 passed, eight skipped** in 89.44 seconds. Overall integration coverage
was **76.54%**. The eight skips are existing opt-in end-to-end scenarios;
they are not failures or newly disabled tests. The simulator installer downloaded
and verified the project's pinned godevccu v0.8.0.

Focused light, panel WebSocket and backend-type-contract tests passed
(171 tests before the additional German name variant). Both final language
variants and the other new light cases passed together (six tests).
The backend's final light regression run passed 59 tests with 92.33% light-module
coverage; the new backend class has complete measured line and branch coverage.

```sh
env PATH=/tmp/hmip-hap-ha/bin:/usr/bin:/bin \
  PREK_HOME=/tmp/hmip-hap-prek prek run --files UPDATE.md
git diff --check
```

The explicit new-document hook gate passed. The whitespace check reports two
trailing-space lines in the new translation entries in `strings.json` and
`translations/de.json`. The existing mandatory `script/check_translations.py`
formatter emits those spaces with its JSON separators; removing them makes the
hook rewrite the files and fail that run. The final files retain the project's
canonical formatter output. Neither the formatter nor the whitespace check was
changed or suppressed.

During validation, an initial parallel run exhausted worker resources and was
interrupted. A subsequent full serial run hit one delayed-storage-timer teardown
error in an existing loom reauthentication test. Both parameter variants passed
when rerun; the original main checkout's complete config-flow module also passed
(152 tests). The final complete serial gate passed without that error. This is
recorded as a transient earlier failure, not as a demonstrated baseline defect.

The first backend dispatch implementation violated the existing requirement to
centralize backend type checks. Moving recognition to `backend_types.py` fixed
the implementation without adding a contract exception. No tests, thresholds,
hooks, typing rules or quality configuration were disabled or relaxed.
`PYTEST_ARGS=-n0` changes only worker scheduling to fit the workspace resources.

### Follow-up: shorter entity names

The user reported a working hardware installation whose label was "Helligkeit
der Systemtaste". The naming follow-up changes the German label to "Systemtaste"
and English to "System button", and resolves the label directly from the HA
catalog. The backend code and dependencies are unchanged.

The focused command passed all eight system button light cases:

```sh
/tmp/hmip-hap-ha/bin/python -m pytest \
  tests/test_light.py::TestAioHomematicSystemButtonLight \
  -n0 --tb=short --show-capture=no
```

The same complete `make check PYTEST_ARGS=-n0` command documented above then
passed every hook and **1,044 tests, with eight existing skips**, in 86.67 seconds.
Overall coverage is **76.54%**; light-platform coverage is **86.51%**.
An initial naming run exposed a translation-key access before HA had attached
the entity platform. The wrapper now returns its description's undefined name
until that platform exists. All four language/name variants retain the explicit
pre-platform assertion and pass; no test was removed or weakened.

The new local test archive is
`/tmp/hmip-hap-manual-test/hmip-hap-system-button-light-systemtaste-test.zip`.
It contains the complete integration, the existing modified aiohomematic wheel,
and an updated German installation guide. Its test-only manifest references
`/config/hmip-hap-test/aiohomematic-2026.10.2-py3-none-any.whl`.
Repository manifests and release versions remain unchanged. Archive checks
verify ZIP integrity, wheel metadata and the new backend class, the local-wheel
reference, all three entity-name catalogs and the dedicated name implementation.
The backend wheel SHA-256 is
`ee8de116514ebc99c96812520efefe36dce4e6472663b2d9b041000a8ac1c94c`.

## Dependency / Release Considerations

No manifest, runtime version constant, dependency pin or package release version
was changed. The backend changelog uses an Unreleased section; the integration
uses its existing unreleased 2.11.2 section, so published release notes remain
unchanged.

Home Assistant validation uses the local editable aiohomematic and
aiohomematic_test_support packages, installed with:

```sh
uv pip install --python /tmp/hmip-hap-ha/bin/python --no-deps \
  --config-settings editable_mode=compat \
  -e /home/codex/git/ha_hmip/aiohomematic \
  -e /home/codex/git/ha_hmip/aiohomematic/aiohomematic_test_support
```

Before publishing the integration, the backend feature must be released and the
HA runtime/test pins updated to that actual release, with the corresponding
integration release chosen by the maintainer. The currently published backend
2026.10.2 does not contain the new exported class and cannot run the modified HA
light platform without the local checkout.

The setuptools-compatible editable mode also makes the local typed package
discoverable by Mypy. Default editable import hooks produced Any-related errors;
an explicit MYPYPATH made the sources visible but checked unrelated backend
modules under the integration's different Mypy configuration. Compatibility mode
lets both projects pass their own unmodified typing rules.

The task's HA environment resides in the integration's already ignored
`venv/` directory, with its original /tmp path linked to it. This reduces RAM
usage because /tmp is a small tmpfs. Node and GNU make were prepared under /tmp;
the Node archive and the pinned godevccu simulator were checksum-verified.

## Limitations / Open Questions

- Real HAP/OpenCCU hardware was not available for this workspace validation.
  The device metadata and XML-RPC state in new tests are synthetic.
  The user subsequently reported that the local test package works on their HAP
  and creates the light entity. This is a user-reported hardware result, not an
  independently executed full hardware test matrix. The revised names still need
  the user's local hardware confirmation.
- External MASTER changes depend on existing configuration refresh support.
- A failed readback can leave the previous state marked uncertain until refresh.
- Local historical tags were absent; no Git remote operation was performed to
  reconstruct production code.
- openccu-loom support needs a corresponding daemon/client implementation and
  release outside these two checkouts.
- A release version and the future dependency pin remain maintainer decisions.

## User-visible Behavior

The HAP device gains one configuration light for system button illumination,
with an on/off toggle and no brightness slider. ON selects full illumination;
OFF disables it. A value configured as 0.5 remains editable in the Device
Configuration panel and appears as ON in the light. Unknown and unavailable
states remain distinct from OFF.
