# SHIFT
iphone blocker
# Shift Mode MDM Toolkit

A minimal, personal-use MDM setup for shifting a supervised iOS device between a focused mode (blocking selected apps) and a normal mode. The system relies entirely on supervised MDM controls—no FamilyControls or ManagedSettings entitlements are required.

## Repository layout

```
/server      FastAPI MDM server, policy store, and CLI toggle
/profiles    Signed configuration profiles and supporting certificates
/scripts     macOS bootstrap helper for supervision and enrollment
```

## 1. Prepare Python environment

```
cd server
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Start the server with:

```
uvicorn main:app --reload
```

Set an admin token (used by the CLI and admin endpoints) before launching:

```
export MDM_ADMIN_TOKEN="super-secret-token"
uvicorn main:app --host 0.0.0.0 --port 8000
```

## 2. Customize the blacklist policy

* Update `server/policy.json` with any bundle identifiers you want to block.
* Edit `profiles/shift_payload.plist` (search for `blacklistedAppBundleIDs`). Inline comments explain how to add or remove bundle IDs; keep it synchronized with `policy.json`.
* After editing the payload, re-sign the profile to regenerate the binary `.mobileconfig`:

```
openssl smime -sign -in profiles/shift_payload.plist \
  -out profiles/shift.mobileconfig \
  -signer profiles/certs/shiftmode.crt.pem \
  -inkey profiles/certs/shiftmode.key.pem -outform der -nodetach
```

Repeat the process for `profiles/unshift_payload.plist` if you change identifiers or descriptions (the unshift profile simply clears the blacklist array).

## 3. Supervise and enroll the device (macOS)

1. Install Apple Configurator 2 from the Mac App Store.
2. Connect the target iOS device via USB and use Configurator to **Prepare** it in Supervised mode (disable activation lock for personal devices). The device must trust the Mac running Configurator.
3. With the device still connected, run the bootstrap script to install the trust anchor and enrollment profile, substituting your server URL if it is not running on the same Mac:

```
./scripts/bootstrap_supervision.sh http://<your-ip>:8000
```

The script waits for a supervised device, installs the CA trust profile, and installs an enrollment profile that points to the FastAPI server. It finally triggers a quick MDM sync via `cfgutil`.

If you need to regenerate the trust or enrollment profiles (for example, after issuing a new certificate), delete the files in `profiles/` and re-run the OpenSSL commands in `profiles/shift_payload.plist` and `profiles/unshift_payload.plist` comments, then re-run the bootstrap script.

## 4. Toggling Shift Mode

Use the CLI to queue MDM commands for a specific device UDID:

```
python server/toggle_cli.py shift <DEVICE_UDID>
python server/toggle_cli.py unshift <DEVICE_UDID>
python server/toggle_cli.py remove <DEVICE_UDID>
```

The CLI respects the `MDM_ADMIN_TOKEN` and `SHIFT_MDM_SERVER` environment variables for authentication and server discovery.

Because this minimal MDM does not implement push notifications, poll the server after queuing a command. You can do this from macOS with:

```
cfgutil mdm -k DeviceInformation
```

or by opening the Settings app on the device (which triggers a background check-in).

## 5. API summary

* `GET /profiles/shift.mobileconfig` — Signed profile with the blacklist applied.
* `GET /profiles/unshift.mobileconfig` — Signed profile that clears the blacklist.
* `POST /admin/shift` — Queue an `InstallProfile` command with the shift payload (requires Bearer token).
* `POST /admin/unshift` — Queue an `InstallProfile` with the clear payload plus a `RemoveProfile` for the blacklist identifier.
* `POST /admin/remove` — Only queue the removal command.
* `POST /mdm/checkin` and `POST /mdm/connect` — Minimal MDM endpoints for the supervised device.

## 6. Notes and caveats

* The included certificates are self-signed for local testing. For long-term use, replace them with your own CA and re-sign all profiles.
* Push notifications/APNs are not implemented; manual polling is required after queuing commands.
* The repository is intentionally minimal and targets a single personal device. Extend the JSON policy or database layer if you need multi-device management.
