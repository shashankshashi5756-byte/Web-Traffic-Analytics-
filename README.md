# Web-Traffic-Analytics-
import folium
from folium.plugins import MarkerCluster
import numpy as np
import pandas as pd

# ==============================================================================
# Step 1: Load & Clean Regional Dataset
# ==============================================================================
# Sample data generation (replace with: pd.read_csv('your_dataset.csv'))
data = {
    "city": [
        "New York",
        "Los Angeles",
        "Chicago",
        "Houston",
        "Phoenix",
        "Philadelphia",
        "San Antonio",
        "San Diego",
        "Dallas",
        "Austin",
    ],
    "state": ["NY", "CA", "IL", "TX", "AZ", "PA", "TX", "CA", "TX", "TX"],
    "postal_code": [
        "10001",
        "90001",
        "60601",
        "77001",
        "85001",
        "19101",
        "78201",
        "92101",
        "75201",
        "73301",
    ],
    "latitude": [
        40.7128,
        34.0522,
        41.8781,
        29.7604,
        33.4484,
        39.9526,
        29.4241,
        32.7157,
        32.7767,
        30.2672,
    ],
    "longitude": [
        -74.0060,
        -118.2437,
        -87.6298,
        -95.3698,
        -112.0740,
        -75.1652,
        -98.4936,
        -117.1611,
        -96.7970,
        -97.7431,
    ],
    "revenue": [
        950000,
        820000,
        610000,
        340000,
        180000,
        420000,
        150000,
        510000,
        290000,
        210000,
    ],
    "user_count": [
        12000,
        10500,
        8900,
        9200,
        6800,
        7100,
        6400,
        5800,
        8100,
        7300,
    ],
    "population": [
        8336817,
        3979576,
        2693976,
        2320268,
        1680992,
        1584064,
        1547253,
        1423851,
        1343573,
        978908,
    ],
}

df = pd.DataFrame(data)

# Data cleaning: drop missing coordinates and standardize postal codes
df = df.dropna(subset=["latitude", "longitude", "city", "state"])
df["postal_code"] = (
    df["postal_code"].astype(str).str.zfill(5)
)  # Ensure 5-digit format
df["city"] = df["city"].str.strip().str.title()
df["state"] = df["state"].str.strip().str.upper()

# ==============================================================================
# Step 2: Aggregate Metrics & Underserved Opportunity Scoring
# ==============================================================================
# Calculate key ratios
df["revenue_per_user"] = (df["revenue"] / df["user_count"]).round(2)
df["market_penetration_rate"] = (
    (df["user_count"] / df["population"]) * 100
).round(3)

# Underserved High-Potential Metric:
# High population base + decent revenue per user, but low penetration rate
df["expansion_opportunity_score"] = (
    (df["population"] / df["population"].max()) * 0.5
    + (1 - (df["market_penetration_rate"] / df["market_penetration_rate"].max()))
    * 0.5
).round(3)

# ==============================================================================
# Step 4: Identify Top 3 High-Potential Underserved Regions
# ==============================================================================
top_3_underserved = df.sort_values(
    by="expansion_opportunity_score", ascending=False
).head(3)
underserved_cities = set(top_3_underserved["city"])

print("--- Top 3 Underserved Regional Markets ---")
print(
    top_3_underserved[
        [
            "city",
            "state",
            "population",
            "revenue",
            "market_penetration_rate",
            "expansion_opportunity_score",
        ]
    ].to_string(index=False)
)

# ==============================================================================
# Step 3 & 5: Interactive Spatial Visualization (Folium Map)
# ==============================================================================
# Center map around US geographic centroid
m = folium.Map(location=[39.8283, -98.5795], zoom_start=4, tiles="CartoDB positron")

for _, row in df.iterrows():
    is_top_opportunity = row["city"] in underserved_cities

    # Marker styling: Red for top 3 expansion priorities, Blue for standard markets
    marker_color = "crimson" if is_top_opportunity else "#2A81CB"
    fill_color = "red" if is_top_opportunity else "#38AADD"
    radius = np.sqrt(row["revenue"]) / 40  # Proportional bubble size

    popup_html = f"""
    <div style="font-family: Arial; min-width: 180px;">
        <h4 style="margin-bottom: 5px; color: {'#d9534f' if is_top_opportunity else '#333'};">
            {row['city']}, {row['state']} {'⭐ (Priority)' if is_top_opportunity else ''}
        </h4>
        <b>Revenue:</b> ${row['revenue']:,}<br>
        <b>User Base:</b> {row['user_count']:,}<br>
        <b>Penetration:</b> {row['market_penetration_rate']}%<br>
        <b>Opportunity Score:</b> {row['expansion_opportunity_score']}
    </div>
    """

    folium.CircleMarker(
        location=[row["latitude"], row["longitude"]],
        radius=radius,
        popup=folium.Popup(popup_html, max_width=250),
        tooltip=f"{row['city']}: Opportunity Score {row['expansion_opportunity_score']}",
        color=marker_color,
        weight=2.5 if is_top_opportunity else 1,
        fill=True,
        fill_color=fill_color,
        fill_opacity=0.65,
    ).add_to(m)

# Export interactive map
output_file = "geospatial_analysis_map.html"
m.save(output_file)
print(f"\n[+] Interactive spatial map successfully saved to: {output_file}")
