# Mahjong Leaderboard

A live leaderboard for our mahjong group. It shows who's winning, how often, and each
player's rating, and it updates on its own whenever a game is logged.

**Live site:** https://ynitsed3000.github.io/MahjongTracking/

## What it shows

- **Standings:** every player ranked by rating, with games, wins, win %, expected win % and luck.
  Click a column to re-sort. Players with fewer than 10 games are tagged "Provisional".
- **Player details:** click a player for their last 10 results, win streaks and head-to-head
  against everyone they've played with.
- **Rating history:** a chart of each player's rating after every game (also available as a table).
- **Recent games:** who won each game and who was at the table.
- **How ratings work:** the full formula, worked examples with the current numbers, and why the
  rating is built this way.

## How it works

```
Google Form  ->  Google Sheet (Log tab)  ->  published CSV  ->  this site
  (log a game)      (one row per player        (Google keeps       (reads the CSV on every
                     per game)                  it up to date)      page load, does the math)
```

There is no server and no database. The site is three static files on GitHub Pages. Each time
someone opens the page it downloads the Log tab, rebuilds every game, and recomputes all stats
and ratings in the browser using the same formulas as the sheet's Stats tab. If the sheet can't
be reached it shows a saved copy of the data and says so in the header.

| File | What it is |
| --- | --- |
| `index.html` | The whole site: layout, styles, and the code that reads the sheet and does the math. |
| `config.js` | One setting: the link to the published Log tab. |
| `snapshot.js` | A saved copy of the game log, shown only if the live sheet can't be reached. |
| `make_snapshot.py` | Optional helper that regenerates `snapshot.js` from an xlsx download. |

## Day to day

**Log a game:** use the Google Form as usual. The game shows up on the site after Google refreshes
the published CSV, usually within a few minutes.

**Add a player:** nothing to do. A new name appears on the site the first time it's in a game.

**Change the sheet link:** edit `config.js` on GitHub (pencil icon > paste the new link > Commit).

**Stop sharing the data:** in the sheet, File > Share > Publish to web > Stop publishing. The site
will then fall back to the saved copy.

## Set up from scratch

### 1. Publish the Log tab as CSV

1. Open the Google Sheet and click the **Log** tab.
2. File > Share > Publish to web.
3. Choose **Log** (not "Entire document") and **Comma-separated values (.csv)**, then Publish.
4. Copy the link and paste it into `config.js`:

   ```js
   window.MAHJONG_CONFIG = {
     sheetCsvUrl: "https://docs.google.com/spreadsheets/d/e/.../pub?gid=...&single=true&output=csv",
   };
   ```

Only the Log tab is published (names, dates and win/loss, which is what the site shows). Form
Responses and the other tabs stay private.

### 2. Host it on GitHub Pages

1. Create a public GitHub repository.
2. Upload `index.html`, `config.js` and `snapshot.js` to the repo root.
3. Settings > Pages > Build and deployment: Source "Deploy from a branch", branch `main`, folder `/ (root)`.
4. After about a minute the site is live at `https://<username>.github.io/<repo-name>/`.

Any change you commit redeploys automatically. Hard-refresh (Cmd+Shift+R) to skip the browser's cached copy.

## The math

- **Rating** = 1000 + ( (W + 10 x p) / (N + 10) - p ) x 2000
  - W = the player's wins, N = their games played
  - p = the group's win rate = everyone's wins / everyone's games
  - The 10 "imaginary games" at the group average keep small samples from swinging the
    ranking. As N grows the adjustment fades (50% weight on your real record at 10 games, 86% at 60).
  - 1,000 means an average win rate, and one percentage point above or below it is worth 20 points.
- **Expected win %** = the average of 1 / (players at the table) over the games a player took part in.
  It is 25% at a 4-player table and 33% at a 3-player table.
- **Luck** = win % - expected win %, in percentage points.

The site's "How ratings work" section explains all of this in more detail, with worked examples.

## Troubleshooting

| What you see | What it means |
| --- | --- |
| Header says "Live from the sheet" | Everything is working. |
| "Couldn't reach the sheet, showing a saved copy" | The link in `config.js` is wrong, or publishing was stopped. Re-publish the Log tab (step 1) and check the link. The saved copy is only as new as the last `snapshot.js`. |
| "Saved copy of the game log" | `config.js` has no link, so the site never tries the sheet. |
| A new game isn't showing | Google can take a few minutes to refresh the published CSV. Wait, then hard-refresh. |
| Red note: "G12 has 2 winners" | That game in the Log tab doesn't have exactly one Win row. Fix it in the sheet. |
| Old version of the page | Hard-refresh (Cmd+Shift+R). Browsers cache the page for a few minutes after a deploy. |

## Known limitations

- **Edited form responses don't update the Log tab.** The site shows whatever the Log says, so an
  edited response can leave the leaderboard out of sync with the form. Fix the row in the Log tab directly.
- **The rating doesn't know who you played** (a win at a strong table counts the same as a win at a
  weak one) and **doesn't adjust for table size**. The Luck column shows the table-size effect.
- Updates are not instant: they follow Google's refresh of the published CSV.

## Refresh the saved copy (optional)

`snapshot.js` only matters when the sheet can't be reached. To update it from an xlsx download of the
sheet:

```
pip3 install openpyxl
python3 make_snapshot.py "path/to/Mahjong Tracker with Elo.xlsx"
```

Then upload the new `snapshot.js` to the repo.

## Preview locally

From this folder:

```
python3 -m http.server 8000
```

Then open http://localhost:8000. Opening `index.html` directly from disk also works, but a local
server behaves more like the real site.
