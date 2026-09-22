# Autonomous Shuttle Safety Analysis

Post-hoc **surrogate safety measure (SSM)** analysis for interactions between an autonomous shuttle and surrounding road users (vehicles and pedestrians), computed from georeferenced object trajectories extracted from roadside video.

Given a timestamp flagged as an "anomalous" or interesting event, the notebook reconstructs the local traffic scene around the shuttle at that moment, identifies every nearby vehicle and pedestrian, and computes a set of dimension-aware, heading-aware safety indicators for each shuttle–agent pair (Time-to-Collision, Post-Encroachment Time, Deceleration Rate to Avoid a Crash, Distance Headway, lane-encroachment classification, with a plain-language narrative). It also produces a bird's-eye-view 2D plot of the scene.

This code was developed as part of research carried out by **Fondazione LINKS** and **Politecnico di Torino**, and accompanies an academic publication (see [Citation]).

## What this code does

The notebook operates on a table of per-object, per-frame trajectory points (typically extracted upstream by a video-based object detection and tracking pipeline, then georeferenced to real-world planar coordinates). It:

1. **Builds trajectories.** Loads the trajectory table into a `movingpandas.TrajectoryCollection`, grouped by object class and object ID, and splits it into four groups: the shuttle, moving cars, parked cars, and pedestrians.
2. **Defines the scene geometry.** A block of named constants describes the physical footprint of the shuttle, cars, and pedestrians (length/width in meters), the road/lane geometry, and the thresholds used to flag a conflict (proximity radius, forward corridor length, critical/warning TTC, PET, and DRAC values). These are the parameters to tune for a different vehicle, road, or study design.
3. **Classifies each nearby agent relative to the shuttle**, for a chosen event timestamp and a time window (default ±5 s) around it:
   - relative position (front/rear/left/right/etc.) and forward/lateral offset in the shuttle's own reference frame;
   - heading and speed, and whether the agent is moving in the same direction, the opposite direction, or crossing the shuttle's path;
   - lane relationship (same lane, adjacent lane, oncoming lane, or a crossing conflict), using oriented-bounding-box (OBB) projections so that vehicle/pedestrian size and heading are taken into account rather than treating agents as points.
4. **Computes surrogate safety measures** for each shuttle–agent pair:
   - **TTC (Time-to-Collision)** — two formulations: a dimension-aware car-following/lane-based TTC, and a Hydén-style crossing-path TTC/PET pair for path-crossing conflicts;
   - **PET (Post-Encroachment Time)**, including a cut-in/overtaking variant;
   - **DRAC (Deceleration Rate to Avoid a Crash)**;
   - **DHW (Distance Headway)** and the deceleration the shuttle would need to apply to stop in time;
   - lateral clearance/closing-rate metrics for overtaking situations;
   - a short natural-language narrative summarizing the interaction.
     All of the above are also recomputed at the point of closest approach within the time window, not just at the exact event timestamp.
5. **Visualizes the scene.** A 2D top-down plot draws the shuttle and every relevant agent as oriented rectangles (optionally as icons) at the event time, with recent trajectory tails and an overlay showing the crossing angle between the shuttle's and an agent's path, saved as a PNG.

The output of the core function, `analyze_anomaly(...)`, is a Python dictionary (`report`) containing the shuttle's state at the event and a list of per-agent `interactions`, each holding the fields above; it is printed to the console as a readable report and can also be consumed programmatically (e.g., for plotting or further aggregation).

## Repository contents

```
notebooks/
  shuttle_anomaly_safety_analysis.ipynb   the analysis notebook, unmodified from the original research code
requirements.txt                           pip-installable dependency list
```

