---
name: stategraph-capabilities
description: |
  Identity, access tokens, and capabilities in Stategraph.

  Use this skill for:
  - "who am I logged in as", "what can my token do", "show my capabilities"
  - "create an access token", "give CI a read-only key", "a plan-only token for state X", "an apply token for one state"
  - listing or revoking the current user's Stategraph access tokens
  - "default permissions for new users": show or replace the system-wide default capabilities
  - "map the IdP group X to plan rights": group rules that grant plan, apply, admin, or users-manage in a tenant
  - listing the tenants the current user belongs to

  Do not use this skill for:
  - IAM, RBAC, or permission questions about AWS, GCP, Azure, Kubernetes, GitHub, or any system other than Stategraph
  - cloud or Terraform provider credentials
  - configuring OIDC, OAuth, or password sign-in on the server
  - running plans or applies (stategraph-change) or read-only queries (stategraph-query)

tags:
  - stategraph
  - access-tokens
  - capabilities
  - identity
metadata:
  author: Stategraph
  version: "1.0"
---

# Stategraph capabilities skill

## Environment

- `STATEGRAPH_API_BASE` replaces `--api-base`. `STATEGRAPH_API_KEY` is the token the CLI authenticates with. Every command here needs both.
- `STATEGRAPH_TENANT_ID` replaces `--tenant` on the `capabilities group` commands and on `states list`.
- `stategraph whoami` is an alias for `stategraph user whoami`. `stategraph caps` is an alias for `stategraph capabilities`.
- Add `--format json` to parse output. `--format simple` prints tab-separated rows with no header.

## Capability model

- Capabilities: `admin`, `plan`, `apply`, `users-manage`, `sudo`, `access-token-create`, `access-token-refresh`. In JSON, `plan` is `preview` and `apply` is `commit`.
- There is no read capability. A token that grants nothing still runs `whoami`, `user tenants list`, `user access-tokens list`, `states list`, `states summary`, `states resources summary`, and `sql query`. A read-only token is a token that grants nothing.
- `plan` and `apply` are scoped per tenant, per state, and per resource address pattern: `modified` limits what a change may touch, `pulled-in` limits what it may pull in as dependencies. Without a `--*-tenant` or `--*-modified` flag, `--plan` and `--apply` reach every tenant and state.
- A token never exceeds the session that creates it. With no capability flags, the token inherits the session's capabilities in full.
- `admin: ["*"]` in `whoami` is installation-wide admin. An admin session shows `plan` and `apply` as `(none)` in the table; admin covers them.
- Tenant and user values: an id, a prefix ending in `*`, `*` alone for everything, a leading `!` to refuse. State patterns are `STATE_ID=PATTERN`: `sid=*` for the whole state, `sid=module.db.*` for a prefix, `sid=!*` to deny. Repeated flags add up.
- Read `references/capabilities-json.md` when you must write or read raw capabilities JSON, or interpret the `whoami` table.

## 1. Who am I, what can this session do

```bash
stategraph whoami --format json
```

Read `capabilities`: `admin: ["*"]` is full access. `preview.modified` and `commit.modified` list what plan and apply may change, per tenant and state; an empty list with no admin means no plan or apply. `access-token-create: false` means this session cannot create tokens: report that and stop before any create.

## 2. List tenants

```bash
stategraph user tenants list --format json
```

## 3. Create an access token

Output is `{"token": "..."}`. The value is shown once and cannot be retrieved later: hand it over or export it right away.

```bash
# Inherit the session's capabilities (no capability flags)
stategraph user access-tokens create --name NAME --format json

# Read-only: grants nothing, can list and query, cannot plan or apply
stategraph user access-tokens create --name ci-reader --format json --capabilities-json \
  '{"access-token-create":false,"access-token-refresh":false,"admin":[],"commit":{"modified":[],"pulled-in":[]},"preview":{"modified":[],"pulled-in":[]},"sudo":[],"users-manage":[]}'

# Plan-only, one state
stategraph user access-tokens create --name ci-plan --format json --plan --plan-modified 'STATE_ID=*'

# Apply-only, one state
stategraph user access-tokens create --name ci-apply --format json --apply --apply-modified 'STATE_ID=*'

# Plan and apply on one state: give both pairs. Whole tenant instead of one state: --plan --plan-tenant TENANT_ID
```

