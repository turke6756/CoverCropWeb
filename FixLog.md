# FixLog — webApp.html correctness pass

Baseline: HEAD `894944e`. `Web_App/Data/webApp.html` (59,801 bytes) and
`Web_App/Data/manifest.json` were both verified unmodified in the working tree
before editing, so the line numbers in the brief matched exactly.

Nothing is committed. All edits are left in the working tree.

Method note: every claim below was checked against the data files the app
actually loads (`streaks.json`, `2017..2023.json`, `manifest.json`) or against
the code that computes the value — never against UI copy, comments, or config
constants.

---

## TIER 1 — fixes applied

### A. "7+ consecutive seasons" streak filter matches zero orchards

**Claim:** the record is only six winters, so `longest >= 7` is always empty.

**Verified: YES — empirically, not by inference.** Across all 7,448 records in
`streaks.json` the `longest` field distribution is:

| longest | 1 | 2 | 3 | 4 | 5 | 6 | 7+ |
|---|---|---|---|---|---|---|---|
| orchards | 4306 | 1788 | 817 | 249 | 153 | 135 | **0** |

Max `longest` = 6. `hasConsecutiveYears()` (webApp.html:554-560) is
`(streaksData[orchId]?.longest || 0) >= minConsecutive`, so `cover7` can never
match. A user could tick "Cover cropped 7+ consecutive seasons" and get a blank
map with no explanation.

**Changed:** removed the `chkCover7` checkbox (:140), its entry in
`getTimeAgnosticFilters()` (:642), both `filters.cover7` filter clauses (:685,
:705), and `chkCover7` from the listener-registration array (:1311).

**Deliberately did NOT:** touch `chkCover6` (135 orchards match — it is a real,
non-empty option), or change `hasConsecutiveYears`. I also left the now-unused
`consecutiveStreak7Plus` field in the `analyzeOrchardPatterns` fallback object
(:592) — it is inert dead schema in a fallback path, and ripping it out is
unrelated churn.

### B. manifest lists 2017, but 2017.json is a stub

**Claim:** 2017.json holds 3 orchards (792 bytes) — an early-export stub, not a
season. The scrubber offers a season that renders nothing.

**Verified: YES, with one correction to the wording.** `2017.json` is 792 bytes
and contains exactly **3** orchards. All three are `cluster_role_mapped: "Bare"`
— the file contains **zero** Cover records. Every other season file holds
**16,365** orchards (~4.5 MB each).

Correction: the 2017 scrubber position renders *3 orchards*, not literally
nothing. Substantively the claim holds — 3 of 16,365 is a blank map — but it is
not a zero-row season, and `buildRowsFromSeasons` (:382) does successfully load
it rather than erroring.

**Verified the fix is safe before applying it.** `manifest.years` is read in
only three places: `buildRowsFromSeasons` (:383, drives the scrubber domain),
a `years.length` fallback for a season count (:962), and an emptiness guard
(:1463). Critically, the `streaks.json` `mask` decoder (`wasActiveInYear`, :562)
does **not** read `manifest.years` — it hardcodes its own offset — so dropping
2017 from the year list cannot shift any mask bit interpretation.

**Changed:** removed `2017` from `manifest.json` `years`.

**Deliberately did NOT:** delete `2017.json` (explicitly out of scope), and did
not touch `manifest.json`'s `bit_order` field — see finding H, which is a real
problem but not one I should resolve unilaterally.

### C. UI copy is false twice (:184, :193)

**Claim 1 — opacity does not encode confidence for uncertain orchards.**

**Verified: YES.** webApp.html:1039-1041:

```js
if (confidence <= UNCERTAIN_THRESHOLD) {
  const intensity = Math.max(0.5, confidence * 3);
  return [80, 80, 80, Math.floor(255 * intensity)];
}
```

This branch only runs when `confidence <= 0.15`, so `confidence * 3 <= 0.45`,
which is always below the `Math.max` floor of 0.5. `intensity` is a **constant
0.5** across the entire uncertain range — alpha is always exactly 127.

Confirmed against the data rather than the constant: in `2019.json` the 3,360
records with `uncertain: true` have `best_soft_norm` spanning `0.0 → 0.1499394`,
and the 13,005 with `uncertain: false` span `0.1501086 → 1.0`. The uncertain
population never reaches the 0.1667 that `conf*3` would need to clear the floor.
So the gray is flat for every uncertain orchard in the real data, not just in
principle.

(Note: the app never reads the JSON `uncertain` flag — it recomputes uncertainty
from `best_soft_norm <= UNCERTAIN_THRESHOLD`. The two agree exactly on this
data, so this is consistent, not a bug.)

