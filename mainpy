from flask import Flask, request, jsonify, send_from_directory
from flask_cors import CORS
import osmnx as ox
import networkx as nx
import requests
import json
from shapely.geometry import Point
from shapely.ops import unary_union
import pickle  
from dotenv import load_dotenv
import os
import gzip
app = Flask(__name__)
CORS(app)

# 🔑 TomTom API key
load_dotenv()
TOMTOM_API_KEY = os.getenv("API_KEY")

# ✅ Serve frontend
@app.route("/")
def index():
    return send_from_directory("static", "map.html")

#Load Graph

print("Loading road network for Delhi…")

try:
    # ✅ Try loading compressed pickle file
    with gzip.open("delhi_graph.pkl.gz", "rb") as f:
        G = pickle.load(f)
    print("Graph loaded from compressed pickle (.gz) ✅")
except FileNotFoundError:
    # ⚙️ If file missing, build from GraphML and save compressed
    print("Compressed pickle not found. Loading from GraphML (this will take a while)…")
    G = ox.load_graphml("delhi.graphml")
    G = ox.add_edge_speeds(G)
    G = ox.add_edge_travel_times(G)

    # ✅ Save as compressed pickle for future use
    with gzip.open("delhi_graph.pkl.gz", "wb") as f:
        pickle.dump(G, f, protocol=pickle.HIGHEST_PROTOCOL)

    print("Graph saved as delhi_graph.pkl.gz and loaded ✅")




# ✅ Load zone data
print("Loading risk zones from zones.json …")
with open("zones.json", "r") as f:
    zones_data = json.load(f)

# Preprocess into Shapely geometry
zones = []
for z in zones_data:
    p = Point(z["lon"], z["lat"]).buffer(z["radius"] / 111320)  # convert radius meters -> degrees roughly
    zones.append({
        "geometry": p,
        "zone": z["zone"],
        "radius": z["radius"],
        "total_incidents": z["total_incidents"]
    })
print(f"Loaded {len(zones)} zones ✅")

# ✅ Helper: compute zone penalty
def get_zone_penalty(lat, lon):
    point = Point(lon, lat)
    penalty = 1.0
    for z in zones:
        if z["geometry"].intersects(point):
            if z["zone"] == "red":
                penalty = max(penalty, 4)
            elif z["zone"] == "yellow":
                penalty = max(penalty, 4)
    return penalty

# ✅ Cost functions
def distance_cost(u, v, data):
    return data.get("length", 1)

def safe_cost(u, v, data):
    lat = (G.nodes[u]['y'] + G.nodes[v]['y']) / 2
    lon = (G.nodes[u]['x'] + G.nodes[v]['x']) / 2
    base_cost = data.get("length", 1)
    penalty = get_zone_penalty(lat, lon)
    return base_cost * penalty

# ✅ TomTom geocoding suggestion endpoint
@app.route("/suggest")
def suggest():
    query = request.args.get("q")
    if not query:
        return jsonify([])

    url = f"https://api.tomtom.com/search/2/search/{query}.json"
    params = {
        "key": TOMTOM_API_KEY,
        "typeahead": "true",
        "limit": 5,
        "countrySet": "IN"
    }

    try:
        response = requests.get(url, params=params, timeout=5)
        data = response.json()
        results = []
        for item in data.get("results", []):
            if "position" in item:
                results.append({
                    "name": item["address"].get("freeformAddress", "Unknown"),
                    "lat": item["position"]["lat"],
                    "lon": item["position"]["lon"]
                })
        return jsonify(results)
    except Exception as e:
        return jsonify({"error": str(e)}), 500

# ✅ Routing endpoint
@app.route("/route")
def route():
    try:
        start = request.args.get("start")
        end = request.args.get("end")
        mode = request.args.get("mode", "fast")

        slat, slon = map(float, start.split(","))
        elat, elon = map(float, end.split(","))

        orig = ox.distance.nearest_nodes(G, slon, slat)
        dest = ox.distance.nearest_nodes(G, elon, elat)

        weight_func = safe_cost if mode == "safe" else distance_cost

        path = nx.shortest_path(G, orig, dest, weight=weight_func)

        route_gdf = ox.routing.route_to_gdf(G, path)
        coords = []
        for geom in route_gdf.geometry:
            coords += list(geom.coords)

        geojson = {
            "type": "Feature",
            "geometry": {"type": "LineString", "coordinates": coords},
            "properties": {"mode": mode}
        }
        return jsonify(geojson)
    except nx.NetworkXNoPath:
        return jsonify({"error": "No route found"}), 404
    except Exception as e:
        return jsonify({"error": str(e)}), 500

# ✅ New: Zones endpoint for frontend visualization
@app.route("/zones")
def get_zones():
    frontend_zones = []
    for z in zones_data:
        frontend_zones.append({
            "lat": z["lat"],
            "lon": z["lon"],
            "zone": z["zone"],
            "radius": z["radius"]
        })
    return jsonify(frontend_zones)

# ✅ Health check
@app.route("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
      app.run(debug=True)
