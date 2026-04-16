# Project Lessons Learned

Hard-won lessons from building STAC Items for SHYFEM UGRID Icechunk stores.

## Icechunk

- **Virtual chunk containers must be explicitly authorized**: pass `authorize_virtual_chunk_access={f"s3://{BUCKET}/": None}` to `Repository.open()` — even when the virtual chunks live in the same bucket as the store. `None` means "use env credentials".
- **Open store once**: use `xr.open_zarr(session.store, consolidated=False)`, then wrap with `xu.UgridDataset(ds)` to get xugrid. Don't call `xu.open_dataset(session.store)` — it doesn't know to use the zarr engine with `consolidated=False`.

## xstac

- **UGRID datasets have no regular X/Y axis**: pass `x_dimension=False, y_dimension=False` to skip CF axis auto-detection, which raises `KeyError` on `ds.cf['X']` for unstructured grids.
- **Forecast-encoded time (step + reference time) breaks xstac**: if the temporal dim is `timedelta64` (e.g. `step`), xstac raises `TypeError: dtype timedelta64[ns] cannot be converted to datetime64[ns]`. Pass `temporal_dimension=False` and set `start_datetime`/`end_datetime` manually.
- **Empty `cube:dimensions` breaks Arrow/Parquet**: when all dimensions are skipped, xstac leaves `cube:dimensions: {}`. Arrow cannot write empty structs. Replace it manually with a populated dict.
- **`cube:dimensions` node type**: unstructured UGRID node index must use `type: "other"` — the datacube v2.2.0 schema forbids `type: "spatial"` without an `axis` field (reserved for horizontal/vertical spatial dims).

## CF Forecast Time Encoding

Both SHYFEM stores use forecast-style encoding: scalar or 1-D `time` (reference time, `datetime64`) + 1-D `step` (forecast lead time, `timedelta64`). Valid time = `time + step`. Derive temporal extent with:

```python
start = pd.Timestamp(ds["time"].values.ravel()[0])  + pd.Timedelta(ds["step"].values[0])
end   = pd.Timestamp(ds["time"].values.ravel()[-1]) + pd.Timedelta(ds["step"].values[-1])
```

Use `.ravel()[0]` to handle both scalar (0-d) and 1-d `time` arrays.

## storage Extension v2.0.0

- `storage:schemes` must go in **`item.properties`**, not `item.extra_fields`.
- Each scheme entry requires **`type`** and **`platform`** (a URI template). For custom S3-compatible (e.g. RustFS/MinIO): `type: "custom-s3"`, `platform: f"{ENDPOINT_URL}/{{bucket}}"` (double-brace so `{bucket}` stays as a literal template variable in the JSON).
- Keep `endpoint_url` as an extra field for consumers who need it for boto3/fsspec.

## rustac API

- `rustac.write` and `rustac.search` return `asyncio.Future` objects (Rust/PyO3 implementation) — `inspect.iscoroutinefunction` returns `False` but they still need `await` in async contexts.
- Use `rustac.geoparquet_writer` (async context manager) for writing, and `rustac.search_sync` for synchronous search — no `DuckdbClient` needed.
- **Always use absolute paths** — relative paths are resolved from `/`, not cwd. Use `os.path.join(os.getcwd(), "shyfem-stac/catalog.parquet")`.
- When reading back from geoparquet, assets are flat dicts with extension fields at the top level (not nested under `extra_fields`). Match assets by `"icechunk:snapshot_id" in asset` rather than media type.

## pystac Catalog Re-runs

- `catalog.save()` fails if the output directory already exists. Add `shutil.rmtree(CATALOG_DIR)` before saving.

## xugrid Geometry

- `grid.bounds` → `(xmin, ymin, xmax, ymax)` bbox
- `grid.bounding_polygon()` → Shapely Polygon of actual mesh boundary (tighter than bbox)
- Use `shapely.geometry.mapping(poly)` to convert to GeoJSON dict for pystac

## Metadata from Dataset Attributes

- `institution` → `ds.attrs["institution"]`
- `model` → parse from `ds.attrs["source"]` (format: "Model data produced by {MODEL} at {INSTITUTION}")
- `conventions` → `ds.attrs.get("Conventions", "CF-1.4")`
- Dataset `title` attrs are often internal codes (e.g. "uae", "SURF") — keep human-readable titles in config
