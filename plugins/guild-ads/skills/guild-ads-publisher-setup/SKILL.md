---
name: guild-ads-publisher-setup
description: Use when the user wants to monetize an iOS, macOS, watchOS, tvOS, or visionOS app by joining Guild Ads as a publisher. Triggers include "add Guild Ads to my app," "monetize my app," "show ads in my app," "integrate the Guild Ads SDK," "I want to publish on Guild Ads," "add an ad to my app for revenue," or any flow that ends with the Guild Ads iOS SDK shipped in the user's app. Covers signing up, registering the app on the network, generating an SDK token, integrating GuildAds via SwiftPM, configuring at launch, placing the banner, and setting up Stripe Connect for weekly payouts. Assumes guild-ads-cli has already been used to install and authenticate the CLI.
---

# Guild Ads Publisher Setup

Guides an agent through everything an iOS/macOS/watchOS/tvOS/visionOS developer needs to start earning from Guild Ads. The end state: the user's app ships with a Guild Ads banner, an SDK token in the app's launch code, and a Stripe Connect account ready to receive weekly payouts.

**Prerequisite:** the `guild-ads` CLI is installed and the user is signed in (see the `guild-ads-cli` skill). If not, switch to that skill first.

## The shape of the flow

1. Confirm or create the user's account
2. Register the app on the network (gets an `appId`)
3. Generate an SDK token for the app (gets a `token`)
4. Integrate the [Guild Ads iOS SDK](https://github.com/Guild-Ads/guild-ads-ios) into the user's Xcode project
5. Set up Stripe Connect to receive payouts

Steps 1–3 are CLI calls. Step 4 is editing the user's Swift code. Step 5 opens a browser.

## Step 1 — Confirm the account

```bash
guild-ads auth whoami --json
```

If exit code 2, the user is not signed in — defer to `guild-ads-cli` to log in or sign up. If they need to sign up, recommend they confirm the email link before continuing; otherwise step 3 will fail.

## Step 2 — Register the app

Ask the user for these values if you don't already know them:
- App name (display name shown to advertisers — usually the same as the App Store title)
- Bundle identifier (e.g. `com.example.myapp`)
- App Store URL (e.g. `https://apps.apple.com/app/id123456789`)
- Platform (`ios` is most common; also accepts `macos`, `watchos`, `tvos`, `visionos`)

If the user has an Xcode project open, you can usually find the bundle ID in `*.xcodeproj/project.pbxproj` under `PRODUCT_BUNDLE_IDENTIFIER` or the Info.plist. Confirm with the user before sending it.

```bash
APP_ID=$(guild-ads apps create \
  --name "$APP_NAME" \
  --bundle-id "$BUNDLE_ID" \
  --store-url "$STORE_URL" \
  --platform ios \
  --json | jq -r '.appId')

echo "App registered: $APP_ID"
```

If the bundle ID has been registered before by the same user, `apps create` returns the existing app rather than erroring. Use `guild-ads apps lookup --bundle-id <id> --json` to find an existing app without creating one.

## Step 3 — Generate an SDK token

```bash
TOKEN_RESPONSE=$(guild-ads tokens create --app "$APP_ID" --name "Production" --json)
TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r '.token')
TOKEN_ID=$(echo "$TOKEN_RESPONSE" | jq -r '.tokenId')
```

**The token value is shown once and only once.** Save `$TOKEN` immediately. Listing tokens later (`guild-ads tokens list`) returns only the ID and name, never the value. If the token is lost, the user must revoke and recreate it.

For development vs. production, create separate tokens with different `--name` labels so they can be revoked independently.

## Step 4 — Integrate the iOS SDK

The SDK lives at [Guild-Ads/guild-ads-ios](https://github.com/Guild-Ads/guild-ads-ios). Three substeps: add the package, configure at launch, place the banner.

### 4a. Add the package via Swift Package Manager

If the user has a `Package.swift`, add the dependency:

```swift
.package(url: "https://github.com/Guild-Ads/guild-ads-ios.git", from: "1.0.3")
```

…and add `"GuildAds"` to the appropriate target's dependency list.

If the user is using Xcode without a `Package.swift`, walk them through **File > Add Package Dependencies** and pasting `https://github.com/Guild-Ads/guild-ads-ios`. Do not attempt to edit `*.xcodeproj/project.pbxproj` directly — let Xcode handle it.

### 4b. Configure at app launch

In the SwiftUI `App` initializer (or the UIKit `application(_:didFinishLaunchingWithOptions:)`), call `GuildAds.configure(token:)` exactly once before any view appears.

**Do not paste the token literal into source.** Read it from a build setting, environment variable, `.xcconfig`, or `Info.plist` so it doesn't leak into git history. If the user uses xcconfigs, the simplest path is:

1. Add `GUILD_ADS_TOKEN = <token>` to a `Secrets.xcconfig` listed in `.gitignore`
2. Reference it from `Info.plist` as `GuildAdsToken = $(GUILD_ADS_TOKEN)`
3. Read it at launch:

```swift
import SwiftUI
import GuildAds

@main
struct MyApp: App {
    init() {
        guard let token = Bundle.main.object(forInfoDictionaryKey: "GuildAdsToken") as? String,
              !token.isEmpty else {
            assertionFailure("GuildAdsToken missing from Info.plist")
            return
        }
        GuildAds.configure(token: token)
    }

    var body: some Scene {
        WindowGroup { ContentView() }
    }
}
```

If the user is fine with embedding a public-ish token directly in source for now (it can be revoked from the dashboard or CLI any time), a hard-coded literal is acceptable for prototyping. Recommend the xcconfig path for anything they intend to ship.

### 4c. Place the banner

Drop `GuildAdsBanner` somewhere in the SwiftUI hierarchy. The `placementID` is a string the user picks — `snake_case`, descriptive, stable. It identifies where in the app the ad lives; the dashboard groups stats by placement.

```swift
import SwiftUI
import GuildAds

struct ContentView: View {
    var body: some View {
        VStack {
            // … the user's content …
            Spacer()
            GuildAdsBanner(placementID: "settings_footer")
        }
    }
}
```

Choose a placement ID that will still make sense six months later (`home_feed_footer`, `settings_bottom`, `library_inline`). Avoid `ad`, `banner1`, etc.

The banner targets 360pt × 50pt. It collapses to zero height if no ad is available, so layouts don't need to reserve space. Don't wrap it in `scaleEffect`, `rotationEffect`, or parent `opacity` modifiers — see the iOS SDK README for embedding tips.

For UIKit, host with `UIHostingController` (the SDK README has the exact pattern).

### 4d. Verify it loads

Run the app in a `DEBUG` scheme and filter the console for `[GuildAds]`. The SDK logs:

```
[GuildAds] Banner load for placement 'settings_footer'
[GuildAds] Refreshed ad for 'settings_footer': Acme App
[GuildAds] Reporting impression for 'settings_footer'
```

If you see "No cached ad" followed by nothing else, the network may be unreachable, the token may be wrong, or the app's `appId` may not match the token's app. If you see no logs at all, `GuildAds.configure(token:)` did not run.

## Step 5 — Set up Stripe Connect for payouts

Earnings accumulate as you serve ads. Before they can be paid out, the user needs to complete Stripe Connect onboarding (legal name, address, bank account, tax info — Stripe handles all of it).

```bash
guild-ads payouts setup --json
```

This opens a browser to Stripe Connect's hosted onboarding form. The CLI then polls until `detailsSubmitted` is `true` or until it times out (exit code 5). If it times out, the user can finish later — re-run `payouts setup` or check status:

```bash
guild-ads payouts status --json | jq '.detailsSubmitted'
```

In a fully headless context where no browser is available, defer this step and tell the user they'll need to complete it on a machine where they can interact with a browser.

There's a 7-day hold on each new payment before it becomes payable. After that, payouts run automatically on a weekly schedule. The user can request an immediate payout with `guild-ads payouts dashboard` (which opens the Stripe Express dashboard) or wait for the weekly run.

## When the user is "done"

Confirm all of these before declaring the setup complete:

- [ ] `guild-ads auth whoami --json` returns an email
- [ ] `guild-ads apps list --json` shows the registered app with the right bundle ID
- [ ] An SDK token exists for the app (`guild-ads tokens list --app "$APP_ID" --json`)
- [ ] The token is wired into the running app (not just printed in the terminal — actually inserted into the project's launch code or xcconfig)
- [ ] `GuildAdsBanner(placementID:)` appears in at least one view
- [ ] The app builds and runs without errors after the integration
- [ ] (Optional) `guild-ads payouts status --json` shows `detailsSubmitted: true` — only required to actually receive money

If the user wants to **also** advertise, switch to `guild-ads-advertiser-setup` next. The publisher account works the same with or without the advertiser side.
