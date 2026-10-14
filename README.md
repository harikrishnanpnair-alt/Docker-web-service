# First cloud GDB QC update

This package replaces the GDB QC ArcPy engine with Python, GDAL/OGR OpenFileGDB, Shapely and pyproj. It runs on a Linux cloud server. Users upload two ZIP files through the browser and download an Excel report. No ArcGIS installation or Windows Python clone is required for QC.

## Start here — Render deployment

Create a **new QC service** for this first trial. The package is a complete standalone service, not a patch for an unseen conversion backend `main.py`.

1. Extract this ZIP. Upload the contents of `gdb_qc_cloud` to a new GitHub repository. `Dockerfile`, `main.py`, `requirements.txt`, the other Python files and `index.html` must be at the repository root. Include `.dockerignore` as well.
2. In Render, choose **New → Web Service**, connect that repository, and select **Docker** as the language/runtime. If shown, set Dockerfile Path to `./Dockerfile`. Leave any custom Docker Command blank; the Dockerfile already starts the server.
3. Set **QC_API_KEY** to a long random secret of your own in Render's environment settings. Set **QC_ALLOWED_ORIGINS** to your existing portal's origin, e.g. `https://your-portal.example`, without a page path or trailing slash. Comma-separate multiple origins. For the first trial, `*` allows any browser origin but still requires the key. Do not put the secret in GitHub or hardcode it in the HTML.
4. Set **Health Check Path** to `/health`, then deploy. The server listens on Render's `PORT`. It runs one web worker and one sequential processing queue.
5. Copy the actual HTTPS service URL shown by Render. Replace the current portal's `index.html` with the included updated file. In **GDB QC Check**, enter the new URL and the same key, then click **Check connection**. A successful check displays `GDAL` and its version. The example URL in the field is a placeholder, not a deployed service.
6. ZIP the entire input `.gdb` folder, including its internal files. Do the same for the reference schema `.gdb`. Upload them separately using **Load input classes** and **Load reference schema**. Select classes, set the short-line threshold, then click **Run QC & generate Excel**.
7. Download **QC Excel report** and optionally **QC summary JSON**. Review `Summary`, `Check_Coverage`, `Run_Details` and the linked detail sheets.

You can also open the new QC service URL directly: it serves the same portal. Its QC URL field still needs the deployed URL and key. Key and connection fields are not saved in browser storage.

Render Docker guidance: https://render.com/docs/docker
Render Web Service setup: https://render.com/docs/web-services
Health checks: https://render.com/docs/health-checks

The existing conversion/profile API remains `https://gis-processing-backend.onrender.com`. GDB Transfer retains its separate ArcGIS service settings. Transfer has not been rewritten in this update.

## QC checks in this version

| Check | Rule |
|---|---|
| Nulls and blanks | Separate counts per field and totals; numeric zero is a value, not a null. System/binary/GlobalID fields are excluded. |
| Short lines | Total feature XY geodesic length in metres on the input CRS ellipsoid, below the selected threshold. Default 2 m. This is not a test of each vertex-to-vertex segment. |
| Exact duplicates | Normalized WKB, retaining XY/Z/M where supported; line direction and part order normalized. Each feature after the first matching geometry counts as one extra. Attributes are not part of this comparison. |
| Empty geometry | Missing or empty shape. |
| Schema | Required reference fields, types, string lengths, nullable flags, domain assignments, geometry/Z/M and CRS. Extra input fields are allowed. |
| Domains | Values checked against reference coded-value and numeric range domains; descriptions are flagged when a code is expected. Nulls are handled in the null check. |
| Subtypes | Parses exposed layer XML for subtype field/default/code and per-subtype domain assignments. Native ArcGIS-created subtype GDBs still need comparison testing. |

Feature classes inside feature datasets are supported. Schema matching uses the full dataset/class path, then a unique class name. Both uploaded GDBs are opened read-only. Private copies are made for each QC job.

