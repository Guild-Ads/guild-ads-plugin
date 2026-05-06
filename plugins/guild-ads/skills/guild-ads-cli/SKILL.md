---
name: guild-ads-cli
description: Use whenever the user mentions Guild Ads — the indie-app ad network — or wants to install or use the `guild-ads` CLI. Triggers include installing or authenticating the CLI, registering an app, generating an SDK token, listing or booking ad slots, managing campaigns, viewing performance, redeeming a promo code, setting up payouts, or any operation against the Guild Ads platform. Always use this skill before either guild-ads-publisher-setup or guild-ads-advertiser-setup so the CLI is installed and the user is signed in.
---

# Guild Ads CLI

[Guild Ads](https://guildads.com) is an ad network for indie apps. The `guild-ads` CLI ([npm](https://www.npmjs.com/package/guild-ads)) gives full dashboard parity from the terminal — registering apps, generating SDK tokens, managing campaigns, booking ad slots, viewing performance, and managing publisher payouts. This skill is the foundation; the publisher and advertiser setup skills assume the CLI is installed and authenticated.

## Install the CLI

The CLI is published to npm as `guild-ads`. Node.js 20 or newer is required.

```bash
npm install -g guild-ads
```

If the user does not have Node 20+, install it first (Homebrew, fnm, nvm, or volta — pick what fits the user's setup; do not assume).

After install, verify:

```bash
guild-ads --version
```

If the binary is missing from `$PATH` after a global install, the user's npm prefix may not be on `$PATH`. Show them `npm config get prefix` and add `<prefix>/bin` to `$PATH`.

## Authenticate

Credentials are stored in `~/.config/guild-ads/auth.json` with mode `0600`. Check whether the user is already signed in:

```bash
test -f "$HOME/.config/guild-ads/auth.json" && guild-ads auth whoami --json
```

If `whoami` succeeds, the session is valid. If it fails with exit code 2, the session expired — re-run login.

### Existing user

```bash
guild-ads auth login --email "$EMAIL" --password "$PASSWORD" --json
```

Both flags are required for non-interactive use. With a TTY available, `guild-ads auth login` (no flags) prompts interactively — fine for human-driven sessions.

### New user

```bash
guild-ads auth signup --email "$EMAIL" --password "$PASSWORD" --json
```

After signup the user must click a confirmation link sent to the email address. Until they confirm, login will fail. Tell the user to check their inbox; offer `guild-ads auth resend-confirmation --email "$EMAIL"` if the email did not arrive.

If the user does not yet have an account and cannot describe their intent, ask whether they want to set up as a **publisher** (monetize an app), an **advertiser** (promote an app), or **both** before signing them up — the next skill to invoke depends on the answer.

## JSON output

Every command supports `--json` for machine-readable output. For agent sessions, set the env var once and forget it:

```bash
export GUILD_ADS_JSON=1
```

When `--json` is set:
- **stdout** is a single JSON value — parseable with `jq` or any JSON library
- **stderr** carries human-readable progress and errors — never parse it

## Discover the full command surface

The CLI publishes its own command tree as JSON:

```bash
guild-ads commands --json
```

Returns an array of `{ command, description, flags }` objects. Use this to enumerate available commands without reading documentation. `guild-ads <command> --help` prints help for a single command.

## Exit codes

| Code | Meaning | Action |
|------|---------|--------|
| 0 | Success | — |
| 1 | Invalid arguments / validation failure | Re-read the help; check flag values |
| 2 | Not logged in / session expired | Run `guild-ads auth login` |
| 3 | Network or API unreachable | Retry; check `https://guildads.com` is up |
| 4 | Resource not found | Verify IDs |
| 5 | Stripe or browser flow timed out | Re-run; check status with the matching `* status` command |

Always check `$?` after each call. Code 2 is the most common after long-running sessions because tokens refresh automatically but a stale machine can drift.

## Common operations

| Goal | Command |
|------|---------|
| Register an iOS app | `guild-ads apps create --name <name> --bundle-id <id> --store-url <url> --platform ios --json` |
| List my apps | `guild-ads apps list --json` |
| Generate an SDK token | `guild-ads tokens create --app <appId> --name <label> --json` (token shown once) |
| List my campaigns | `guild-ads campaigns list --json` |
| Create a campaign | `guild-ads campaigns create --app <appId> --name <name> --headline <text> --url <storeUrl> --json` |
| List bookable slots | `guild-ads slots list --json` |
| Book a slot with credits | `guild-ads book create --campaign <id> --slot <id> --percentage <n> --apply-credits --json` |
| Book a slot with cash | `guild-ads book create --campaign <id> --slot <id> --percentage <n> --json` (opens Stripe Checkout) |
| Show account balance | `guild-ads balance show --json` |
| Show activity | `guild-ads activity --json` |
| Redeem a promo code | `guild-ads promo redeem <CODE> --json` |
| Set up payouts (publisher) | `guild-ads payouts setup --json` (opens Stripe Connect) |
| Set up a payment method | `guild-ads payment-methods setup --json` (opens Stripe Checkout) |
| View ad performance | `guild-ads ad-performance --app <appId> --weeks <n> --json` |
| View publisher earnings | `guild-ads publisher-performance --app <appId> --weeks <n> --json` |

## Semi-interactive commands

A few commands open a browser to complete a Stripe flow and then poll until the result appears in the database:

- `payment-methods setup` — saves a card via Stripe Checkout
- `payouts setup` — Stripe Connect onboarding for publisher payouts
- `book create` — when cash is due (not fully covered by credits), opens Stripe Checkout

When running on a machine where the user can complete a browser flow, these are appropriate. For fully headless contexts (CI, remote agents), arrange for the payment method or payout account to be set up beforehand. Exit code 5 means the flow timed out — re-run and check the matching `* status` command for current state.

## Error handling pattern

```bash
OUTPUT=$(guild-ads apps list --json 2>/dev/null)
EXIT_CODE=$?

case $EXIT_CODE in
  0) echo "$OUTPUT" | jq '.[] | .appId' ;;
  2) guild-ads auth login --email "$EMAIL" --password "$PASSWORD" --json
     guild-ads apps list --json ;;
  *) echo "Failed with exit $EXIT_CODE" >&2; exit $EXIT_CODE ;;
esac
```

Redirect stderr to `/dev/null` (or capture it separately) so the JSON stream stays clean.

## When to hand off

Once the CLI is installed and the user is authenticated:
- If they want to monetize an app → switch to **guild-ads-publisher-setup**
- If they want to promote an app → switch to **guild-ads-advertiser-setup**
- If they want both → run publisher-setup first (so the iOS SDK is integrated and shipping), then advertiser-setup

If the user asks an open-ended question like "set me up on Guild Ads," ask which role they want before proceeding. Do not guess.