**Claim 2 — statistics do not uniformly exclude uncertain orchards.**

**Verified: YES.** In the live stats path `updateStatsFromRows` (:1510-1567),
cover/bare/percentage/acreage are computed from `reliable`
(`conf(r) > UNCERTAIN_THRESHOLD`, :1532), but the flipped count at :1561 is
computed from `rowsThisSeason` — the population *before* the uncertainty split.
So Flipped includes uncertain orchards while every other stat excludes them.

The same inconsistency exists in the second stats writer at :851
(`nFlip` from `dataWithValuesFiltered`), which is the non-PMTiles path.

**Changed:** rewrote both copy blocks (:184 and :193) to describe actual
behavior — gray is a flat fill for uncertain orchards, opacity tracks confidence
only for confident ones, and Flipped is explicitly called out as all-population.

### D. Misleading comment at :1040

**Claim:** "Scale 0-0.15 to 0.5-0.45" describes a mapping `Math.max` defeats,
and is decreasing besides.

**Verified: YES, on both counts.** The stated range is decreasing (0.5 → 0.45,
i.e. *less* opaque as confidence rises, which would be backwards even if it
worked), and `Math.max(0.5, …)` clamps the whole thing to a constant 0.5 anyway.
The comment describes neither the intent nor the behavior.

**Changed:** replaced with a comment stating the actual behavior (constant 0.5)
and flagging that the `confidence * 3` term is inert, so the next reader does
not "fix" the comment to match a mapping that never runs.

**Deliberately did NOT:** change the *behavior* — that is Tier 2 item E, a
design decision. The fill is byte-for-byte identical after my edit.

---

## FURTHER FINDINGS (not fixed — reported per the ALSO section)

### H. `streaks.json` does not reconcile with the season files (significant)

This is the most serious thing I found, and I did **not** fix it — it needs the
pipeline, which is out of scope for this pass.

The time-agnostic streak filters are driven entirely by `streaks.json`. That
file's `mask`/`longest` cannot be reproduced from the season JSONs the same app
renders:

- **Internally consistent:** for all 7,448 records, `longest` equals the longest
  run of consecutive set bits in `mask`, and `years_active` equals its popcount.
  So the file is self-consistent.
- **Externally inconsistent:** defining confident cover as
  `cluster_role_mapped == "Cover" AND not uncertain`, the best-fitting bit
  alignment (bit0 = 2018) reproduces the mask for only **5,794 of 7,446**
  orchards (~78%). The ~1,652 mismatches are *not* an ID-coverage artifact — I
  checked, and 7,446 of 7,448 streak IDs are present in all six season files.
- **No alignment fixes it:** bit0=2017 → 1,639/7,446; latest=bit0 → 688/7,448.

So for roughly a fifth of ever-cover orchards, the streak filters disagree with
what the season-by-season map shows for the same orchard. A user filtering
"3+ consecutive seasons" and then scrubbing the timeline can see a contradiction.

**Recommendation:** regenerate `streaks.json` from the shipped season files, or
document it as a separate pipeline product with its own vintage. Do not paper
over it in the UI.

### I. `mask` bit order is genuinely ambiguous, and both stated sources are wrong

`manifest.json` declares `"bit_order":"earliest=bit0"` with `years[0] == 2017`,
which means bit0 = 2017. webApp.html:565 hardcodes `year - 2018; // 2018=bit0`.
**These contradict each other, and the data refutes both:**

- Against bit0=2017: `2017.json` contains zero Cover records, yet **2,109**
  records have bit0 set. 2017 cannot be bit0.
- Against bit0=2018: the mask uses **7** bit positions (bit6 is set in 533
  records, max mask 126 = `0b1111110`), but 2018–2023 is only six years. Under
  bit0=2018, bit6 would be 2024 — a season that does not exist in this dataset.

**Live impact: none today.** `wasActiveInYear` (:562-573) is the only mask
consumer and it is **never called** — I grepped every reference. The off-by-one
is latent, not user-visible. That is exactly why it is dangerous: the next
person to wire up that function inherits a silently-wrong year mapping.

**Recommendation:** resolve the bit order at the pipeline, then make the decoder
derive its offset from `manifest.years` instead of hardcoding a literal, and fix
`bit_order` or delete it. I did not touch this because guessing between three
mutually-inconsistent sources is exactly the kind of propagation this pass is
meant to prevent. Note my fix B leaves `bit_order: "earliest=bit0"` in the
manifest — with 2017 removed, that string now *reads* as "bit0 = 2018", which
matches the code comment but still contradicts the 7-bit evidence. Flagging
explicitly: **fix B does not resolve finding I, and does not make it worse.**

