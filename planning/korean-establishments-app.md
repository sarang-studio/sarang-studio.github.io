# Plan: Korean establishments app for iPhone

Working title: **Maeul** (마을, "village"). Placeholder; rename freely.

A directory of Korean businesses near you, starting with restaurants and groceries, with community voting that produces ranked lists such as "Top 5 Korean restaurants".

This document is the plan. The app code will live in its own repository (suggested: `sarang-studio/maeul`); this site keeps only the privacy policy page and the landing-page entry, following the Daylog pattern in the README.

---

## 1. What we are building

### v1 scope (ship this)

1. **Browse** Korean restaurants and Korean groceries in one region, as a list and on a map, sorted by distance or by votes, with search.
2. **Detail page** per place: map, address with one-tap Apple Maps directions, phone, website, hours, subcategory tags, vote button, report button.
3. **Vote**: each person can upvote places they like. One vote per person per place, toggle on and off. No account sign-up; identity comes from the phone's iCloud account (opaque ID, no name or email).
4. **Top 5 lists**, computed from votes: "Top 5 Korean restaurants" and "Top 5 Korean groceries" for the region, refreshed live.
5. **Submit a place**: anyone can suggest a missing business. Submissions go into a moderation queue and appear only after approval.

### Explicitly out of v1

- Reviews, comments, photos from users. These are the expensive part of user-generated content (moderation, App Store rules, abuse). Votes plus submissions give the community feature without free-text.
- More than one region. The data model supports many regions from day one, but launch with one.
- Other categories (cafes, bakeries, beauty, hagwon, churches, clinics). The model is category-agnostic so these are a data change later, not a code change.
- Android, web.

### Decisions needed before coding (see section 9)

- Launch region.
- App name.
- Subcategory taxonomy for restaurants (BBQ, soup/tofu house, chicken, noodles, bakery/café, bunsik, etc.).

---

## 2. Architecture

### Why this needs a server, and why CloudKit

Daylog's promise is "everything stays on your device". A vote is meaningless unless it is counted with everyone else's, so this app must share data between users. The choice is what kind of server.

**Recommendation: CloudKit public database.**

- Apple-hosted, free at this scale, nothing for us to run or patch.
- No sign-up. CloudKit gives the app a stable, opaque user record ID from the phone's iCloud account. We never see a name, email or Apple ID. This keeps the "private" studio ethos intact: the only identifier that leaves the phone is one Apple already holds.
- Moderation can be done in the CloudKit Dashboard in a browser. No admin app needed for v1.
- SwiftData or plain Codable caching on-device gives offline browsing.

Trade-offs to accept:

- No server-side code. Vote totals are kept on the place record and updated with optimistic concurrency (see 4.3). Fine at community scale.
- People without an iCloud account signed in can browse but not vote or submit. The UI must say so plainly.
- Data is US-hosted by Apple. Note it in the privacy policy.

Alternative considered: Supabase (Postgres, anonymous auth, SQL for rankings). Better if we ever want reviews, search across regions, or a web version. More moving parts and a second privacy relationship. Revisit if v2 needs server logic.

### Stack

| Layer | Choice |
|---|---|
| Language / UI | Swift 6, SwiftUI, iOS 17 minimum |
| Shared data | CloudKit public database |
| Local cache | SwiftData (places for the active region, my votes) |
| Maps | MapKit (`Map`, `MKLocalSearch` for submission autofill, `MKMapItem.openInMaps` for directions) |
| Location | CoreLocation, "While Using" only, reduced accuracy is enough |
| Tests | XCTest with an in-memory fake of the CloudKit repository protocol |
| CI | Xcode Cloud or GitHub Actions on a macOS runner: build + unit tests on every push |

Note: this cloud session runs on Linux and cannot build or run an iOS project. Scaffolding, models, tests and docs can be written here; builds and simulator runs happen on a Mac with Xcode.

---

## 3. Screens

Four tabs.

**Explore** (default)
- Segmented control: Restaurants | Groceries.
- Sort: Nearest | Most votes. Search field filters by name (English or Korean) and subcategory.
- Map toggle in the toolbar shows the same filtered set as pins.
- Row: name, Korean name, subcategory, distance, vote count, vote button.
- Empty state when location is denied: sorted by votes, with a one-line explanation.

**Top 5**
- One card per category: ranked 1 to 5 with name, subcategory, vote count.
- Tapping a row opens the detail page.
- Small "how this works" line: "Ranked by votes from people using this app in <region>."

