# Cover Crop Explorer

**An interactive map that uses satellite imagery to estimate which California almond orchards planted winter cover crops — across six winters and nine counties.**

![Screenshot of the Cover Crop Explorer: a map of California almond orchards shaded by estimated winter cover-crop behavior](assets/hero.png)

*A cover crop is a plant (grass, clover, a mix) that a grower puts in the ground over winter instead of leaving bare dirt between the tree rows. It can improve the soil's health. This project tries to spot that habit from space — no field visits — by watching how green or bare the ground between the almond trees looks each winter.*

**At a glance:** 16,365 orchards · 9 counties · 6 winters (2018–2023) · a fast, self-contained web page with no server behind it.

---

## Why I built this

> I built this because I was learning a lot about remote sensing and I have a long-term interest in agriculture and sustainability. I wanted to try my hand at building a web app — the whole thing started with a class I took at an AGU conference on how to host these kinds of apps. I believe cover cropping is a really important method farmers can use to enhance long-term soil health and nutrient management. And it isn't just about *tracking* who's cover cropping — it's about analyzing the data behind who's doing it, when, where, and why. Mostly I just wanted to see what was possible. So you could call it an experiment. It's a work in progress, and it's something that can be improved as new sensor technology comes online. Instead of performing a census, you can do it from space.
>
> — Edward Turk

To be clear about what that means for the reader: **this is an experiment and a work in progress, not a published finding.** The numbers below are estimates the model made from pictures of the ground. They have not been checked against what growers actually planted (more on that under "What it can't tell you").

---

## What it does

Open the app and you get a map of Central Valley almond orchards, each one shaded by whether the model thinks it was cover-cropped that winter. From there you can:

- **Move through time.** A slider steps you across the six winters, one at a time, and the map recolors for the season you land on.
- **Click a single orchard.** You get that one orchard's history — its estimated behavior across every winter in the record, drawn as a small chart.
- **Filter by habit.** You can ask to see only orchards that cover-cropped several winters in a row, to separate the steady adopters from the one-off tries.
- **See the model's doubt.** Orchards the model wasn't confident about are shown in a plain gray instead of a confident color, so a low-certainty guess never masquerades as a firm one.

**One quirk worth knowing:** a winter spans two calendar years, and each season here is named for the year it *starts*. So "2023" means the winter that runs from about November 2023 into March 2024. The six winters go from 2018–19 through 2023–24.

---

## How it works

No field visits are involved. The whole thing is built from satellite pictures and arithmetic. In plain terms, five steps:

1. **Pick the orchards.** Start from a public map of U.S. crop fields and keep only the parcels that were reliably labeled "almond" for years running — that drops fields in rotation or recently replanted, so we're watching stable orchards.
2. **Measure the ground each winter.** Using free imagery from the European **Sentinel-2** satellites, the pipeline measures, for every orchard, how *green* the ground looks and how *bare* the soil looks across the winter. Green cover between the tree rows is the tell-tale sign of a planted cover crop.
3. **Account for tree age.** Old, full-canopy trees look different from young, sparse ones from above, and that can fool a simple greenness reading. So each orchard is first sorted into a maturity class ("young" or "old") and then judged against orchards of its own size — not against one valley-wide yardstick.
4. **Classify the behavior.** The seasonal measurements are grouped into likely patterns, and each orchard-winter is called either **cover-cropped** or **bare**.
5. **Flag the shaky calls.** Every call carries a confidence score. Anything below a set threshold is marked *uncertain* and shown in gray, and it's left out of the headline counts so the totals only reflect calls the model could make with some confidence.

---

## What it found

Every number here is an **estimate the model made from imagery** — a read of what the ground looked like, not a confirmed record of what any grower did.

- **16,365 almond orchards** were tracked, across **9 counties** (Butte, Colusa, Fresno, Glenn, Kern, Madera, Merced, Stanislaus, and Tehama), over **6 winters**.
- In the **2023** winter, the model called **~2,700 orchards cover-cropped** and about **10,350 bare**, with roughly **3,300 more too uncertain to call**. Among the orchards it *could* call with confidence, that's about **21%** estimated cover-cropped (about 16% of all orchards tracked).
- Looking across all six winters, of the orchards that ever cover-cropped, several thousand did it two or more winters in a row, a few hundred kept it up for four or more, and **135 appear cover-cropped in all six** — the most the record can show.

Because the record is six winters long, a "streak" can be at most six seasons — the data simply doesn't reach back far enough to show anything longer.

---

## What it can't tell you

The estimates above are only as good as their caveats, so here they are plainly:

- **No ground-truth check.** This is the big one. **Nothing here has been compared against what growers actually planted.** The classification is defensible from the imagery and consistent across the six winters, but it has never been validated against a single grower's own records. Treat every figure as an informed estimate, not a fact about the ground.
- **Almonds only.** The whole study is one crop. It says nothing about other orchards or other crops.
- **Not everything on the map was studied.** The map draws far more orchard shapes (across more counties) than the 16,365 that actually carry cover-crop estimates. What you *see* is wider than what was *analyzed*.
- **Fixed calendar windows, not weather.** The winter is measured on the same set of dates every year. A late-rain year and an early one are read on the identical schedule, with no weather adjustment.
- **It's a heavy page.** A fresh load pulls tens of megabytes of data, so the first open can be slow.

---

## The tech, briefly

*(This is the one section written for a technical reader — skip it if that's not you.)* The imagery and orchard measurements are produced in **Google Earth Engine** from **Sentinel-2** surface reflectance. Scoring and clustering (including the tree-age stratification and the confidence scoring) run in **Python** with scikit-learn, pandas, and NumPy. The web app is fully static — no backend, no build step, no API keys — using **deck.gl** and **MapLibre GL** for rendering, **PMTiles** for the vector map tiles, and Plotly for the per-orchard charts.

---

## How to verify this

Every figure in this README is reproducible from the code and the shipped data in this repository — nothing here is hand-entered. The Google Earth Engine script, the Python scoring pipeline, and the per-orchard results are all included, so any number above can be traced back to the step that produced it.

- **Source code and data:** everything lives in this repository, at commit `97ef317`.
- **Run the app yourself:** it's fully static — serve `Web_App/Data/` with any local web server (for example, `python -m http.server 8000` from that folder) and open `webApp.html`.

*Built by Edward Turk. An experiment and a work in progress.*
