# --- paths (edit these) ---
pricedatapath = r"C:\Users\Will\Desktop\Data Projects VS code folder\Data projects\House Prices\Datasets\House_Prices_Yearly_Regional_Formatted.csv"
shapepath     = r"C:\Users\Will\Desktop\Data Projects VS code folder\Data projects\House Prices\Datasets\Local_Authority_Districts_December_2022_UK_BUC_V2_3106738338966916168\LAD_DEC_2022_UK_BUC_V2.shp"
out_dir       = "frames_fancy"
gif_path      = "house_price_choropleth_fancy.gif"

# --- imports ---
import os, re, math
import numpy as np
import imageio.v2 as imageio
import pandas as pd
import geopandas as gpd
import matplotlib.pyplot as plt
from matplotlib.colors import LogNorm
from matplotlib.cm import ScalarMappable
from matplotlib.ticker import FuncFormatter
import matplotlib.patheffects as pe

# ------------------------
# Helper: nice log ticks (£)
# ------------------------
def log_ticks(vmin, vmax):
    if vmin <= 0 or vmax <= 0:
        return []
    lo = int(math.floor(math.log10(vmin)))
    hi = int(math.ceil(math.log10(vmax)))
    ticks = []
    for e in range(lo, hi + 1):
        for b in (1, 2, 5):
            val = b * (10 ** e)
            if vmin <= val <= vmax:
                ticks.append(val)
    return sorted(set(ticks))

# --- load price CSV ---
pricedata = pd.read_csv(pricedatapath)
pricedata.columns = [c.strip() for c in pricedata.columns]

# Rename CSV join keys to LAD22*
rename_map = {}
if "Local authority code" in pricedata.columns:
    rename_map["Local authority code"] = "LAD22CD"
if "Local authority code " in pricedata.columns:
    rename_map["Local authority code "] = "LAD22CD"
if "Local authority name" in pricedata.columns:
    rename_map["Local authority name"] = "LAD22NM"
pricedata = pricedata.rename(columns=rename_map)

# --- load LAD polygons (Dec 2022) ---
gdf = gpd.read_file(shapepath)

# Ensure the shapefile exposes LAD22CD; fall back if it's LAD24CD
if "LAD22CD" not in gdf.columns:
    if "LAD24CD" in gdf.columns:
        gdf["LAD22CD"] = gdf["LAD24CD"]
    else:
        raise KeyError("Join key not found in shapefile: expected 'LAD22CD' (or 'LAD24CD').")

# Trim join keys
pricedata["LAD22CD"] = pricedata["LAD22CD"].astype(str).str.strip()
gdf["LAD22CD"] = gdf["LAD22CD"].astype(str).str.strip()

# Merge attributes into polygons (join on LAD22CD)
gdf = gdf.merge(pricedata, on="LAD22CD", how="left")

# Reproject to EPSG:27700 (meters) for nice cartographic layout
gdf = gdf.to_crs(27700)

# Optional: simplify geometry for faster, cleaner rendering
plot_gdf = gdf.copy()
plot_gdf["geometry"] = plot_gdf.geometry.simplify(50, preserve_topology=True)

# Dissolve once for a neat outline stroke
outline = plot_gdf.dissolve().boundary

# --- pick December-only columns & order by year ---
dec_cols = sorted(
    [c for c in pricedata.columns if re.match(r"^Year ending\s+Dec\s+\d{4}$", c)],
    key=lambda c: int(re.search(r"(\d{4})$", c).group(1)),
)
if not dec_cols:
    raise ValueError("No columns found matching 'Year ending Dec YYYY'. Check your CSV headers.")

# Ensure numeric
for c in dec_cols:
    plot_gdf[c] = pd.to_numeric(plot_gdf[c], errors="coerce")

# Global log scale across all Decembers
all_vals = pd.concat([plot_gdf[c] for c in dec_cols], axis=0)
vmin = float(all_vals[all_vals > 0].min(skipna=True))
vmax = float(all_vals.max(skipna=True))
if not np.isfinite(vmin) or not np.isfinite(vmax) or vmax <= 0:
    raise ValueError("Log scale needs positive values; found no positive prices.")
norm  = LogNorm(vmin=vmin, vmax=vmax)
cmap  = "inferno"
ticks = log_ticks(vmin, vmax)
fmt   = FuncFormatter(lambda x, pos: f"£{x:,.0f}")

# Output folder
os.makedirs(out_dir, exist_ok=True)

# ------------------------
# Render frames (polished)
# ------------------------
for col in dec_cols:
    year = re.search(r"(\d{4})$", col).group(1)

    fig = plt.figure(figsize=(10, 8), dpi=150, facecolor="#f8f8f8")
    ax  = fig.add_axes([0.06, 0.10, 0.78, 0.80])  # map area
    cax = fig.add_axes([0.86, 0.18, 0.03, 0.62])  # colorbar axis

    # Values (mask non-positive as missing)
    vals = plot_gdf[col].where(plot_gdf[col] > 0, np.nan)

    # Choropleth
    ax.set_aspect("equal"); ax.set_axis_off()
    plot_gdf.assign(_v=vals).plot(
        column="_v", ax=ax, cmap=cmap, norm=norm,
        linewidth=0.2, edgecolor="white", alpha=0.95,
        missing_kwds={"color": "#d9d9d9", "hatch": "///", "label": "No data"},
    )

    # Subtle drop shadow + crisp outer boundary
    if ax.collections:
        ax.collections[-1].set_path_effects([
            pe.Stroke(linewidth=0.6, foreground="0.2", alpha=0.25),
            pe.Normal()
        ])
    outline.plot(ax=ax, color="none", linewidth=2.2, alpha=0.25)  # shadow
    outline.plot(ax=ax, color="black", linewidth=0.6, alpha=0.6)  # crisp line

    # Clean title block (no overlap)
    fig.suptitle("UK Local Authority House Prices", x=0.06, y=0.93,
                 ha="left", fontsize=18, fontweight="bold", color="#222222")
    fig.text(0.06, 0.885, f"Year ending December {year}",
             ha="left", fontsize=12, color="#4a4a4a")

    # Colorbar with custom £ ticks
    sm = ScalarMappable(norm=norm, cmap=cmap); sm.set_array([])
    cbar = plt.colorbar(sm, cax=cax)
    if ticks:
        cbar.set_ticks(ticks)
        cbar.set_ticklabels([fmt(t, None) for t in ticks])
    cbar.set_label("Average house price (log colour scale)", fontsize=10)
    for tick in cbar.ax.yaxis.get_ticklabels():
        tick.set_fontsize(9)

    # Credits
    fig.text(0.06, 0.04, "Data: ONS • Viz: Will", ha="left", fontsize=9, color="#6e6e6e")

    # Save frame
    frame_path = os.path.join(out_dir, f"{col.replace(' ', '_')}.png")
    plt.savefig(frame_path, facecolor=fig.get_facecolor())
    plt.close(fig)

# ------------------------
# Build GIF
# ------------------------
images = [imageio.imread(os.path.join(out_dir, f"{c.replace(' ', '_')}.png")) for c in dec_cols]
imageio.mimsave(gif_path, images, fps=2)
print(f"GIF saved to {gif_path}")