Workbook styling follows the earlier blue-header QC report. Known issue indicator severity is Critical at 15+, Warning at 5–14, Minor at 1–4. These are issue indicators, not counts of distinct affected features. Failed or unsupported checks produce **INCOMPLETE** and `NOT CHECKED`, with reasons in `Check_Coverage`. A clean report only means that the implemented checks found no issues.

## Differences and limits

- Exact duplicate results can differ from ArcGIS `FindIdentical`: ArcGIS XY/Z snapping tolerance is not applied. Curves are not silently linearized; affected short-line and duplicate checks are incomplete.
- Unknown or unsupported CRS prevents a valid short-line check. Non-numeric range and glob domains are not implemented. Metadata failures are reported.
- Topology rules, connectivity/networks, relationship and attachment integrity, nonspatial tables, raster layers, and project-specific rules not represented by the reference schema are outside this first version.
- The Docker image uses Ubuntu 24.04 GDAL (tested here with 3.8.4). 64-bit OBJECTID geodatabases need a newer, separately validated GDAL build; this package is for ordinary 32-bit OBJECTID GDBs. GDAL also documents restrictions on SDC/CDF compressed data. See https://gdal.org/en/stable/drivers/vector/openfilegdb.html.
- Pilot defaults: 128 MB per ZIP, 1 GB extracted per GDB, 250,000 selected features, 50,000 workbook detail rows, 10 admitted jobs, one processing job at a time and 30-minute processing timeout. Full summary counts are retained when detail rows are capped; omitted-row counts appear in `Check_Coverage`.
- Private GDB copies can use substantial disk space. Start with small files, measure memory/disk use and choose cloud resources based on your own data. The Docker image has not been built or deployed to Render from this workspace.
- Jobs/uploads/results use local server disk. Completed job folders expire after 24 hours, with cleanup triggered on a new submission. Restarted in-flight jobs become failed; on ephemeral hosting, redeployment may also remove uploads/results. Download promptly.
- This is a shared-key trial for a trusted team: everyone with the key has the same service access. It has no per-user accounts, ownership or billing limits. It is not a validated 100-user deployment. Multiple web workers/instances need an external queue, job store and per-user access controls; do not increase `--workers` on this implementation.
- Synthetic real GDB tests passed. Your actual GDB files have not been provided or validated; compare initial cloud results against your existing QC tool, especially subtype/domain rules, before adopting it for project sign-off.

## Files

- `index.html`: complete updated menu/portal.
- `main.py`: FastAPI app, CORS, request-size limit and public readiness check.
- `cloud_gdb_router.py`: authenticated inspection/QC queue, bounded ZIP extraction, status and downloads.
- `cloud_gdb_worker.py`: isolated processing process.
- `cloud_gdb_common.py`: read-only GDB inventory and layer XML metadata.
- `cloud_qc.py`: QC calculations and Excel/JSON result data.
- `report_style.py`: Excel formatting.
- `Dockerfile`, `.dockerignore`, `requirements.txt`: cloud environment.

Do not copy only `main.py`: it imports the accompanying modules. Do not replace the conversion backend with this app unless you separately integrate its original routes. Upload this complete package to the new QC repository instead.

## Optional environment settings

| Variable | Default |
|---|---|
| QC_API_KEY | Required; service refuses startup without it |
| QC_ALLOWED_ORIGINS | `*` |
| QC_MAX_UPLOAD_MB | `128` |
| QC_MAX_UNPACKED_MB | `1024` |
| QC_MAX_TOTAL_FEATURES | `250000` |
| QC_MAX_DETAIL_ROWS | `50000` |
| QC_MAX_QUEUED_JOBS | `10` |
| QC_JOB_TIMEOUT_SECONDS | `1800` |
| QC_JOB_ROOT | `/app/qc_jobs` in Docker |

## Optional Docker test on your computer

Only if Docker is already available; a local installation is not needed for the Render method above:

```bash
docker build -t gdb-qc-cloud .
docker run --rm -p 8000:8000 -e QC_API_KEY=YOUR_OWN_SECRET -e QC_ALLOWED_ORIGINS=http://localhost:8000 gdb-qc-cloud
```

Open `http://localhost:8000`, then enter that same address in Cloud QC service URL and your secret in the key field.
