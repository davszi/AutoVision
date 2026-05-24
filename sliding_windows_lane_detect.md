# Sliding Windows Lane Detection — Design Notes

> **Status:** Experimental, **not working well yet**. The original prototype
> (formerly on a `luca_sliding_windows` branch) has been deleted. This doc is
> the only record — enough to reimplement from scratch if the approach is
> revived.

## Motivation

The default `LaneDetectFilter` is a Hough/line-based detector. It struggles with:

- Strongly curved lanes (single-line approximation breaks down).
- Lanes that span large pitch/perspective changes within a single frame.
- Noisy edge images where the histogram of line angles is multi-modal.

A **sliding-windows polynomial fit** (classic Udacity self-driving-car approach)
was prototyped as an alternative. The two detectors can run side-by-side in the
same pipeline for visual comparison.

## Algorithm

1. **Perspective warp** the input frame to bird's-eye view using a
   `perspective_transform` ROI polygon (4 points: TL, TR, BR, BL).
   - Transform is computed dynamically from the ROI polygon via
     `cv2.getPerspectiveTransform` — no hardcoded warp points per resolution.
   - Destination rectangle width = max(top edge, bottom edge) of polygon,
     centered horizontally on `video_width / 2`, full `video_height` tall.
2. **Histogram peak find** on the bottom half of the warped binary image —
   left peak = `argmax` of left half, right peak = `argmax` of right half.
   These are the starting x-positions for the two lane bases.
3. **Sliding windows** (default `n_windows = 9`, `window_margin = 10% of width`):
   - For each window from bottom up, collect non-zero pixels within
     `[x_current - margin, x_current + margin]`.
   - **Dynamic re-center threshold:**
     `threshold = max(min_pixels, window_area * min_pixel_percent)` where
     `min_pixel_percent = 0.10`. If pixel count > threshold, re-center
     `x_current` to the mean of collected pixels.
   - This is the "percent thresholding / resolution-aware scaling" idea — fixed
     `min_pixels` does not scale with image size.
4. **Outlier removal** on collected pixels (z-score, `z_threshold = 2.5`).
   A DBSCAN variant (`eps=30, min_samples=5`) is also implemented but not used
   by default.
5. **Polynomial fit** — third-degree `np.polyfit(y, x, 3)` per lane. Requires
   at least 3 points per side or raises.
6. **Generate fit points** along `ploty = linspace(0, H-1, H)`.
7. **Lateral offset:**
   `half_lane = (right_x_bottom - left_x_bottom) / 2`,
   `dist_to_left = video_width/2 - left_x_bottom`,
   `lateral_offset = (dist_to_left - half_lane) / half_lane`.
8. **Unwarp overlay** back to the original perspective with the inverse
   transform and `cv2.addWeighted(orig, 1, overlay, 0.3, 0)`.
9. **Draw fit lines on original frame** by transforming the fit points through
   the inverse perspective matrix (`cv2.perspectiveTransform`).

## Files that existed in the prototype

| File | Size | Role |
|---|---|---|
| `perception/filters/sliding_windows_lane_detect.py` | ~444 lines | Main filter class `SlidingWindowLaneDetectFilter(BaseFilter)`. |
| `configuration/perception_config/lane_detect_config_sliding.json` | ~69 lines | Pipeline config that runs both `lane_detection` (Hough) **and** `lane_detection_sliding_window` in parallel for comparison. |
| `tests/sliding_windows_lane_detect_old.py` | ~626 lines | Earlier reference implementation kept for diffing. |

## Integration points

1. **Register the filter** in `perception/objects/pipeline_config_types.py`:
   ```python
   from perception.filters.sliding_windows_lane_detect import SlidingWindowLaneDetectFilter

   FILTER_CLASS_LOOKUP["lane_detect_sliding_window"] = FilterClassWithExpectedParams(
       SlidingWindowLaneDetectFilter, ["visualize"]
   )
   ```
2. **Add a `perspective_transform` ROI** per resolution in
   `configuration/perception_config/roi.json`. Example for 1280x720:
   ```json
   "perspective_transform": [[107, 465], [1173, 465], [1580, 720], [-300, 720]]
   ```
   Point order must be TL, TR, BR, BL (clockwise from top-left).
