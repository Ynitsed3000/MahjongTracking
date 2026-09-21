# Mahjong Leaderboard

A live leaderboard for our mahjong group. It shows who's winning, how often, each player's
rating, and how every game night went, and it updates on its own whenever a game is logged.

**Live site:** https://ynitsed3000.github.io/MahjongTracking/

## What it shows

- **Header:** a **Log a game** button that opens the Google Form, plus a status line showing whether
  the data is live from the sheet.
- **Filters:** a **Period** menu (All time, Last 30 days, or any month) and a **Rating** toggle
  (Table-adjusted, the default, or Group average). Everything below recalculates for the choice.
- **Awards:** longest win streak, longest cold streak, best night, most games, beating the odds,
  and biggest climb.
- **Standings:** every player ranked by rating, with games, wins, win %, expected win % and luck.
  Arrows next to each rating show the change since the previous game night. Click a column to
  re-sort. Players with fewer than 10 games are tagged "Provisional".
- **Player details:** click a player for their last 10 results, streaks, nemesis, lucky charm and
  head-to-head against everyone they've played with. **Copy link to this player** gives a link that
  opens straight to them.
- **Rating history:** a chart of each player's rating after every game (also available as a table).
- **Game log:** games grouped by night (with a night MVP) or listed game by game.
- **Head to head:** a grid of every pair of players and who wins more often when they share a table.
- **How ratings work:** the full formula, worked examples with the current numbers, and why the
  rating is built this way.

## How it works

```
Google Form  ->  Google Sheet (Log tab)  ->  published CSV  ->  this site
  (log a game)      (one row per player        (Google keeps       (reads the CSV on every
                     per game)                  it up to date)      page load, does the math)
```

There is no server and no database. The site is static files on GitHub Pages. Each time someone
opens the page it downloads the Log tab, rebuilds every game, and recomputes all stats and
ratings in the browser. The browser remembers the last data it loaded, so if Google is slow or down
the page shows that instead, and says when it was from. On a first visit with no sheet available it
falls back to `snapshot.js`.

The site only reads columns A to D of the Log tab (Date, Game ID, Player, Result). The Player Count,
Winners in Game and hidden banding columns are ignored, since it recalculates those itself. Games are
ordered by Game ID number (G1, G2, ...), not by row position, so a game that gets rewritten lower down
the Log after an edit keeps its place in the history.

| File | What it is |
| --- | --- |
| `index.html` | The whole site: layout, styles, and the code that reads the sheet and does the math. |
| `config.js` | Two settings: the link to the published Log tab, and the link to the Google Form. |
| `snapshot.js` | A saved copy of the game log, used only if the sheet has never been reachable. |
| `og.png` | The preview image shown when the site link is pasted into a chat. |
| `make_snapshot.py` | Optional helper that regenerates `snapshot.js` from an xlsx download. |

## How games get into the sheet

1. The Google Form asks two questions: **Who played?** (checkboxes, with "Other" for a new name)
   and **Who won?** (dropdown, kept in sync with the Players sheet).
2. Each submission lands in the **Form Responses 1** tab. Column D is used by the script for
   validation warnings and column E stores the Game ID that response produced.
3. A script in the spreadsheet's Apps Script project turns each response into Log rows, one per
   player. It adds any new player to the Players sheet, checks that the winner was actually at the
   table, and assigns the next Game ID.
4. **Editing a response** re-runs the same script. It reuses the stored Game ID, deletes that game's
   old Log rows and writes them again, so corrections flow through to the Log and the website.

Two installable triggers drive this: `onFormSubmit` for new submissions and `onFormEdit` for edits.
Both must be created on the **spreadsheet's** Apps Script project, not the form's. Binding them to the
form instead breaks the edit trigger.

## Day to day

**Log a game:** use the **Log a game** button on the site, or the Google Form directly. The game shows
up on the site after Google refreshes the published CSV, usually within a few minutes. Anyone with the
page already open can press **Refresh** in the header.

