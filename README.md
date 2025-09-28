# England-And-Wales-House-Price-Visualisation

This script builds a year-by-year choropleth of average UK local-authority house prices and exports it as an animated GIF.

---

## What it does

* Loads **ONS house price data** (CSV) with columns like `Year ending Dec YYYY`.
* Loads **Local Authority Districts (LAD) boundaries** (shapefile, Dec 2022).
* Joins by local authority code, renders a log-scaled choropleth for each December, and stitches frames into a GIF.

Output:

* A folder of PNG frames (`frames_fancy/`)
* An animation `house_price_choropleth_fancy.gif` (2 fps)

