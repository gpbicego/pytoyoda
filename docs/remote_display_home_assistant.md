# Home Assistant workaround for missing `remoteDisplay`

This branch provides a temporary workaround for a Toyota Connected Services API change that can cause the Toyota EU community integration for Home Assistant to fail with messages such as:

```text
No vehicles found for this account
Toyota refresh failed for all vehicles
````

The issue can occur even when authentication with Toyota succeeds.

## Cause

For some Toyota accounts, the `/v2/vehicle/guid` response no longer contains the `remoteDisplay` field.

In pytoyoda 5.2.0, the field is defined as:

```python
remote_display: Any | None = Field(alias="remoteDisplay")
```

Although the type allows `None`, with Pydantic this still means that the field itself is required to be present in the API response.

If Toyota omits `remoteDisplay`, validation of the vehicle response fails and pytoyoda can discard the vehicle, eventually resulting in:

```text
No vehicles found for this account
```

This branch changes the field to:

```python
remote_display: Any | None = Field(alias="remoteDisplay", default=None)
```

This makes the field genuinely optional: if `remoteDisplay` is absent from Toyota's response, its value becomes `None` instead of causing validation of the entire vehicle response to fail.

## Related issue

The problem is discussed in:

**pytoyoda/ha_toyota issue #360**

## Branch

Use:

```text
fix_remote_display_optional
```

This branch is based on upstream pytoyoda v5.2.0 and contains the `remoteDisplay` workaround.

It also contains a small packaging compatibility adjustment required to allow installation from Git inside the Home Assistant OS Python environment.

## Home Assistant installation

The workaround can be used with the Toyota EU community integration by changing its `manifest.json`.

The file is normally located at:

```text
/config/custom_components/toyota/manifest.json
```

Replace the normal pytoyoda requirement:

```json
"requirements": [
  "pytoyoda>=5.2.0,<6.0",
  "arrow"
],
```

with:

```json
"requirements": [
  "pytoyoda @ git+https://github.com/gpbicego/pytoyoda.git@fix_remote_display_optional",
  "arrow"
],
```

Do not change the Toyota integration version only because of this workaround.

For example, with Toyota integration v2.5.0:

```json
"version": "v2.5.0"
```

should remain unchanged.

Restart Home Assistant after modifying the manifest.

## Verify the installed package

From the Home Assistant terminal, run:

```bash
docker exec homeassistant python -c "import importlib.metadata as m; from pytoyoda.models.endpoints.vehicle_guid import VehicleGuidModel; f=VehicleGuidModel.model_fields['remote_display']; print('version:', m.version('pytoyoda')); print('required:', f.is_required()); print('default:', f.default)"
```

A correctly installed version of this branch should report:

```text
required: False
default: None
```

The exact development version will depend on the current commit.

For example:

```text
version: 5.2.0.postN.dev0+<commit>
required: False
default: None
```

## Verify Toyota operation

After restarting Home Assistant, check the logs with:

```bash
ha core logs | grep -Ei 'toyota|pytoyoda|No vehicles found|refresh failed|ValidationError'
```

If this API change was the cause of the problem, the following messages should no longer appear:

```text
No vehicles found for this account
Toyota refresh failed for all vehicles
```

and the Toyota vehicle entities should become available again.

## HAOS build compatibility

Installing the current pytoyoda source directly from Git inside Home Assistant OS can fail because the Home Assistant Python package environment currently provides Poetry Core 1.9.1.

Without the compatibility changes in this branch, installation may fail with an error similar to:

```text
The Poetry configuration is invalid:
- The fields ['authors', 'description', 'name'] are required in package mode.
```

This branch therefore contains additional `pyproject.toml` metadata compatible with the Poetry Core version available in the Home Assistant build environment.

These packaging changes are specific to making this temporary Git branch installable from Home Assistant OS and are separate from the actual Toyota API fix.

## Manual installation test

To test whether the branch can be built and installed inside the Home Assistant container:

```bash
docker exec homeassistant python -m uv pip install --system --upgrade "pytoyoda@git+https://github.com/gpbicego/pytoyoda.git@fix_remote_display_optional"
```

A successful installation should contain output similar to:

```text
Building pytoyoda @ git+https://github.com/gpbicego/pytoyoda.git@...
Built pytoyoda
Installed 1 package
```

The package version should be based on v5.2.0.

## Important notes

### Home Assistant updates

A Home Assistant Core update may recreate the container and reinstall Python dependencies.

Using the Git requirement in `manifest.json` ensures that Home Assistant can reinstall this branch instead of relying on a manual modification of files inside:

```text
/usr/local/lib/python3.14/site-packages/
```

### Toyota integration updates

Updating or redownloading the Toyota integration through HACS may replace its `manifest.json` and restore the official pytoyoda requirement.

After updating the Toyota integration, check whether the official pytoyoda release already includes the `remoteDisplay` fix.

If it does not, the Git requirement may need to be restored.

### Do not manually patch site-packages as a permanent solution

Changing:

```text
/usr/local/lib/python3.14/site-packages/pytoyoda/
```

directly is useful for testing, but those changes can disappear when Home Assistant recreates or updates its container.

Using this Git branch through the integration manifest is the persistent workaround.

## Status

This branch is intended only as a temporary workaround.

Once the `remoteDisplay` change is included in an official pytoyoda release, users should return to the normal released dependency, for example:

```json
"requirements": [
  "pytoyoda>=<fixed-version>,<next-major-version>",
  "arrow"
],
```

Using the official pytoyoda package is preferable once the fix has been released upstream.