### J. `never_cover` is always false in streaks.json; the filter works by accident

Every one of the 7,448 records has `ever_cover: true` and `never_cover: false` —
`streaks.json` only contains orchards that *did* cover crop at some point. So
the `filters.neverCover && data.never_cover` clause at :679 can never fire.

The "Never cover cropped confidently" filter nonetheless works, via a different
branch: :669-675 treats an orchard absent from `streaks.json` as never-cover and
returns `filters.neverCover`. (16,365 total − 7,448 ever-cover = 8,917 orchards
take that path.)

Not a user-visible bug — the feature is correct. But :679 is dead and misleading.
**Recommendation:** low priority; drop the dead clause or add a note that
never-cover is encoded by absence.

### K. `nMixed` is dead in the JSON path

`mapRole` (:1535) prefers `r.cluster_role_mapped`, but rows built by
`buildRowsFromSeasons` (:395-412) never set that field — only `cluster_role`.
It therefore always falls through to the ternary, which emits only `'Active'` or
`'Baseline'`, never `'Mixed'`. So `nMixed` (:1541) is always 0.

Harmless: `nMixed` is computed but never displayed — the "Uncertain" stat slot
shows `nGray` instead (:1559). No user-visible effect. Not fixed; noted so the
next reader does not mistake it for a live counter.

### M. Blank map under `python -m http.server` — NOT caused by this pass

Reported as a suspected regression: page renders, UI navigates, zero orchards
draw. **Diagnosed as pre-existing and environmental. Our diff is exonerated by
direct A/B.** Not fixed — diagnosis only.

**Root cause:** `python -m http.server` does not implement HTTP Range
(byte-serving). All geometry in this app comes from `orchards.pmtiles`
(`CONFIG.geometryMode: 'pmtiles'`, `centroidsUrl: null`), and PMTiles is built
entirely on range requests. The server answers a `Range: bytes=0-127` request
with **HTTP 200 and the whole 3,474,288-byte file**, no `Content-Range`, no
`Accept-Ranges`. The PMTiles client detects this and throws:

> `Server returned no content-length header or content-length exceeding request.
> Check that your storage backend supports HTTP Byte Serving.`

No tiles load → no geometry → nothing draws, regardless of the season JSONs.

**Evidence (all reproducible):**

1. **A/B against HEAD.** Extracted unmodified `webApp.html` from `894944e` to a
   temp dir outside the repo, hardlinked the same data files, served on :8001.
   Drove the real `pmtiles@3.2.1` client against both ports: **HEAD and ours
   throw the identical error.** The blank map predates this pass.
2. **The archive is fine.** Parsed the PMTiles v3 header from disk: valid magic,
   spec 3, 104 tiles, MVT, zoom 0–10, layer `orchards`, bounds
   `-122.7,34.9 → -118.5,40.4` — covering the initial view (`-120.5, 36.5` @ z7).
3. **Every URL serves 200.** manifest, streaks, all six season JSONs, and the
   pmtiles file. The only 404s are `viz_table.csv` and
   `FresnoAlmondOrchards.geojson` — never requested (`dataMode: 'json'`,
   `centroidsUrl: null`). No missing data.
4. **Proof of remedy.** Served the *same* files from a Range-capable server on
   :8002 → `206 Partial Content` → PMTiles reads cleanly: header OK, and the z7
   tile at the initial view returns **305,880 bytes of MVT geometry**. Same
   files, same code, different server, geometry loads.
5. **No dangling refs from our diff.** Statically resolved every
   `getElementById` and listener-array id against the DOM in both versions:
   **zero** dangling refs in either (ours 35 ids/29 refs, HEAD 36/30 — the
   deltas are exactly the removed `chkCover7`). The removal was clean.
   Independently, `filterOrchardsByPattern` only runs when `isTimeAgnostic` is
   true (default false), so the streak filters are not on the default render
   path at all. Dropping 2017 only shortens the slider domain to
   `[2018..2023]`; the default position was already the last season.

**Recommended fix (NOT applied):** serve with a range-capable static server.
Any of: `npx http-server -p 8000 --cors`, `python -m RangeHTTPServer 8000`, or
nginx/Caddy. GitHub Pages and S3/CloudFront all support byte serving, so a real
deployment is unaffected — this only bites local preview via `http.server`.
Worth a line in the README next to the preview instructions, since the naive
command silently produces an empty map with no on-page error.

**Caveat, stated plainly:** the Range failure is *sufficient* to explain a blank
map, and it is total — no tiles load at all. It is not proof that nothing else
is wrong downstream. Once tiles actually load, the MVT `orch_id` ↔ season-JSON
join could surface further issues that this failure currently masks. I have not
seen the app render, in any version.

