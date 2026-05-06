# Guild Ads — Coding Agent Plugin

Skills that teach Claude Code and Codex how to set up a [Guild Ads](https://guildads.com) account and use the [`guild-ads`](https://www.npmjs.com/package/guild-ads) CLI. Once installed, you can ask your agent things like:

> Set me up on Guild Ads as a publisher — I want to monetize my iOS app.

> I want to advertise my new app on Guild Ads. Walk me through it.

> Show ads in my app and pay me weekly.

…and the agent will install the CLI, sign you up if you don't have an account, register your app, generate an SDK token, integrate the iOS SDK, set up Stripe payouts — whatever the role calls for.

## What's inside

Three skills, designed to be invoked by description match:

| Skill | When the agent uses it |
|---|---|
| `guild-ads-cli` | Foundational. Installing the CLI, signing in, running any `guild-ads` command. |
| `guild-ads-publisher-setup` | The user wants to monetize their app — register, get an SDK token, integrate the iOS SDK, set up payouts. |
| `guild-ads-advertiser-setup` | The user wants to advertise their app — create a campaign, book ad slots, manage recurring plans. |

Skills compose. The publisher and advertiser skills both lean on `guild-ads-cli` for CLI install + auth.

## Install — Claude Code

Run these inside Claude Code:

```
/plugin marketplace add Guild-Ads/guild-ads-plugin
/plugin install guild-ads@guild-ads
/reload-plugins
```

After `/reload-plugins`, the skills are loaded and the agent will pull them in when relevant.

## Install — Codex

Run this inside Codex:

```
codex plugin marketplace add Guild-Ads/guild-ads-plugin
```

Once added, `/plugins` shows Guild Ads in the list. If you cloned the repo locally, Codex auto-detects the marketplace from the local checkout.

## What the agent will do

The agent does not silently sign you up for things or charge your card. The flows include explicit confirmations for:

- Account creation (you have to confirm an emailed link)
- Stripe Checkout (opens a browser; you complete the form yourself)
- Stripe Connect onboarding for payouts (browser; you submit your own bank/tax info)
- Booking with cash payment (Stripe Checkout, browser)

Booking with credits is fully synchronous and does not require a browser. The agent will tell you what it's about to do before it does it.

## What's underneath

The skills are thin teaching wrappers around the [`guild-ads`](https://www.npmjs.com/package/guild-ads) CLI. The CLI does the actual work — talking to the Guild Ads API, opening browsers when needed, managing your local credentials at `~/.config/guild-ads/auth.json`.

If you want to use the CLI directly without an agent, the [README on npm](https://www.npmjs.com/package/guild-ads) has the same commands the skills walk through.

## Source

- CLI: [npm: guild-ads](https://www.npmjs.com/package/guild-ads)
- iOS SDK: [Guild-Ads/guild-ads-ios](https://github.com/Guild-Ads/guild-ads-ios)
- Dashboard: [guildads.com](https://guildads.com)

## License

MIT
