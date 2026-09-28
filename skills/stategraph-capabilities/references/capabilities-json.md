# Capabilities JSON and the whoami table

## Raw JSON shape

The object that `--capabilities-json` and `--capabilities-file` take, that `stategraph whoami --format json` and `stategraph capabilities default show --format json` return, and that a group rule's `grant` holds. This value grants nothing:

```json
{
  "access-token-create": false,
  "access-token-refresh": false,
  "admin": [],
  "commit": { "modified": [], "pulled-in": [] },
  "preview": { "modified": [], "pulled-in": [] },
  "sudo": [],
  "users-manage": []
}
```

- `preview` is plan, `commit` is apply.
- `admin`, `sudo`, `users-manage` are lists of rule values: an id, a prefix ending in `*`, `*` alone, or a leading `!` to refuse.
- `preview.modified`, `preview.pulled-in`, `commit.modified`, `commit.pulled-in` are lists of `{"tenant": RULE, "states": [{"state": RULE, "addresses": [PATTERN, ...]}]}`.
- An empty `addresses` list for a state denies that state.

## Flag to JSON

Each row is the `capabilities` a token created with those flags reports in `whoami --format json`.

| Flags | Result |
|-------|--------|
| none | The creating session's capabilities, unchanged. |
| `--admin` | `"admin": ["*"]` |
| `--admin --admin-tenant T` | `"admin": ["T"]` |
| `--admin --admin-tenant '!T'` | `"admin": ["*", "!T"]` |
| `--plan` | `"preview": {"modified": [{"tenant": "*", "states": [{"state": "*", "addresses": ["*"]}]}], "pulled-in": [same]}` |
| `--plan --plan-tenant T` | As `--plan`, with `"tenant": "T"` in both lists. |
| `--plan --plan-modified 'S=*'` | `"preview.modified": [{"tenant": "*", "states": [{"state": "S", "addresses": ["*"]}]}]`, `pulled-in` unrestricted. |
| `--plan --plan-modified 'S=module.db.*' --plan-pulled-in 'S=*'` | `modified` addresses `["module.db.*"]` in S, `pulled-in` state S only. |
| `--plan --plan-modified '*=*' --plan-modified 'S=!*'` | `modified` states `[{"state": "*", "addresses": ["*"]}, {"state": "S", "addresses": []}]`: every state except S. |
| `--apply ...` | As `--plan ...`, under `commit`. |
| `--access-token-create --access-token-refresh` | Both booleans `true`. |
| `--users-manage`, `--sudo-user U` | Clipped to the creating session. From an installation-admin session whose `users-manage` and `sudo` lists are empty, both come back empty. Check the token with `whoami` before handing it out. |

`--plan-modified` alone implies `--plan`; `--apply-modified` alone implies `--apply`. Write the pair anyway.

`capabilities group create` scopes every grant to `--tenant`: `--plan` in a rule gives `"tenant": "TENANT_ID"` in both `preview` lists, and `--apply --apply-modified 'S=*'` gives `commit.modified` state S in that tenant.

## Reading the whoami table

```text
capabilities
  admin                                         *
  access-token-create                           true
  plan
    modified
      * / de783642-eaaf-4965-80f7-7428b74ac71d  *
    pulled-in
      * / *                                     *
  apply
    modified                                    (none)
```

- `admin *`: installation-wide admin. `admin (none)`: no admin. `admin T`: admin of tenant T.
- Under `plan` and `apply`, each row is `TENANT / STATE` on the left and the address pattern on the right. `* / SID  *` is every tenant, one state, every address. `* / *  *` is unrestricted.
- `(none)` under `modified` means the capability is not granted, unless `admin` is set.

## Installation default

The default shipped for new users grants plan and apply everywhere plus token create and refresh, and no admin:

```json
{
  "access-token-create": true,
  "access-token-refresh": true,
  "admin": [],
  "commit": { "modified": [{"states": [{"addresses": ["*"], "state": "*"}], "tenant": "*"}],
              "pulled-in": [{"states": [{"addresses": ["*"], "state": "*"}], "tenant": "*"}] },
  "preview": { "modified": [{"states": [{"addresses": ["*"], "state": "*"}], "tenant": "*"}],
               "pulled-in": [{"states": [{"addresses": ["*"], "state": "*"}], "tenant": "*"}] },
  "sudo": [],
  "users-manage": []
}
```

Change one field without losing the rest:

```bash
stategraph capabilities default show --format json > default.json
# edit default.json
stategraph capabilities default set --capabilities-file default.json
```