### L. Debug logging left in a hot path

:659-661, :670-672 and :688-690 log to the console for any orchard whose ID
contains the substring `2bd1`, inside `filterOrchardsByPattern` — which runs per
orchard per render. Not a correctness bug and explicitly not perf work, so not
touched. Noted only because it is leftover debug scaffolding in shipped code.

---

## TIER 2 — recommendations only, nothing implemented

### E. Should uncertain opacity encode confidence, or should flat gray stay?

**Recommendation: keep the flat gray. Do not make opacity encode confidence.**

The bug was the copy, not the rendering — and I have now fixed the copy. Reasons
to keep flat gray:

1. The uncertain band is `[0, 0.15]`. Encoding it as opacity would spread a
   15-point range across a visible alpha ramp, making a 0.02-confidence orchard
   and a 0.14-confidence orchard look meaningfully different when the honest
   message is "we don't know" for both. That is false precision.
2. Flat gray reads categorically — "uncertain" is a *class*, not a magnitude.
   That matches how the stats treat it (excluded wholesale, not weighted).
3. Confident orchards already use opacity for confidence (:1045), so a flat gray
   is a clear visual signal that a *different* rule applies.

**Diff I would make if you disagree** — restore a real, increasing ramp by
dropping the floor that defeats it:

```diff
   if (confidence <= UNCERTAIN_THRESHOLD) {
-    const intensity = Math.max(0.5, confidence * 3);
-    return [80, 80, 80, Math.floor(255 * intensity)];
+    // Map [0, 0.15] -> [0.25, 0.55] alpha: increasing, and no floor to defeat it
+    const intensity = 0.25 + (confidence / UNCERTAIN_THRESHOLD) * 0.30;
+    return [80, 80, 80, Math.floor(255 * intensity)];
   }
```

Note this derives from `UNCERTAIN_THRESHOLD` rather than hardcoding `3`, so it
survives a manifest threshold change — the current `* 3` silently assumes 0.15.
If you take this, the :184/:193 copy I wrote must change back to claiming
confidence-encoded opacity.

### F. Should the flipped count exclude uncertain orchards?

**Recommendation: exclude uncertain, making it consistent with every other
stat.** Change `rowsThisSeason` → `reliable` at :1561.

Rationale: `flipped_this_season` for an uncertain orchard is a flip *between two
classifications the app is simultaneously telling the user it does not trust*.
Counting it as a flip in a panel where every neighbouring stat has already
dropped that orchard invites a real misreading — the four stat tiles sit in one
grid and visibly fail to reconcile (Cover + Uncertain + Bare are drawn from a
partition that Flipped is not). Consistency across one stat block is worth more
than the extra sensitivity.

**Diff I would make:**

```diff
-    document.getElementById('statFlipped').textContent = (rowsThisSeason.filter(r => r.flipped_this_season).length).toLocaleString();
+    document.getElementById('statFlipped').textContent = (reliable.filter(r => r.flipped_this_season).length).toLocaleString();
```

The same change would apply at :851 (`dataWithValuesFiltered` → `reliableDataFiltered`)
for the non-PMTiles path, to keep the two writers in agreement.

I did **not** apply this — it changes a displayed number, which is a product
decision, and my Tier 1 copy fix already removes the dishonesty by labeling the
count as all-population. If you take F, the :184/:193 copy should drop the
"Flipped is the exception" sentence and go back to a clean "statistics exclude
uncertain orchards".

### G. Is `confidence_bucket` dead schema?

**Confirmed dead. Recommendation: drop it from the pipeline export, but not in
this pass.**

Evidence:
- **Null in every record, not just 2019.** I checked 2018, 2019 and 2023:
  16,365/16,365 records are `null` in each. The 3-record 2017 stub is also null.
- **Referenced nowhere in webApp.html.** Grepped — zero hits outside the data.

So it is inert in both producer output and consumer. It costs ~30 bytes/record
(~0.5 MB per season file, ~3 MB across the shipped set) for a field that is
always null.

**Recommendation:** stop emitting it from the export step and drop it from the
season JSONs on the next regeneration. I did not do this because (a) it means
regenerating the season files, which is a data change well outside a
correctness/honesty pass, and (b) the emitting code path lives in the pipeline,
and `RewardCalculationPipline/Reward_Pipeline_and_Clustering.py` is off-limits
this pass. If the field was intended to carry a discretized confidence class,
the decision is "implement it or delete it" — leaving a null column in a
published artifact implies a signal that does not exist.