`--capabilities-json` and the capability flags are mutually exclusive. `--capabilities-file FILE` reads the same JSON from a file. `--admin` grants installation-wide admin; `--admin --admin-tenant TENANT_ID` grants admin of one tenant.

Verify the new token from a subshell, so the session's key stays in place:

```bash
TOKEN=$(stategraph user access-tokens create --name ci-reader --format json --capabilities-json '...' | jq -r .token)
( export STATEGRAPH_API_KEY="$TOKEN"; stategraph whoami --format json; stategraph states list --format json )
```

A plan or apply the token does not cover fails with `Forbidden: you are not authorized for this request:` and the reason: `your access token does not include the plan capability`, `... the apply capability`, or `no access to state STATE_ID`. A plan-only token passes `stategraph tf plan` and fails `stategraph tf apply` at the apply step, after printing the plan.

## 4. List and revoke tokens

```bash
stategraph user access-tokens list --format json
stategraph user access-tokens delete --token-id TOKEN_ID
```

`list` shows `id`, `name`, `created_at`, `owner_id`, `owner_name`, `owner_type`, never the token value. Id by name: `stategraph user access-tokens list --format simple | awk -F'\t' '$2=="NAME"{print $1}'`. `delete` prints `revoked true`; the token then gets `Unauthorized` on every call. An unknown id prints `Access token not found`.

## 5. Default capabilities for new users (installation admin)

```bash
stategraph capabilities default show --format json
stategraph capabilities default set --plan
```

`set` replaces every default: a capability not passed is no longer granted. It takes the capability flags of token creation and no `--name`. To change one capability, save `default show --format json` to a file, edit it, and pass `--capabilities-file FILE`. With no flags it fails with `No capabilities specified`. A new default applies to users created afterwards only.

## 6. Group rules: IdP group to capabilities (tenant admin)

```bash
stategraph capabilities group create '{"group":"platform-eng"}' --tenant TENANT_ID --plan --description "platform-eng may plan" --format json
stategraph capabilities group list --tenant TENANT_ID --format json
stategraph capabilities group delete RULE_ID --tenant TENANT_ID
```

The condition is a positional JSON argument: `{"group":"NAME"}`, `{"group":"eng-*"}`, `{"any":[{"group":"a"},{"group":"b"}]}`, or `{"all":[...]}`. Grants are scoped to `--tenant`. Flags: `--admin`, `--plan`, `--plan-modified`, `--plan-pulled-in`, `--apply`, `--apply-modified`, `--apply-pulled-in`, `--users-manage`, `--capabilities-json`, `--capabilities-file`, `--description`. There are no `--*-tenant` or `--sudo-user` flags here. `create` prints the rule `id`; `list --format json` returns `rules[]` with `id`, `condition`, `grant`, `description`, `created_at`, `created_by`. A rule takes effect at each member's next sign-in through the provider that reports groups.

## Failure handling

Report the message once and stop. Do not retry with a different flag.

- Exit 124: unknown flag or missing required option, for example `--name`. `--token-id` must be a UUID.
- Exit 1, `Forbidden: you are not authorized for this request:` plus the missing capability: the session lacks it. `capabilities default` needs installation admin, `capabilities group` needs admin of that tenant, `access-tokens create` needs `access-token-create`.
- `Unauthorized`: the key is invalid or revoked.
- `--capabilities-json cannot be combined with individual capability flags`: use one or the other.
- `Unprocessable: GRANT_BEYOND_TENANT`: a group rule cannot grant unscoped admin, sudo, or token create and refresh.
- `Bad request: INVALID_REQUEST_BODY: expected a single-operator-key condition object`: the condition key is not `group`, `any`, or `all`.
- `No group rule with id ...`: already deleted, or another tenant's rule.
- `Error: STATEGRAPH_API_KEY not set.`: export the key first.

## Response contract

Report the command run, the identity or token name, the capabilities the token holds as `whoami --format json` under that token shows them, the token id for later revocation, and the token value once, only when the user asked for a token.
