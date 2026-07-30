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

## Design decisions

1. Use Hubitat’s built-in `hubitat / Generic Component Button Controller` child driver. The similarly named `Generic Component Button` driver is intended to receive events from a parent and does not provide the user-facing push command needed here. The Button Controller driver delegates button pushes to the parent through `componentPush(child, button)`.
2. Child network IDs use the parent network ID, a fixed saved-code marker, and a stable digest of the exact saved-code name. Do not use a sanitized name as the identity: punctuation, Unicode, case, and collisions make it unsafe. The display label remains the original saved-code name.
3. Store the exact saved-code name in child data (for example, `codeName`) in addition to encoding it in the network ID. Reconciliation must verify both the managed ID prefix and this data before using a child.
4. Renaming recreates the child. The name-derived identity necessarily changes, and child network IDs should be treated as immutable. The replacement retains the saved-code association through the new `codeName` value and label. This is a documented consequence for dashboard tiles and automations that reference the old child.
5. Each child exposes one button. The parent ignores the button number passed to `componentPush` and sends the child’s associated code once through `sendSavedCode`; held, released, and double-tapped behavior are out of scope.
6. The System Manager app needs no child-specific UI or synchronization changes. It continues to manage the parent’s saved-code map; parent reconciliation owns the child devices.

The decisions follow Hubitat’s [Parent/Child Drivers documentation](https://docs2.hubitat.com/en/developer/driver/parent-child-drivers) and official [Generic Component Parent Demo](https://github.com/hubitat/HubitatPublic/blob/master/examples/drivers/genericComponentParentDemo.groovy). The distinction between the built-in Button and Button Controller behavior is also documented in the [Hubitat community discussion](https://community.hubitat.com/t/generic-component-central-scene-switch-and-generic-component-button-how-to-use/43600).

## Suggested implementation order

1. Add parent support for the built-in Generic Component Button Controller.
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