**Fix a mistake:** edit the response in the Form Responses 1 tab (or through the form's edit link). The
Log tab updates automatically and the site follows once Google refreshes the published CSV.

**Add a player:** nothing to do. A new name appears on the site the first time it's in a game.

**Change the sheet or form link:** edit `config.js` on GitHub (pencil icon > paste the new link > Commit).
Leave `formUrl` empty to hide the Log a game button.

**Stop sharing the data:** in the sheet, File > Share > Publish to web > Stop publishing. Browsers that
have visited before will keep showing the last data they saw.

## Set up from scratch

### 1. Publish the Log tab as CSV

1. Open the Google Sheet and click the **Log** tab.
2. File > Share > Publish to web.
3. Choose **Log** (not "Entire document") and **Comma-separated values (.csv)**, then Publish.
4. Copy the link and paste it into `config.js`:

   ```js
   window.MAHJONG_CONFIG = {
     sheetCsvUrl: "https://docs.google.com/spreadsheets/d/e/.../pub?gid=...&single=true&output=csv",
     formUrl: "https://forms.gle/...",
   };
   ```

Only the Log tab is published (names, dates and win/loss, which is what the site shows). Form
Responses and the other tabs stay private.

### 2. Host it on GitHub Pages

1. Create a public GitHub repository.
2. Upload `index.html`, `config.js`, `snapshot.js` and `og.png` to the repo root.
3. Settings > Pages > Build and deployment: Source "Deploy from a branch", branch `main`, folder `/ (root)`.
4. After about a minute the site is live at `https://<username>.github.io/<repo-name>/`.

Any change you commit redeploys automatically. Hard-refresh (Cmd+Shift+R) to skip the browser's cached copy.

The link-preview tags near the top of `index.html` (`og:url` and `og:image`) contain the site's full
address. If the site ever moves to a different address, update those two lines.

## The math

- **Rating** = 1000 + ( (W + 10 x p) / (N + 10) - p ) x 2000
  - W = the player's wins, N = their games played
  - The 10 "imaginary games" keep small samples from swinging the ranking. As N grows the
    adjustment fades (50% weight on your real record at 10 games, 86% at 60).
  - One percentage point above or below p is worth 20 rating points.
  - **p depends on the Rating toggle:**
    - **Table-adjusted (default):** p is the player's own expected win %, so each player is measured
      against what their table sizes predict. 1,000 means winning exactly as often as expected.
    - **Group average:** p is the group's win rate, everyone's wins / everyone's games. This is how the
      spreadsheet's Stats tab calculates Rating, so use it to match the sheet.
- **Expected win %** = the average of 1 / (players at the table) over the games a player took part in.
  It is 25% at a 4-player table and 33% at a 3-player table.
- **Luck** = win % - expected win %, in percentage points.
- **Arrows** compare a rating with the rating after the previous game night.
- **Period** recalculates everything from only the games in that period, so a month's ratings start
  fresh on the first of the month.

The site's "How ratings work" section explains all of this in more detail, with worked examples.

## Troubleshooting

| What you see | What it means |
| --- | --- |
| "Live from the sheet" | Everything is working. |
| "Couldn't reach the sheet. Showing data from ..." | Google didn't respond. The page is showing the last data this browser loaded. Press Refresh, or check the link in `config.js`. |
| "Couldn't reach the sheet, showing a saved copy" | Same, but this browser has never loaded live data, so it's using `snapshot.js`. Check the link in `config.js` and that the Log tab is still published. |
| "Saved copy of the game log" | `config.js` has no sheet link, so the site never tries the sheet. |
| A new game isn't showing | Google can take a few minutes to refresh the published CSV. Wait, press Refresh, or hard-refresh. |
| Red note: "G12 has 2 winners" | That game in the Log tab doesn't have exactly one Win row. Fix it in the sheet. |
| Edited a response but the site looks the same | Give Google a few minutes to refresh the published CSV, then press Refresh. If the Log tab itself didn't change, the edit trigger isn't running: check that `onFormEdit` exists on the spreadsheet's Apps Script project, and that column D of that row has no warning. |
| Old version of the page | Hard-refresh (Cmd+Shift+R). Browsers cache the page for a few minutes after a deploy. |
| Link preview shows the old image or none | Chat apps cache previews. Paste the link with `?v=2` on the end to force a fresh one. |
| "No games in this period" | The Period menu is set to a period with no games. Choose All time. |

## Known limitations

- **The site trusts the Log tab.** If a game in the Log has no winner or more than one, the site shows
  a red note naming it, but it can't tell that a winner was entered wrongly. Corrections are made by
  editing the form response (or the Log rows directly).
- **The rating doesn't know who you played.** A win at a strong table counts the same as a win at a
  weak one.
- **Table-adjusted ratings don't match the spreadsheet's Stats tab.** Switch the Rating toggle to
  Group average to see the sheet's numbers.
- Updates are not instant: they follow Google's refresh of the published CSV.

## Refresh the saved copy (optional)

`snapshot.js` only matters when a browser has never been able to load the sheet. To update it from an
xlsx download of the sheet:

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