3. **ROI point order convention** for the other ROIs (`lines`, `signs`, etc.)
   was also reordered to clockwise on this branch for consistency. The current
   `main` keeps the original order — verify before merging if reused.
4. **Point the pipeline at the sliding-window config**:
   ```python
   # configuration/config.py
   pipeline_config_path = 'configuration/perception_config/lane_detect_config_sliding.json'
   ```

## Class API

```python
SlidingWindowLaneDetectFilter(
    video_info: VideoInfo,
    visualize: bool,
    n_windows: int = 9,
    min_pixels: int = 50,
    min_pixel_percent: float = 0.10,
)
```

Key methods:

- `process(data: PipeData) -> PipeData` — pipeline entry point.
- `_init_transformation_matrices(roi_polygon)` — lazy on first frame.
- `find_lane_pixels(binary_warped)` — runs steps 2-4, returns
  `(left_x, left_y, right_x, right_y, window_boxes, out_img)`.
- `fit_polynomial(leftx, lefty, rightx, righty, min_points=3)` — static,
  raises `ValueError` if either side has fewer than `min_points`.
- `create_lane_overlay(warped_shape, left_fitx, right_fitx, ploty)` — green
  fill + colored polylines in warped space.
- `unwarp_overlay(overlay, original_img)` — warpPerspective with inverse matrix.

## Dependencies (new)

- `scipy` — `scipy.stats.zscore` for outlier removal.
- `scikit-learn` — `sklearn.cluster.DBSCAN` for the alternative outlier path.

Add to `env_linux.yml` / Dockerfile when re-enabling.

## Known issues / why it didn't work well

- **Polynomial degree 3 overfits** on sparse pixels — first/last frames can
  produce wild fits when one side is empty. The `fit_polynomial` raise is
  caught only implicitly by the `len > 0` guard, not by a try/except.
- **Hardcoded constants:**
  - `lane_width_in_cm = 35`, `camera_width_in_cm = 41` — same magic numbers as
    the Hough filter (which was changed from 51 to 41 on this branch — verify
    before merging back to `main` which still has 51).
  - `min_pixels = 50` floor irrespective of resolution.
- **Perspective transform fragility:** dst rect = `[left_x, 0] → [right_x, H]`,
  with `left_x/right_x` derived from `max_width` of the polygon. For oddly
  shaped polygons this stretches lanes unrealistically. A fallback to identity
  on `cv2.error` silently masks bad input.
- **`process_termination` fix** in the branch (`8ef935b`) changed
  `mp_manager.terminate()` to `mp_manager.join()` in `main.py`. Required for
  this filter to shut down cleanly during testing, but may hang the main
  process if any child does not exit on its own.
- `road_markings` assignment is commented out in two places — the filter
  computes lateral offset but does not publish `RoadMarkings` downstream, so
  control/planning sees nothing.
- DBSCAN path is implemented but never called.
- Outlier removal runs **after** window assignment, so windows can still drift
  before outliers are stripped.

## Recreating

1. Reimplement `SlidingWindowLaneDetectFilter` from the algorithm description
   above as a new `BaseFilter` subclass at
   `perception/filters/sliding_windows_lane_detect.py`.
2. Create `configuration/perception_config/lane_detect_config_sliding.json` so
   the new filter runs alongside the existing Hough detector for visual
   side-by-side comparison.
3. Apply the integration edits above (filter lookup, ROI key, config path).
4. Install `scipy` and `scikit-learn`.
5. Pick a video where the Hough detector already produces reasonable lines
   (don't debug both at once). Use the parallel pipeline config to visually
   diff the two outputs frame-by-frame.

## Next steps if revived

- Replace deg-3 fit with deg-2 unless curves justify it; deg-3 with <20 points
  is numerically unstable.
- Make `min_pixels`, `n_windows`, `window_margin` come from JSON config, not
  the constructor defaults.
- Wire `RoadMarkings` output so downstream consumers actually receive lines.
- Add a confidence score (e.g. fraction of windows that found > threshold
  pixels) and fall back to the Hough detector when low.
- Calibrate `lane_width_in_cm` / `camera_width_in_cm` per camera, not as code
  constants.
