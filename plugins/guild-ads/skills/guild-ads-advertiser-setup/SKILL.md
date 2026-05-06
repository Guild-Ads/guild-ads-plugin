---
name: guild-ads-advertiser-setup
description: Use when the user wants to advertise their app on Guild Ads — promoting it to other indie apps in the network. Triggers include "advertise on Guild Ads," "promote my app," "create a Guild Ads campaign," "book an ad slot," "I want to run ads on Guild Ads," "set up an advertiser account," "buy ad inventory," or any flow that ends with the user's app being shown to other apps' users. Covers signing up, registering the app being advertised, creating a campaign with creative copy, understanding the weekly auction pricing model, booking ad slots one-time or as a recurring weekly plan, and paying via Stripe Checkout or by applying credits. Assumes guild-ads-cli has already been used to install and authenticate the CLI.
---

# Guild Ads Advertiser Setup

Walks an agent through everything needed for a developer to start advertising their app on Guild Ads. The end state: the user's app has a confirmed booking on a current or future week, paid for either by card or by applying credits.

**Prerequisite:** the `guild-ads` CLI is installed and the user is signed in (see the `guild-ads-cli` skill). If not, switch to that skill first.

## How Guild Ads pricing works (read this before booking)

Important context before walking through booking:

- The network sells a **share of weekly impressions**, not a fixed CPM. Each week has a network-wide price, set automatically based on the previous week's demand.
- Each advertiser pays for a percentage of that week's impressions (e.g. 10% of all impressions network-wide). The minimum percentage is 1%.
- The advertiser is billed up-front for the week. Bookings cannot exceed `weekly_budget_cap_cents`, which acts as a safety ceiling.
- **Delivery is renormalized.** If the week is undersold (advertisers paid for 80% total), each advertiser delivers proportionally more so the inventory always fills to 100%. Advertisers do not pay for the bonus impressions.
- A booking can be **one-time** (just this week) or a **recurring weekly plan** (renews every week until paused or canceled). Recurring plans show up on the user's dashboard with edit/pause/cancel controls.

When the user asks "how much does it cost?" the honest answer is "depends on the week's price — let's check." Do not invent CPMs or fixed rates.

## The shape of the flow

1. Confirm the account exists and is signed in
2. Register the app being advertised (or look it up if already registered)
3. Create a campaign with creative copy
4. Set up a payment method or confirm credits are available
5. List available slots and pick one
6. Book the slot (one-time or recurring)

## Step 1 — Confirm the account

```bash
guild-ads auth whoami --json
```

If exit code 2, the user is not signed in — defer to `guild-ads-cli`.

## Step 2 — Register (or find) the app being advertised

The advertiser side and publisher side share the same `apps` table — registering is identical to the publisher flow. If the user is already a publisher of the same app, the same `appId` works for advertising.

To find an existing app by bundle ID:

```bash
guild-ads apps lookup --bundle-id "$BUNDLE_ID" --json
```

Or list all apps the user owns:

```bash
guild-ads apps list --json
```

To register a new one:

```bash
APP_ID=$(guild-ads apps create \
  --name "$APP_NAME" \
  --bundle-id "$BUNDLE_ID" \
  --store-url "$STORE_URL" \
  --platform ios \
  --json | jq -r '.appId')
```

If the user is advertising someone else's app or a website (not an iOS/macOS app), the `--platform` should still match what they're promoting — the campaign URL is the actual destination.

## Step 3 — Create a campaign

A campaign is the creative — headline, optional description, destination URL — attached to an app. Multiple campaigns per app are fine; book each separately.

Ask the user for:
- **Campaign name** (internal label — "Spring Launch" or similar)
- **Headline** (the prominent text in the banner — short, action-oriented)
- **Destination URL** (where clicks go — typically the App Store or product page)

```bash
CAMPAIGN_ID=$(guild-ads campaigns create \
  --app "$APP_ID" \
  --name "$CAMPAIGN_NAME" \
  --headline "$HEADLINE" \
  --url "$DESTINATION_URL" \
  --json | jq -r '.campaignId')
```

To list existing campaigns:

```bash
guild-ads campaigns list --app "$APP_ID" --json
```

To edit a campaign's headline or URL after creation:

```bash
guild-ads campaigns edit "$CAMPAIGN_ID" --headline "New headline" --json
```

