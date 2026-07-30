# Saved-code child buttons

## Overview

Expose saved Broadlink IR/RF codes as child button devices beneath the existing Broadlink Remote parent device.

The parent remains responsible for Broadlink communication and code storage. Each saved code is represented by a child device using Hubitat’s Button capability, allowing the code to be triggered directly from dashboards without requiring a separate virtual button or a Rule Machine rule for every code.

This follows the parent/child pattern used by integrations such as Harmony: one parent device manages the integration, while individual child devices provide user-facing controls.

## Current behavior

- The Broadlink Remote driver stores codes as a name-to-hex-code map in `state.codes`.
- The device exposes a `savedCodes` attribute containing a readable list of code names.
- The System Manager app displays the names saved in the app.
- Dashboard users cannot directly invoke the parameterized `sendSavedCode(name, reps)` command.
- The current workaround is a virtual button plus Rule Machine or Button Controller.

## Proposed device model

```text
Broadlink Remote (parent)
├── Power (Generic Component Button)
├── Volume Up (Generic Component Button)
├── Volume Down (Generic Component Button)
└── Fan (Generic Component Button)
```

Each child corresponds to one saved code and is named from the saved-code name.

### Parent device

The parent continues to:

- Store and synchronize the saved-code map.
- Handle IR and RF transmission.
- Perform learning, importing, renaming, deleting, and clearing operations.
- Create, update, and remove child devices.
- Expose the existing `savedCodes` listing.

### Child devices

Each child should:

- Use the Hubitat Button capability.
- Represent exactly one saved code.
- Send its associated code when pushed.
- Delegate transmission to the parent rather than duplicating Broadlink logic.
- Be usable as a dashboard button without an intermediate virtual device.

The child’s button event should be a momentary action. It should not represent a persistent on/off state, since sending a remote command does not create a durable switch state.

## Child lifecycle

### Code saved or imported

When a code is added to the parent:

1. Store the name and hex code in `state.codes`.
2. Create the corresponding child if it does not exist.
3. Update the child metadata if it already exists.
4. Refresh the parent’s `savedCodes` attribute.

This must cover learned codes, imported codes, Pronto/LIRC imports, and codes synchronized from the System Manager app.

### Code renamed

When a code is renamed:

1. Preserve the code data.
2. Rename or recreate the corresponding child device.
3. Preserve the child’s association with the renamed code.
4. Refresh the parent’s code listing.

If Hubitat does not safely support renaming child device names or network IDs, delete and recreate the child with clear logging and documented behavior.

### Code deleted

Delete the corresponding child device and remove the code from the parent map.

### All codes cleared

Remove all code children and leave the parent device intact.

### Driver initialization or update

Reconcile child devices against `state.codes` so that existing installations recover from missed events and stale children are removed.

## Sending behavior

The child button should call the parent’s saved-code send path, equivalent to:

```groovy
parent.sendSavedCode(codeName)
```

The parent should continue deciding whether the code is IR or RF. RF codes do not use IR repetition behavior; the child button can use the normal single-send behavior.

## Dashboard behavior

Users would authorize the Broadlink child devices in EZ Dashboard and add the desired child buttons as tiles. Each tile would send its one associated code directly.

This avoids:

- One virtual button device per code.
- One Rule Machine custom action per code.
- Treating a momentary remote command as a fake switch state.
- Requiring dashboard users to enter command parameters.

Hubitat dashboard tiles still represent individual controls, so one tile per desired action is expected. The important distinction is that the controls are real child button devices managed by the Broadlink parent.

## Naming and identity

- Display names should use the saved-code names.
- Child network IDs must be deterministic and safely sanitized.
- Names containing punctuation, spaces, or Unicode characters must not produce invalid IDs.
- Duplicate names should follow the existing saved-code behavior; the child reconciliation logic must not silently associate a child with the wrong code.
- Code names should be HTML-escaped anywhere they are displayed in the app.

## Compatibility considerations

- Existing parent devices and saved codes must continue working after the driver update.
- Existing commands such as `sendSavedCode`, `push`, and `importCodes` must remain available.
- The System Manager app should continue to synchronize code data through the parent API.
- Child-device creation must be idempotent so repeated initialization does not create duplicates.
- A device with no saved codes should simply have no code children.

## Open implementation choices

1. Confirm the exact built-in Hubitat child driver name for a component button. If no suitable generic driver is available, provide a small local child driver implementing Button.
2. Decide whether child IDs are based on a sanitized code name or a stable hash of the code name.
3. Decide whether renaming updates the existing child device or recreates it.
4. Decide whether a child should expose only `pushed`, or also support optional held/double-tapped behavior in the future.
5. Decide whether the app should display child-device status or only the parent’s saved-code list.

## Suggested implementation order

1. Add a child button driver or confirm the built-in component button driver.
2. Add parent child-device reconciliation helpers.
3. Integrate reconciliation with add, import, rename, delete, clear, and initialize paths.
4. Add parent delegation from child button pushes.
5. Test IR, RF, imports, renames, deletions, app synchronization, driver updates, and EZ Dashboard tiles.
## Reference child driver

The installed ST_Anything `Child Switch` driver is useful as a reference for Hubitat’s parent/child structure:

<https://raw.githubusercontent.com/DanielOgorchock/ST_Anything/master/HubDuino/Drivers/child-switch.groovy>

It should not be used directly for this feature. It implements the Switch capability and delegates `on()`/`off()` through `parent.sendData(...)`. Our child should instead implement the Button capability and delegate `push()` to the Broadlink parent’s saved-code send method.

## Harmony integration reference

The Logitech Harmony parent driver is another useful reference for dynamic child-device creation:

<https://github.com/ogiewon/Hubitat/blob/master/Drivers/logitech-harmony-hub-parent.src/README.md>

Its README documents creating child devices for Harmony activities, synchronizing children from the parent, and using Hubitat’s built-in Generic Component drivers. The newer Harmony implementation uses Generic Component Switch for activities, partly for HomeKit compatibility. We should reuse the parent/child lifecycle approach while retaining Button semantics for Broadlink saved codes.

