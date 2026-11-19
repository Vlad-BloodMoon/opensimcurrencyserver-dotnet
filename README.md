# BloodMoon MoneyServer — OpenSimulator Currency Server

**Version 1.0.0 (2026)** · **OpenSimulator 0.9.3.1 / .NET 8** · **Tested with Firestorm 7.2.5 (81788)**

A maintained BloodMoon fork of [Manfred Aabye's OpenSimulator currency server](https://github.com/ManfredAabye/opensimcurrencyserver-dotnet), based on the original DTL/NSL MoneyServer work by **Fumi.Iseki and NSL**.

This fork focuses on **MySQL connection recovery**, **safer transfers**, and **automatic recovery of region-side currency sessions after a MoneyServer restart**. The original project's features remain available, subject to their existing configuration and limitations.

> **Download:** [Latest stable release — v1.0.0](https://github.com/Vlad-BloodMoon/opensimcurrencyserver-dotnet/releases/tag/v1.0.0)  
> **Previous test release:** [v1.0.0-beta.1](https://github.com/Vlad-BloodMoon/opensimcurrencyserver-dotnet/releases/tag/v1.0.0-beta.1)  
> **Region installation guide:** [docs/REGION_INSTALLATION.md](docs/REGION_INSTALLATION.md)

## What changed in BloodMoon v1.0.0?

### 1. MoneyServer: recovery from lost MySQL connections

- Checks pooled database connections before reusing them.
- Attempts to reconnect when a MySQL connection is stale.
- Avoids blindly replaying financial operations when the result of a failed database request is unknown.

### 2. MoneyServer: safer transfers

- Processes transfers inside a database transaction.
- Uses transaction-status and account-row locking to help guard against concurrent or duplicate processing.
- Improves consistency compared with separate, non-atomic debit/credit steps.

### 3. Regions: automatic restoration of avatar currency sessions

After restarting MoneyServer, an avatar who has remained connected to a region may otherwise receive a session-authentication failure until teleporting or logging in again.

The updated `OpenSim.Modules.Currency.dll` detects **explicit MoneyServer session rejections**, re-registers the same connected avatar, and retries the rejected operation **once**. It deliberately does **not** replay an operation following a timeout or an ambiguous network failure.

**Both the MoneyServer-side DLLs and the region-side Currency DLL must be updated** to use the complete set of fixes.

## Compatibility and test status

| Component | Verified environment |
| --- | --- |
| OpenSimulator | BloodMoon OpenSimulator 0.9.3.1 |
| .NET | .NET 8 |
| Viewer | Firestorm **7.2.5 (81788)** |
| Database | MySQL (dedicated test credentials against the currency database) |
| Build | GitHub Actions, Ubuntu 24.04, Release configuration |

**Successful functional checks:**

- Restarting the **MoneyServer** while an avatar stays logged into the same region; balance refresh, avatar-to-avatar transfer, and object purchase continue to work without teleport or relog.
- Closing the dedicated test MySQL user's **89 database connections**, without restarting MySQL, MoneyServer, or the region; MoneyServer logs `Reconnecting database`, and subsequent balance refresh, transfers, and object purchases succeed.

**Not yet fully validated:** a real MySQL service restart, a connection failure midway through a transaction, high-concurrency/fault-injection scenarios, and every script-payment path. A release designation does not guarantee the absence of financial edge cases. Test your own deployment and back up the currency database before upgrading.

## Download and install

Download the appropriate archive from the [v1.0.0 release page](https://github.com/Vlad-BloodMoon/opensimcurrencyserver-dotnet/releases/tag/v1.0.0):

| Archive | Contents | Install on |
| --- | --- | --- |
| `MoneyServer-1.0.0-Complete.zip` | All three DLLs and documentation | Both server and region hosts |
| `MoneyServer-1.0.0-Server.zip` | `MoneyServer.dll`, `OpenSim.Data.MySQL.MySQLMoneyDataWrapper.dll`, configuration example and guide | MoneyServer host |
| `MoneyServer-1.0.0-Regions.zip` | `OpenSim.Modules.Currency.dll` and guide | Each region simulator |
| `MoneyServer-1.0.0-SourceModules.zip` | Source trees for the three modules | Developers |
| `SHA256SUMS.txt` | Release archive checksums | Verification |

These are **modules for an existing compatible OpenSimulator installation**, not a complete standalone simulator or grid.

### MoneyServer host

1. Back up the **currency database**, existing DLLs, and `MoneyServer.ini`.
2. Stop the **MoneyServer process** (do not stop MySQL or ROBUST just to replace these DLLs).
3. Replace `MoneyServer.dll` and `OpenSim.Data.MySQL.MySQLMoneyDataWrapper.dll` in the MoneyServer `bin/` directory.
4. **Keep your existing `MoneyServer.ini`**; do not overwrite real credentials or keys with sample values.
5. Start MoneyServer and check its console/logs for database connection errors.

The server endpoint and MySQL details are configured in `MoneyServer.ini`. An example is available in [bin/MoneyServer.ini.example](bin/MoneyServer.ini.example). **Do not publish your production database password, API keys, or money-script access key.**

### OpenSimulator region hosts

1. Back up the old `OpenSim.Modules.Currency.dll` and affected configuration files.
2. Stop the relevant region process.
3. Copy the new `OpenSim.Modules.Currency.dll` into that region's `bin/` directory.
4. Ensure the **active** region configuration selects `DTLNSLMoneyModule`.
5. Restart the region and test the balance, avatar payments, and object purchases.

The following is a **sample** for the region's active `[Economy]` section, typically in `OpenSim.ini` or an included INI file:

```ini
[Economy]
    SellEnabled = true
    EconomyModule = DTLNSLMoneyModule

    ; MoneyServer endpoint — adjust host/protocol/port for your grid
    CurrencyServer = "${Const|BaseURL}:8008/"

    ; ROBUST user service endpoint — adjust for your grid
    UserServer = "${Const|BaseURL}:8002/"

    ; Original sample setting; review certificate validation for your deployment
    CheckServerCert = false

    ; Optional economy policy settings
    PriceUpload = 0
    PriceGroupCreate = 0
```

**Important:** `${Const|BaseURL}` must point to the appropriate **grid-service host**. If it points to the region host instead, replace these URLs with the actual reachable MoneyServer and ROBUST addresses. Avoid enabling `BetaGridLikeMoneyModule` at the same time. `MeshModelAllowTextureToInventory` is an independent mesh setting and is not required for the currency module.

See the [full region installation and configuration guide](docs/REGION_INSTALLATION.md), including notes on `OpenSimDefaults.ini`, certificate checking, and rollout validation. The upstream source also includes [bin/OpenSim.ini.sample](bin/OpenSim.ini.sample).

### Recommended grid startup order

**ROBUST → MoneyServer → region simulators**

The new region module helps recover existing avatar sessions after an ordinary MoneyServer restart, without restarting the region. It does not eliminate the need to validate your deployment's startup and network settings.

## Building from source

For a matching custom OpenSimulator build, include this repository's `addon-modules/` source trees in the OpenSimulator source directory, regenerate the solution, and build with the appropriate .NET SDK:

```bash
./runprebuild.sh
dotnet build --configuration Release OpenSim.sln
```

See [BUILDING_Money.md](BUILDING_Money.md) for the original compilation guidance. The provided release DLLs were built against the [BloodMoon OpenSimulator fork](https://github.com/Vlad-BloodMoon/opensim), branch `release/bloodmoon-webrtc.3`. Compatibility with other OpenSimulator forks and runtime combinations should be checked separately.

## Upstream, acknowledgements and contribution

This repository is a **fork**, not the original MoneyServer project:

- **Upstream maintainer:** [Manfred Aabye — opensimcurrencyserver-dotnet](https://github.com/ManfredAabye/opensimcurrencyserver-dotnet).
- **Original DTL/NSL MoneyServer:** Fumi.Iseki and NSL ([historical project reference](http://www.nsl.tuis.ac.jp/)).
- **BloodMoon modifications, testing and packaging:** [Vlad-BloodMoon](https://github.com/Vlad-BloodMoon).

We appreciate the work of the original contributors. The reconnect and session-recovery fixes have been prepared as separate contributions to facilitate upstream review.

For project licensing, see [LICENSE](LICENSE). Please report reproducible issues, ideally with redacted logs, OpenSimulator version, .NET version and relevant currency-module configuration. **Never include passwords, API keys, or private account data in reports.**