## Step 4 — Payment method or credits

Bookings settle in two ways: by **applying credits** (no card needed) or by **paying cash via Stripe Checkout** (saved card or fresh checkout each time).

Check the user's balance:

```bash
guild-ads balance show --json
```

The response includes `creditCents` (available credits) and `cashCents` (cash balance, e.g. from publisher earnings the user converted).

If the user wants to pay by card and doesn't already have a saved payment method, set one up:

```bash
guild-ads payment-methods setup --json
```

This opens Stripe Checkout to save a card, then polls until the card lands in the database. In a fully headless context, defer this step — the booking will need to use credits.

Promo codes can be redeemed for credits:

```bash
guild-ads promo redeem "$PROMO_CODE" --json
```

## Step 5 — Pick a slot

```bash
guild-ads slots list --json
```

Returns the next several bookable weeks with their current price (`basePriceCents`), already-sold percentage, and `weekStart` (Sunday in UTC). The user picks one. If they want the soonest available, pick the first un-fully-sold week.

## Step 6 — Book the slot

The minimum percentage is 1%. Common choices: 5%, 10%, 25%.

### Credits-only (no browser)

If the user has enough credits to cover the booking, the entire purchase settles synchronously:

```bash
guild-ads book create \
  --campaign "$CAMPAIGN_ID" \
  --slot "$SLOT_ID" \
  --percentage 10 \
  --apply-credits \
  --json
```

Exit 0 means the booking is confirmed. Skip to step 7.

### Cash or mixed (opens browser)

When cash is due (credits don't cover the full amount), the CLI opens Stripe Checkout in the browser and polls until the payment confirms:

```bash
guild-ads book create \
  --campaign "$CAMPAIGN_ID" \
  --slot "$SLOT_ID" \
  --percentage 10 \
  --json
```

Add `--apply-credits` to use any available credits first and only charge the remainder. Exit code 5 means the Stripe Checkout timed out — the user can retry; check `guild-ads book status --booking <id> --json` for the current state.

### Recurring weekly plan

For an always-on campaign that renews each week, use the `plans` subcommand instead of `book`:

```bash
guild-ads plans preview \
  --campaign "$CAMPAIGN_ID" \
  --percentage 10 \
  --json
```

`plans preview` returns the projected next-week price and any caps that would apply, without committing. To actually create a recurring plan:

```bash
guild-ads plans create \
  --campaign "$CAMPAIGN_ID" \
  --percentage 10 \
  --json
```

Plans renew automatically. Manage them with `plans list`, `plans pause <id>`, `plans resume <id>`, `plans edit <id> --percentage <n>`, and `plans cancel <id>`.

A recurring plan needs a saved payment method on file (step 4) — without one, the plan can't auto-renew when credits run out.

## Step 7 — Verify it landed

```bash
guild-ads activity --json
```

Shows a unified ledger of the user's recent activity, including the booking. Or:

```bash
guild-ads balance show --json
```

Shows the post-booking balance. Cash should have decreased (or credits, or both). The booking will appear on the user's dashboard at https://guildads.com.

## When the user is "done"

Confirm before declaring success:

- [ ] `guild-ads auth whoami --json` returns the user's email
- [ ] `guild-ads campaigns list --app "$APP_ID" --json` shows the campaign with the right headline and URL
- [ ] The booking shows up in `guild-ads activity --json` with the expected percentage
- [ ] `guild-ads balance show --json` reflects the charge

If the user **also** wants to monetize (run ads in their own app, get paid), switch to `guild-ads-publisher-setup` next.

## Common gotchas

- **"No campaigns" error when booking** — the campaign and slot must belong to the same `appId`. Re-check `--campaign` and that the slot is for a week the campaign's app is eligible for.
- **Booking exit 5 after Stripe Checkout** — payment might still complete after the timeout. Check `guild-ads book status` before re-booking; otherwise you might double-book.
- **Weekly price feels high** — the network's price is set by the prior week's demand. If the user wants a lower price, suggest waiting for a less-saturated week (visible in `guild-ads slots list --json`).
- **Recurring plan paused unexpectedly** — usually means the auto-renew payment failed (expired card, insufficient credits with no card on file). Run `guild-ads payment-methods setup` and `guild-ads plans resume <id>`.
