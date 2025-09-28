# --- paths (edit these) ---
pricedatapath = r"C:\Users\Will\Desktop\Data Projects VS code folder\Data projects\House Prices\Datasets\House_Prices_Yearly_Regional_Formatted.csv"
shapepath     = r"C:\Users\Will\Desktop\Data Projects VS code folder\Data projects\House Prices\Datasets\Local_Authority_Districts_December_2022_UK_BUC_V2_3106738338966916168\LAD_DEC_2022_UK_BUC_V2.shp"
html_out      = "uk_lad_prices_interactive.html"

# --- imports ---
import re, math, json, numpy as np, pandas as pd, geopandas as gpd
import plotly.express as px

# ------------------------
# Helper: nice 1–2–5 log ticks
# ------------------------
def log_ticks(vmin, vmax):
    if vmin <= 0 or vmax <= 0:
        return []
    lo = int(math.floor(math.log10(vmin)))
    hi = int(math.ceil(math.log10(vmax)))
    ticks = []
    for e in range(lo, hi + 1):
        for b in (1, 2, 5):
            v = b * (10 ** e)
            if vmin <= v <= vmax:
                ticks.append(v)
    return sorted(set(ticks))

# --- load price CSV (rename to LAD22*) ---
pricedata = pd.read_csv(pricedatapath)
pricedata.columns = [c.strip() for c in pricedata.columns]
rename_map = {}
if "Local authority code" in pricedata.columns:  rename_map["Local authority code"]  = "LAD22CD"
if "Local authority code " in pricedata.columns: rename_map["Local authority code "] = "LAD22CD"
if "Local authority name" in pricedata.columns:  rename_map["Local authority name"]  = "LAD22NM"
pricedata = pricedata.rename(columns=rename_map)

# --- load LAD 2022 shapefile (ensure LAD22CD exists) ---
gdf = gpd.read_file(shapepath)
if "LAD22CD" not in gdf.columns:
    if "LAD24CD" in gdf.columns:
        gdf["LAD22CD"] = gdf["LAD24CD"]
    else:
        raise KeyError("Expected 'LAD22CD' (or 'LAD24CD') in shapefile.")

# tidy join keys
pricedata["LAD22CD"] = pricedata["LAD22CD"].astype(str).str.strip()
gdf["LAD22CD"]       = gdf["LAD22CD"].astype(str).str.strip()

# --- merge attributes into polygons (keeps names if present in CSV) ---
gdf = gdf.merge(pricedata, on="LAD22CD", how="left")

# --- find December-only columns & sort by year ---
dec_cols = [c for c in pricedata.columns if re.match(r"^Year ending\s+Dec\s+\d{4}$", c)]
dec_cols = sorted(dec_cols, key=lambda c: int(re.search(r"(\d{4})$", c).group(1)))
if not dec_cols:
    raise ValueError("No 'Year ending Dec YYYY' columns found.")

# --- prepare long table for Plotly (no geometry here) ---
attr_df = gdf[["LAD22CD"] + (["LAD22NM"] if "LAD22NM" in gdf.columns else []) + dec_cols].drop_duplicates("LAD22CD")
long_df = attr_df.melt(
    id_vars=[c for c in ["LAD22CD","LAD22NM"] if c in attr_df.columns],
    value_vars=dec_cols,
    var_name="period",
    value_name="price"
)
long_df["year"] = long_df["period"].str.extract(r"(\d{4})").astype(int)
long_df["price"] = pd.to_numeric(long_df["price"], errors="coerce")
# log values for colour mapping (keep original £ for hover)
long_df["log_price"] = np.where(long_df["price"] > 0, np.log10(long_df["price"]), np.nan)

# --- compute global colour range & ticks (in log space) ---
pos_prices = long_df.loc[long_df["price"] > 0, "price"]
vmin, vmax = float(pos_prices.min()), float(pos_prices.max())
tick_vals  = log_ticks(vmin, vmax)
tick_text  = [f"£{v:,.0f}" for v in tick_vals]
tick_locs  = [math.log10(v) for v in tick_vals]

# --- build a lightweight GeoJSON from polygons ---
# simplify in meters for speed, then convert to WGS84 (EPSG:4326)
geo = gdf[["LAD22CD","geometry"]].drop_duplicates("LAD22CD").copy()
# If not projected, project temporarily to 27700 for simplification
if geo.crs is None:
    raise ValueError("Shapefile has no CRS; please set/confirm CRS before continuing.")
geo_m = geo.to_crs(27700)
geo_m["geometry"] = geo_m.geometry.simplify(50, preserve_topology=True)  # tweak tolerance for speed/quality
geo_wgs84 = geo_m.to_crs(4326)
geojson = json.loads(geo_wgs84.to_json())

# --- choose a nice sequential palette ---
color_scale = "Inferno"

# --- Plotly choropleth with slider (animation_frame) ---
fig = px.choropleth(
    long_df,
    geojson=geojson,
    locations="LAD22CD",
    featureidkey="properties.LAD22CD",
    color="log_price",                      # log mapped
    animation_frame="year",                 # slider!
    hover_name="LAD22NM" if "LAD22NM" in long_df.columns else None,
    hover_data={
        "LAD22CD": False,
        "log_price": False,
        "price": ":,.0f",                   # show £ nicely
    },
    color_continuous_scale=color_scale,
    range_color=(math.log10(vmin), math.log10(vmax)),
)

# tidy the geo layout (no base map; pan/zoom still works)
fig.update_geos(
    fitbounds="locations",
    visible=False,          # hide graticules/axes
)

# colour bar in £ with log ticks
fig.update_coloraxes(
    colorbar=dict(
        title="Average house price (log colour)",
        tickvals=tick_locs,
        ticktext=tick_text,
        len=0.75,
    )
)

# layout polish
fig.update_layout(
    title=dict(
        text="UK Local Authority House Prices — Year ending December",
        x=0.01, xanchor="left", y=0.98,
        font=dict(size=20)
    ),
    margin=dict(l=10, r=10, t=50, b=10),
)

# save standalone HTML (double-click to open)
fig.write_html(html_out, include_plotlyjs="cdn")
print(f"Saved interactive map to {html_out}")
