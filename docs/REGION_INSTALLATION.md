# Region-side installation and economy configuration

This guide covers the **OpenSimulator region** part of the BloodMoon MoneyServer modules.
It is intended for existing OpenSimulator installations using the DTL/NSL MoneyServer module.

## What to install

The GitHub release contains:

- `server/MoneyServer.dll` and `server/OpenSim.Data.MySQL.MySQLMoneyDataWrapper.dll`: install on the **MoneyServer host**.
- `regions/OpenSim.Modules.Currency.dll`: install in the **`bin/` directory of each region simulator** using this economy module.

The DLLs must match your running OpenSimulator version and .NET runtime. The BloodMoon
release was compiled and tested against BloodMoon OpenSimulator 0.9.3.1 / .NET 8.
It is not a standalone OpenSimulator distribution.

Stop the corresponding process before replacing its DLL. Back up the old binaries
and your configuration files first.

## Region configuration

Add or verify these settings in the active `[Economy]` section of your region's
`OpenSim.ini`, or in the appropriate included configuration file
(e.g. `OpenSimDefaults.ini` if you maintain it yourself).
Do **not** edit a template that the running simulator does not load.

```ini
[Economy]
    SellEnabled = true
    EconomyModule = DTLNSLMoneyModule
    CurrencyServer = "${Const|BaseURL}:8008/"
    UserServer = "${Const|BaseURL}:8002/"
    CheckServerCert = false

    PriceUpload = 0
    PriceGroupCreate = 0
    MeshModelAllowTextureToInventory = true
```

- `EconomyModule`: selects this third-party region economy module.
- `CurrencyServer`: MoneyServer endpoint (port **8008** in this example).
- `UserServer`: the **ROBUST user service** endpoint (port **8002** in this example).
- `CheckServerCert`: TLS certificate checking. The example uses `false` to
  match the original sample; configure secure HTTPS validation where supported
  and appropriate in your deployment.
- `SellEnabled`: enables selling objects; `PriceUpload` and
  `PriceGroupCreate` are optional grid policy values.
- `MeshModelAllowTextureToInventory`: independent OpenSimulator mesh-upload
  setting; it is **not required** for MoneyServer reconnection.

**Important:** `${Const|BaseURL}` must resolve to the proper **grid services host**
for both URLs. If a region runs on a separate machine, or that constant resolves to
the region host rather than the ROBUST/MoneyServer host, replace these values with
the actual reachable service URLs (for example
`https://grid.example.org:8008/` and `https://grid.example.org:8002/`,
where those endpoints are genuinely configured). Use your own grid hostnames,
ports and protocol. Do not copy the example literally without checking.

If `EconomyModule = BetaGridLikeMoneyModule` is also active in another
configuration layer, disable that override or ensure the final effective value
is `DTLNSLMoneyModule`. Check your configuration precedence and restart
the region process for a DLL replacement.

## MoneyServer startup

Keep the existing `MoneyServer.ini` settings, including your MySQL details.
The MoneyServer port must match the region's `CurrencyServer` URL.
Typical grid startup order is **ROBUST → MoneyServer → regions**.

Our new region module can recover an avatar's session after an ordinary
MoneyServer restart without requiring them to teleport or reconnect.
The MySQL server-side fix can restore stale database connections.

## Validation after deployment

1. Confirm the MoneyServer starts and connects to MySQL.
2. Confirm the region loads `DTLNSLMoneyModule`.
3. Check the avatar's balance.
4. Make a small avatar-to-avatar payment and verify both balances.
5. Test an object purchase and verify balances.
6. If practical, restart **only MoneyServer**, leaving the avatar logged in,
   and confirm the balance and purchases still work.

Do not perform database restart or in-flight interruption tests against a live
production currency database without a separate controlled plan.

## Upstream documentation

The original project already includes a detailed `[Economy]` sample at
`bin/OpenSim.ini.sample`, but its `BUILDING_Money.md` focuses on general
OpenSimulator setup rather than the region currency settings.
This document makes the required module configuration explicit.