**Add**
- Search field backed by `MKLocalSearch` so the user picks the real business and we get name, address and coordinates without typing.
- Category and subcategory pickers, optional Korean name, optional note to the moderator.
- Submit shows "Thanks, we will review it." Nothing appears publicly until approved.

**Settings**
- Region picker (v1 has one, but the control exists so adding a second is data-only).
- My votes list.
- Links: privacy policy on this site, support (this repo's Issues page).
- iCloud status explanation when signed out.

**Detail** (pushed from Explore or Top 5)
- Map snippet, address (tap: directions in Apple Maps), phone (tap: call), website, hours text, tags.
- Vote button with count.
- Report: sheet with reasons (closed, wrong details, not Korean-owned or Korean-focused, duplicate, other). Creates a Report record; nothing else changes.

---

## 4. Data model (CloudKit record types)

### 4.1 Records

**Region**
- `name` (String), `center` (Location), `radiusKm` (Double), `isActive` (Int)

**Place**
- `name`, `nameKorean` (String, optional)
- `category` (String: `restaurant` | `grocery`) and `subcategory` (String, from the taxonomy)
- `address` (String), `location` (Location), `phone`, `website`, `hours` (String, optional)
- `region` (Reference → Region)
- `status` (String: `pending` | `approved` | `hidden`)
- `voteCount` (Int64, queryable and sortable)
- `submittedBy` (String, the submitter's user record name; never shown)
- `createdAt`, `approvedAt` (Date)

**Vote**
- Record name is `"\(userRecordName)|\(placeRecordName)"`. That makes one-vote-per-person-per-place a property of the key, not a check we might forget.
- `place` (Reference → Place), `region` (Reference), `createdAt`

**Report**
- `place` (Reference), `reason` (String), `note` (String, optional), `reportedBy` (String), `createdAt`

### 4.2 Permissions (CloudKit security roles)

- World: read `Region`, read `Place` where the client filters `status == approved` (CloudKit cannot hide fields per row, so pending places are simply excluded by every query the app runs; nothing sensitive is in a pending record beyond the business's public details).
- Authenticated iCloud users: create `Vote`, `Report`, `Place` (with `status` forced to `pending` by the app). Delete own `Vote`.
- Only the creator can modify a `Vote`. Nobody but us (Dashboard) can modify a `Place` after creation.
- `submittedBy` and `reportedBy` are opaque record names, not personal data, but are still never displayed.

### 4.3 Voting and ranking without server code

Casting a vote:

1. Save the `Vote` record. If it already exists, the user is un-voting: delete it instead.
2. Fetch the `Place`, increment or decrement `voteCount`, save with `savePolicy = .ifServerRecordUnchanged`.
3. On `serverRecordChanged`, re-fetch and retry (up to 3 times). Conflicts only happen when two people vote for the same place in the same second, so retries are rare.
4. Update the local cache optimistically so the UI feels instant; roll back if step 1 fails.

Top 5 is a query: `Place` where `region == X`, `category == Y`, `status == approved`, sorted by `voteCount` descending, limit 5. Ties break by `approvedAt` ascending (older listing wins), which is stable and hard to game.

Integrity: `Vote` records are the source of truth. A small maintenance script (Swift command-line tool using the CloudKit Web Services API with a server-to-server key, run from a Mac or a GitHub Action) can recount `voteCount` from `Vote` records on demand. Not needed for launch, but write it during phase 2 so drift can always be repaired.

Anti-abuse for v1: one iCloud account equals one vote per place, which is already a high bar. Optionally cap votes at 10 per person per category to keep the leaderboard meaningful. No captcha, no rate limiter beyond CloudKit's own.

---

## 5. Seed data

The app launches with a curated list, not an empty screen. Target 30 to 60 places for the launch region.

- Keep the seed in the app repo as `Tools/seed/places.csv` with the columns of the `Place` record.
- Import with a small Swift command-line tool (`Tools/seed/import`) that geocodes addresses through `CLGeocoder` and creates the records via CloudKit. Idempotent: it keys on `name + address`.
- Sources: own local knowledge, Korean community listings, the businesses' own sites. Do not scrape Yelp, Google Maps or Apple Maps into the database; their terms prohibit storing their results, and MapKit search results are for display, not for building a directory.
- Apple's `MKLocalSearch` is used only at submission time to autofill a place the user already chose.

---

## 6. Privacy and App Store notes

- New privacy policy page at `/maeul/privacy/index.md` on this site, same front matter as Daylog. It must state what is different from Daylog: votes, submissions and reports are stored in Apple's CloudKit public database and are visible to other users in aggregate; the only identifier stored is Apple's opaque CloudKit user record ID; location is used while the app is open to sort by distance and is never stored or sent.
- The landing page lede currently says "Everything stays on your device." Change it to a wording that is true for both apps, or add a one-line qualifier in the app's entry.
- App Store review guideline 1.2 (user-generated content) applies because of submissions and votes. Our answer: submissions are held for moderation before publication, there is a report button on every place, and there is no free-text visible to other users. Document this in the review notes.
- Info.plist strings: location "While Using" reason ("To sort places by distance. Your location is never stored."), and no other permission requests.
- Entitlements: iCloud with CloudKit, one container.
- Privacy nutrition label: Location (not linked to identity, app functionality), User Content (submissions, not linked), Identifiers (CloudKit user ID, not linked). No tracking.

---

## 7. Repository layout (new repo `sarang-studio/maeul`)

```
Maeul.xcodeproj
Maeul/
  App/            MaeulApp.swift, RootTabView.swift
  Models/         Place.swift, Region.swift, Vote.swift, Report.swift, Category.swift
  Data/           CloudKitRepository.swift (protocol + live), LocalCache.swift (SwiftData), VoteService.swift
  Features/
    Explore/      ExploreView, PlaceRow, PlaceMapView, ExploreViewModel
    Detail/       PlaceDetailView, ReportSheet
    TopFive/      TopFiveView, LeaderboardCard
    Submit/       SubmitPlaceView, PlaceSearchField (MKLocalSearch)
    Settings/     SettingsView, MyVotesView
  Location/       LocationProvider.swift
  Resources/      Assets, Localizable.xcstrings (English first, Korean second)
MaeulTests/       Ranking, vote toggle, conflict retry, CSV parsing, with FakeRepository
Tools/seed/       places.csv, import tool
README.md, PRIVACY.md (mirror of the site page), CHANGELOG.md
```

`CloudKitRepository` is a protocol so every view model is testable without iCloud. The live implementation is the only file that imports CloudKit.

---

## 8. Phases

Estimates are for one developer working part-time. Each phase ends in a TestFlight build.

**Phase 0: decide and seed** (about a week)
- Pick region, name, subcategory taxonomy.
- Create the app repo, Xcode project, CloudKit container and schema.
- Write `places.csv` for 30 to 60 places and the import tool. Import to the development environment.

**Phase 1: browse** (2 weeks)
- Explore tab with list, map, search, distance sort. Detail page with directions and call. Local cache and offline display. Location permission flow.
- Done when: a fresh install shows the seeded places sorted by distance, and works with location denied.

**Phase 2: vote and Top 5** (2 weeks)
- iCloud status handling, Vote records, optimistic-concurrency counter, Top 5 tab, My votes. Recount tool.
- Done when: two devices voting on the same place agree on the count within a refresh, and un-voting works.

**Phase 3: submit, report, moderate** (1 to 2 weeks)
- Add tab with `MKLocalSearch` autofill, pending status, Report sheet. Moderation checklist for the CloudKit Dashboard (approve, edit, hide).
- Done when: a submission from a test device appears in the Dashboard as pending and shows in the app after approval.

**Phase 4: release** (1 week)
- Privacy policy page and landing page entry on this site. App Store listing, screenshots, review notes about UGC. Deploy CloudKit schema to production. Submit.

**Later (v2 candidates, in rough priority)**
- Themed lists by subcategory ("Top 5 Korean BBQ") since subcategory filtering already exists.
- Second region. Region picker already in Settings.
- More categories (cafés, bakeries, beauty, services).
- Time-boxed polls ("Best kimchi in town, votes close Sunday").
- Korean localization of the UI, a home-screen widget for the Top 5.
- If reviews or photos are ever wanted, reassess the backend (see section 2) and the moderation load before committing.

---

## 9. Open questions for the studio

1. **Region.** Which metro area launches first? This shapes the seed list, and the subcategory taxonomy tends to follow local scene (BBQ-heavy vs. bakery-heavy).
2. **Name.** "Maeul" is a placeholder. Check App Store name availability before Phase 0 ends.
3. **Vote cap.** Unlimited upvotes per person, or a cap per category (suggested: 10)? A cap makes Top 5 more meaningful; unlimited is simpler to explain.
4. **Korean-owned vs. Korean-cuisine.** What qualifies a place? Suggested: serves Korean food or stocks Korean groceries as its main business. Write the rule into the Add screen and the moderation checklist so it is applied consistently.
5. **Hours.** Free text from the seed list, or skip in v1 and link to the business's site? Free text is quick to enter and goes stale; recommendation is to include it but show "hours may vary" and rely on reports to fix.
