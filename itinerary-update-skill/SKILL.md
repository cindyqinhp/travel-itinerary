---
name: itinerary-update
description: Updates Cindy's travel itinerary by merging new information into the GitHub Gist that powers https://cindyqinhp.github.io/travel-itinerary/. Use this skill whenever the user types /itinerary-update, wants to add hotels, flights, restaurants, activities, or any trip details to the itinerary, or pastes in booking confirmations or travel info they want saved. Always use this skill — don't try to update the Gist manually without it.
---

# Itinerary Update

You're helping Cindy update her live travel itinerary. The data lives in a private GitHub Gist and powers a published web app. Your job is to fetch the current data, understand what she wants to add or change, propose the exact changes for her to review, and only push after she confirms.

## Credentials

- **Gist ID**: `0179a64f509b1cc8530fc7e3057e18eb`
- **Gist token**: `<GIST_TOKEN>`
- **Gist file**: `itinerary.json`

## Step 1 — Fetch current data

Fetch the Gist via the GitHub API:

```bash
curl -s \
  -H "Authorization: token <GIST_TOKEN>" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/gists/0179a64f509b1cc8530fc7e3057e18eb \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['files']['itinerary.json']['content'])"
```

Read the current state so you know what's already there before proposing any changes.

## Step 2 — Get new information from the user

If the user hasn't already pasted information, ask them now:

> "Paste anything you want to add — booking confirmations, hotel names, restaurant ideas, flight details, bullet points, whatever. I'll sort it out."

Accept messy input: copied emails, raw text, unformatted notes. Your job is to parse it.

## Step 3 — Plan the changes

Figure out what needs to change by comparing the user's input against the current Gist. Follow these rules:

- **Add** new activities or locations that don't exist yet
- **Keep** existing entries unless the user explicitly says to replace or update something
- **Merge** notes or details into existing entries only if the user says to
- **Infer** dates, times, and categories as best you can from context
- When a date is ambiguous, ask rather than guess

## Data structure

```json
{
  "trip": { "name": "Our Trip", "start": "2026-07-07", "end": "2026-08-04" },
  "days": {
    "YYYY-MM-DD": [
      {
        "name": "Activity name",
        "time": "HH:MM",
        "category": "activity",
        "location": "Venue name, City",
        "mapsUrl": "https://maps.google.com/...",
        "notes": "Confirmation #, tips, what to bring…",
        "photo": ""
      }
    ]
  },
  "locations": {
    "YYYY-MM-DD": { "city": "Paris", "country": "France" }
  },
  "_updatedAt": 1751299200000
}
```

`_updatedAt` is an internal sync field (epoch ms) used by the web app to resolve which copy is newest — always refresh it on push (see Step 5). Leave the rest of the structure untouched.

**Categories** (must be exactly one of): `activity`, `transport`, `food`, `accommodation`, `other`

**Category guidance:**
- `transport` — flights, trains, taxis, shuttles
- `accommodation` — hotel check-in/check-out
- `food` — restaurants, cafés, bars
- `activity` — sightseeing, tours, museums, experiences
- `other` — anything else

## Step 4 — Show proposed changes for review

Before touching the Gist, present a clear summary of exactly what you're about to do. Group by date and make it easy to scan:

```
Here's what I'm planning to add/change:

📅 July 10
  + ADD: 🍽️ Lunch at Le Grand Véfour (12:30, food)
      📍 17 Rue de Beaujolais, Paris
      Notes: Reservation under Smith, confirmation #ABC123

📅 July 15
  ~ UPDATE: Hotel check-in notes → adding "early check-in requested"

Nothing else will be changed.

Looks good? (yes / no, or tell me what to adjust)
```

Wait for explicit confirmation. If they say "yes" or "looks good", proceed. If they want changes, adjust and show the summary again.

## Step 5 — Push to Gist

Once confirmed, build the updated JSON and push.

**IMPORTANT — stamp `_updatedAt`:** the live web app uses a top-level `_updatedAt` field (epoch milliseconds) to decide whose copy is newest. Always set it to the current time on every push, or the site may treat your push as stale and ignore it.

```bash
python3 - <<'EOF'
import json, time, urllib.request, urllib.error

TOKEN = "<GIST_TOKEN>"
GIST_ID = "0179a64f509b1cc8530fc7e3057e18eb"

updated = <your updated dict here>
updated["_updatedAt"] = int(time.time() * 1000)  # mark this push as the newest version

payload = json.dumps({
    "files": {
        "itinerary.json": {
            "content": json.dumps(updated, ensure_ascii=False, indent=2)
        }
    }
}).encode()

req = urllib.request.Request(
    f"https://api.github.com/gists/{GIST_ID}",
    data=payload, method="PATCH",
    headers={
        "Authorization": f"token {TOKEN}",
        "Content-Type": "application/json",
        "Accept": "application/vnd.github.v3+json"
    }
)
try:
    with urllib.request.urlopen(req) as res:
        print("✓ Gist updated successfully")
except urllib.error.HTTPError as e:
    print("Error:", e.code, e.read().decode())
EOF
```

After pushing, confirm success to the user and remind them the live site at https://cindyqinhp.github.io/travel-itinerary/ will reflect the changes within a few seconds.