**No input data are included in this repository.** The notebook is shared as-is, exactly as used in the underlying research, so that the method is transparent and reproducible against a compatible dataset. Cell outputs were not cleared. See [Input data](#input-data-not-included) below for what your own dataset needs to provide.

## Input data (not included)

The notebook expects a single CSV file — one row per detected object per video frame — loaded near the top of the notebook (`pd.read_csv(...)`, one file per analyzed video/scene). No such file is distributed with this repository; you must supply your own, produced by whatever detection/tracking/georeferencing pipeline you use upstream. The columns the code actually reads are:

| Column                     | Type            | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| -------------------------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ts`                       | timestamp       | Detection timestamp for this row. Used to build the trajectory index (one point per object per frame); in the original data this is ~1 Hz (1 fps). A copy is also kept in a `TS` column.                                                                                                                                                                                                                                                                                                                                                                         |
| `obj_id`                   | int/str         | Unique ID of the tracked object (stable across frames for the same physical agent), as produced by the tracker.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `classname`                | str             | Object class label. The code expects the values `'shuttle'`, `'car'`, and `'person'`; any object labeled `'car'` with a total trajectory length under 12 m is treated as parked/stationary rather than moving.                                                                                                                                                                                                                                                                                                                                                   |
| `x_3d`, `y_3d`             | float (meters)  | Georeferenced planar (ground-plane) position of the object, in a local metric coordinate system — e.g. obtained via camera-calibration/homography from image pixel coordinates. **These must be in meters and mutually consistent** (same origin/scale) across all objects and frames; they do not need to be true geographic coordinates. The code assigns the CRS label `EPSG:3857` to the resulting GeoDataFrame purely so `geopandas`/`movingpandas` treat distances as metric — it does not reproject or otherwise require actual Web-Mercator coordinates. |
| `x_px_coord`, `y_px_coord` | float, optional | Raw pixel-space position of the object in the source video frame. Only used as a fallback for a given agent when its `x_3d`/`y_3d` are missing (`NaN`) for the whole analysis window; if present for the shuttle and the agent, distances/positions in that case are computed in pixel space instead of meters (and reported with a `px` unit and a `pixel_fallback` flag).                                                                                                                                                                                      |

If your dataset uses different column names, either rename your columns to match the above or adjust the two or three lines that read `df[...]` near the top of the notebook — the rest of the pipeline is column-name-agnostic once the `GeoDataFrame`/`TrajectoryCollection` is built.

## Configuration

All tunable parameters live together near the top of the analysis section of the notebook (`SHUTTLE_LENGTH_M`, `CAR_WIDTH_M`, `PERSON_WIDTH_M`, lane width, `PROXIMITY_FILTER_M`, `FORWARD_CORRIDOR_M`, and the TTC/PET/DRAC critical and warning thresholds for vehicles vs. pedestrians). Adjust these to match your vehicle's real dimensions, your road's lane configuration, and the conflict thresholds relevant to your study before running the notebook on a new dataset/site.

## Getting started

1. Create the environment:
   with pip:
   ```bash
   pip install -r requirements.txt
   ```
2. Open `notebooks/shuttle_anomaly_safety_analysis.ipynb` in Jupyter.
3. Point the data-loading cell at your own per-scene CSV (see [Input data](#input-data-not-included)).
4. Review and adjust the constants described in [Configuration](#configuration) for your vehicle/site.
5. Set `anomalous_event_ts` to the timestamp you want to analyze and run the notebook. Each call to `analyze_anomaly(...)` prints a full report and returns it as a dictionary; the plotting cells that follow call `plot_anomaly_scene_2d(...)` (and `add_crossing_overlay(...)`) on that report to render and save the 2D scene figure.

## Assumptions and limitations

- Agent and shuttle headings are estimated as the mean direction of travel across the analysis window (not an instantaneous, per-frame heading), so sharp within-window maneuvers are smoothed out.
- Shuttle and agents are modeled as fixed-size, axis-aligned-in-their-own-heading rectangles (oriented bounding boxes); no per-object size estimation is performed.
- The lane-encroachment logic assumes a simple one-or-two-lane road with a single configurable lane width (`SINGLE_LANE_WIDTH_M`, `LANES_PER_DIRECTION`); it is not a general-purpose lane/HD-map model.
- TTC/DRAC/DHW are only computed for interactions the code classifies as relevant conflicts (same-lane, crossing, or overtaking/adjacent-lane situations within the forward corridor); pairs outside the configured proximity filter or forward corridor are reported as filtered out, not scored.

## Citation

If you use this code, please cite the accompanying publication:
Unveiling Anomalies in Autonomous Shuttle Behavior Through Multisource Data — TRA Conference, 2026

## Author

Shadi Nikneshan — Fondazione LINKS, Torino, Italy, in collaboration with Politecnico di Torino.
