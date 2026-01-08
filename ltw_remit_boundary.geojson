import json
from pathlib import Path

# Install once:
#   python -m pip install shapely alphashape
from shapely.geometry import shape, mapping
from shapely.ops import unary_union
import alphashape

ROOT = Path(__file__).resolve().parent

STATIONS_CSV = ROOT / "correct_ltw_list.csv"       # your attached list (keep this name)
GB_LAND_PATH = ROOT / "gb_land.geojson"            # new file you just added
OUT_AREAS = ROOT / "ltw_remit_boundary.geojson"           # output used by index.html

def load_csv_points():
    """
    Reads correct_ltw_list.csv which must have:
      name, latitude, longitude, in_ltw_area
    """
    import csv

    if not STATIONS_CSV.exists():
        raise RuntimeError(f"Missing {STATIONS_CSV.name}. Put your full station list in this folder with that exact name.")

    pts = []
    skipped = 0
    with STATIONS_CSV.open("r", encoding="utf-8-sig", newline="") as f:
        reader = csv.DictReader(f)
        for row in reader:
            in_ltw = str(row.get("in_ltw_area", "")).strip().lower() == "true"
            if not in_ltw:
                continue

            lat = row.get("latitude")
            lon = row.get("longitude")
            if not lat or not lon:
                skipped += 1
                continue

            try:
                lat = float(lat)
                lon = float(lon)
            except ValueError:
                skipped += 1
                continue

            pts.append((lon, lat))  # (x,y) = (lon,lat)

    if len(pts) < 3:
        raise RuntimeError(f"Not enough LTW points to build boundary (found {len(pts)}).")

    print(f"LTW points used: {len(pts)} (skipped {skipped} missing/invalid coords)")
    return pts

def build_ltw_polygon(points):
    # Tightness of the hull; tweak if you want it tighter/looser
    alpha = 1.8
    geom = alphashape.alphashape(points, alpha)
    geom = unary_union(geom)

    # gentle smoothing
    geom = geom.buffer(0.03).buffer(-0.03)

    # simplify to keep file small
    geom = geom.simplify(0.005, preserve_topology=True)
    return geom

def load_gb_land():
    if not GB_LAND_PATH.exists():
        raise RuntimeError(f"Missing {GB_LAND_PATH.name}. Create it in this folder first.")

    geo = json.loads(GB_LAND_PATH.read_text(encoding="utf-8"))
    if geo.get("type") != "FeatureCollection" or not geo.get("features"):
        raise RuntimeError("gb_land.geojson must be a FeatureCollection with at least one feature.")

    geoms = []
    for feat in geo["features"]:
        if feat.get("geometry"):
            geoms.append(shape(feat["geometry"]))

    if not geoms:
        raise RuntimeError("gb_land.geojson contains no usable geometry.")

    return unary_union(geoms)

def main():
    ltw_pts = load_csv_points()
    ltw_geom = build_ltw_polygon(ltw_pts)
    gb_land = load_gb_land()

    # Clip LTW polygon to GB land (ensures it’s land-only)
    ltw_land = gb_land.intersection(ltw_geom)

    # Rest of GB land, excluding LTW
    gb_rest = gb_land.difference(ltw_land)

    out = {
        "type": "FeatureCollection",
        "features": [
            {
                "type": "Feature",
                "properties": {"name": "London TravelWatch area", "remit": "LTW"},
                "geometry": mapping(ltw_land),
            },
            {
                "type": "Feature",
                "properties": {"name": "Great Britain (outside LTW)", "remit": "GB_REST"},
                "geometry": mapping(gb_rest),
            },
        ],
    }

    OUT_AREAS.write_text(json.dumps(out, ensure_ascii=False), encoding="utf-8")
    print(f"Wrote: {OUT_AREAS}")

if __name__ == "__main__":
    main()