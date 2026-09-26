<p align="center">
  <img src="americamap-logo.svg" width="72" height="72" alt="AmericaMap logo" />
</p>

<h1 align="center">AmericaMap</h1>

<p align="center">
  A live operations map for the United States and Canada.<br/>
  <strong>All 50 states, Washington, D.C., 10 Canadian provinces, and 3 Canadian territories are supported.</strong>
</p>

## Current coverage

AmericaMap preserves FloridaMap's original behavior and adds jurisdiction-specific adapters without redesigning the map. The original contiguous-U.S. rollout now includes Alaska, Hawaii, every Canadian province, and every Canadian territory. Official Census and Statistics Canada boundaries drive viewport loading and clipping; jurisdiction-specific 511 systems supply cameras, road events, construction, signs, and road weather where those agencies publish a reusable public feed.

Shared nationwide and continent-wide layers remain available alongside those local feeds: traffic flow, radar, aircraft, DeFlock/OpenStreetMap plate readers, ASOS temperatures, wildfires, power outages, and public emergency sources. Emergency CAD availability varies by agency; AmericaMap labels the source on each incident and only adds verified public feeds. NOAA/NWS covers the United States, while Environment and Climate Change Canada's official GeoMet API supplies active Canadian weather alerts.

At zoom 15 and above, a 3D button lets the user opt into a dark, pitched street view. OpenStreetMap building heights are rendered through MapLibre/OpenFreeMap, while the existing live traffic-flow tiles remain visible and drive the color and relative pace of animated car sprites. The always-visible compass rotates either the 2D or 3D map and resets north when clicked; the selected bearing persists when 3D is toggled. The cars visualize traffic conditions; they are not individually tracked vehicles.

The production domain is **[americamap.app](https://americamap.app/)**. DNS and hosting can be connected when the deployment target is selected.

## Jurisdiction-by-jurisdiction architecture

Jurisdiction-specific values are deliberately isolated near the top of both runtime files:

- `app.js` contains the client `REGIONS` registry: bounds, traffic origin, and available incident feeds.
- `proxy.py` contains the server `REGIONS` registry plus each traffic adapter, ASOS network, stream allowlist, and aircraft search areas.
- `state-boundary.json` contains the simplified boundaries for every supported state, province, and territory.
- `country-boundary.json` contains the U.S. display boundary (the lower 48 and D.C. dissolved from Census state polygons, including U.S. Great Lakes waters), Alaska/Hawaii exteriors, and the existing Canada exterior. Rebuild the U.S. portion with `python3 scripts/build_us_presentation_boundary.py` when changing the presentation mask; the script runs only during development, not on the server.

Adding another jurisdiction means adding its adapter and verified sources instead of cloning the whole application again.

## Shared server cache

Browsers request same-origin AmericaMap endpoints; they do not call source agencies directly. The Python proxy keeps one shared response per jurisdiction and layer, collapses concurrent cache misses into one upstream request, and serves stale data immediately while a single background refresh runs. Normalized 511 layers and tooltips are stored in SQLite so a VM restart can continue serving the last good responses without a continent-wide cold start. High-churn traffic tiles, radar tiles, and camera snapshots use a bounded memory cache instead of filling the disk.

The default persistent database is `.cache/shared-responses.sqlite3`. On a VM, set `AMERICAMAP_CACHE_DIR=/var/cache/americamap` and make that directory writable by the service account. Keep it on a persistent volume if deployments replace the application directory. Cache statistics, byte counts, refreshes, waits, and errors are available at `/healthz`; cached responses also include an `X-AmericaMap-Cache` header such as `HIT`, `MISS`, or `STALE`.

The server also protects a small VM from overload. Request threads, upstream API calls, media fetches, source-refresh workers, and simultaneous video relays all have independent ceilings. Idle keep-alive clients are disconnected after 20 seconds, compressed responses are reused instead of recompressed for every visitor, static files stay in memory, and large HLS segments are relayed in 64 KiB chunks without caching the video. If the request ceiling is reached, AmericaMap returns `503 Service Unavailable` with `Retry-After: 2` rather than allowing memory use to grow without limit.

## Stack

| Layer | Tech |
|---|---|
| Frontend | Vanilla JavaScript + Leaflet 1.9.4, with MapLibre GL JS 5.24.0 for zoom-15+ 3D |
| Video | hls.js 1.6.16 |
| Backend | Python 3 standard-library HTTP server |
| Crypto | `cryptography` (PulsePoint public-web payload support) |

No build step, npm, or framework is required.

## Run locally

```bash
python3 -m pip install cryptography
cp .env.example .env
bash start.sh
```

Open <http://localhost:8765>.

Optional environment variables:

| Variable | Default | Description |
|---|---|---|
| `AMERICAMAP_HOST` | `127.0.0.1` | Bind address |
| `AMERICAMAP_PORT` | `8765` | Bind port |
| `AMERICAMAP_DEBUG` | `0` | Set to `1` for verbose errors |
| `AMERICAMAP_ALLOWED_HOSTS` | local host names | Comma-separated HTTP Host allowlist |
| `AMERICAMAP_TRUSTED_PROXIES` | `127.0.0.1,::1` | Proxy IPs/CIDRs allowed to supply `X-Forwarded-For` |
| `AMERICAMAP_MAX_REQUESTS` | `128` | Maximum simultaneous HTTP requests |
| `AMERICAMAP_REQUEST_QUEUE` | `256` | Kernel listen backlog requested by the server |
| `AMERICAMAP_IDLE_TIMEOUT` | `20` | Idle keep-alive timeout in seconds |
| `AMERICAMAP_MAX_STREAMS` | `48` | Maximum simultaneous uncached video relays |
| `AMERICAMAP_API_UPSTREAM_CONCURRENCY` | `16` | Maximum cache-refresh calls to state/API sources |
| `AMERICAMAP_MEDIA_UPSTREAM_CONCURRENCY` | `32` | Maximum upstream tile and snapshot calls |
| `AMERICAMAP_SOURCE_FETCH_WORKERS` | `16` | Maximum workers inside nationwide aggregate refreshes |
| `AMERICAMAP_CACHE_ENABLED` | `1` | Enable the shared response cache |
| `AMERICAMAP_CACHE_DIR` | `.cache` | Persistent SQLite cache directory |
| `AMERICAMAP_CACHE_MAX_ENTRIES` | `4096` | Maximum persistent normalized responses |
| `AMERICAMAP_CACHE_MAX_BYTES` | `268435456` | Memory and disk budget for normalized responses |
| `AMERICAMAP_MEDIA_CACHE_MAX_ENTRIES` | `2048` | Maximum in-memory tile and snapshot entries |
| `AMERICAMAP_MEDIA_CACHE_MAX_BYTES` | `100663296` | In-memory tile and snapshot budget |
| `AMERICAMAP_RADAR_TILE_CACHE_TTL` | `120` | Fresh lifetime for shared NOAA radar tiles |
| `AMERICAMAP_RADAR_TILE_STALE_TTL` | `600` | Maximum stale lifetime while radar refreshes |
| `AMERICAMAP_CACHE_REFRESH_WORKERS` | `12` | Maximum concurrent background cache refreshes |
| `AMERICAMAP_GZIP_CACHE_MAX_ENTRIES` | `256` | Maximum reusable compressed response variants |
| `AMERICAMAP_GZIP_CACHE_MAX_BYTES` | `67108864` | In-memory compressed-response budget |
| `AMERICAMAP_TDOT_API_KEY` | unset | Tennessee SmartWay feed key; set in the server environment or ignored `.env` |
| `AMERICAMAP_GOKY_API_KEY` | unset | Kentucky GoKY feed key; set in the server environment or ignored `.env` |

For a 1-vCPU / 1-GB VM, start conservatively and raise the limits only after checking `/healthz` and system memory:

```env
AMERICAMAP_MAX_REQUESTS=64
AMERICAMAP_MAX_STREAMS=12
AMERICAMAP_API_UPSTREAM_CONCURRENCY=8
AMERICAMAP_MEDIA_UPSTREAM_CONCURRENCY=12
AMERICAMAP_SOURCE_FETCH_WORKERS=8
AMERICAMAP_CACHE_MAX_BYTES=134217728
AMERICAMAP_MEDIA_CACHE_MAX_BYTES=50331648
AMERICAMAP_GZIP_CACHE_MAX_BYTES=33554432
```

## Repository lineage

AmericaMap preserves the full history of [codebooker/floridamap](https://github.com/codebooker/floridamap). The new repository is [codebooker/AmericaMap](https://github.com/codebooker/AmericaMap).

## Data sources

| Data | Source |
|---|---|
| Florida traffic cameras, incidents, construction, signs | [FL511](https://fl511.com/) |
| Georgia traffic cameras, incidents, construction, signs | [511GA](https://511ga.org/) |
| Alabama traffic cameras, incidents, roadwork, signs | [ALDOT ALGO Traffic](https://algotraffic.com/) |
| Mississippi traffic cameras, incidents, roadwork, signs, road weather | [Mississippi DOT Traffic](https://www.mdottraffic.com/default.aspx?fullsite=1) |
| South Carolina traffic cameras, incidents, roadwork, signs | [South Carolina 511](https://www.511sc.org/) |
| North Carolina traffic cameras, incidents, roadwork, signs | [NCDOT DriveNC](https://drivenc.gov/) |
| Tennessee traffic cameras, incidents, severe impacts, roadwork, signs | [TDOT SmartWay](https://smartway.tn.gov/) |
| Kentucky traffic cameras, incidents, roadwork, signs | [KYTC GoKY](https://goky.ky.gov/) |
| Virginia traffic cameras, incidents, roadwork, signs | [VDOT 511 Virginia](https://511.vdot.virginia.gov/) |
| West Virginia traffic cameras, incidents, roadwork, signs | [WV511](https://www.wv511.org/) |
| Maryland traffic cameras, incidents, road closures, signs | [Maryland CHART / Maryland 511](https://chart.maryland.gov/DataFeeds/GetDataFeeds) |
| Washington, D.C. traffic-camera snapshots and current road-work permits | [MATOC TrafficView](https://www.trafficview.org/live_traffic/) and [DDOT open data feeds](https://maps2.dcgis.dc.gov/dcgis/rest/services/FEEDS/DDOT/FeatureServer) |
| Delaware live cameras, advisories, road restrictions, signs, and road weather | [DelDOT Transportation Management Center](https://tmc.deldot.gov/datamap/) |
| Pennsylvania traffic-camera snapshots, incidents, closures, roadwork, signs, and road weather | [PennDOT 511PA](https://www.511pa.com/) |
| New Jersey live traffic cameras, delays, incidents, closures, and roadwork | [New Jersey Turnpike Authority](https://www.njta.gov/travel-resources/camera-list/) |
| Connecticut traffic-camera snapshots, incidents, closures, roadwork, transit disruptions, projects, and signs | [CTroads](https://www.ctroads.org/map) |
| Rhode Island live traffic cameras, snapshots, incidents, construction advisories, and signs | [Rhode Island DOT](https://www.dot.ri.gov/travel/) |
| Massachusetts live traffic cameras, snapshots, incidents, construction, and signs | [Mass511](https://www.mass511.com/) and [MassDOT](https://www.mass.gov/traffic-and-travel-information) |
| New Hampshire traffic-camera snapshots, incidents, closures, roadwork, signs, and road weather | [New England 511](https://www.newengland511.org/) |
| Vermont traffic-camera snapshots, incidents, closures, roadwork, signs, and road weather | [New England 511](https://www.newengland511.org/region/Vermont) and [VTrans](https://vtrans.vermont.gov/) |
| Maine traffic-camera snapshots, incidents, closures, roadwork, signs, and road weather | [New England 511](https://www.newengland511.org/region/Maine) and [MaineDOT](https://www.maine.gov/dot/publications/traffic-engineering/traffic-cameras) |
| New York live traffic cameras, snapshots, incidents, closures, roadwork, and signs | [511NY](https://www.511ny.org/) |
| Ohio traffic-camera snapshots, incidents, construction, signs, and road weather | [Ohio DOT OHGO](https://www.ohgo.com/) |
| Indiana live traffic cameras, snapshots, incidents, construction, signs, and road weather | [INDOT TrafficWise](https://511in.org/) |
| Illinois traffic-camera snapshots, incidents, construction, signs, and road weather | [Illinois DOT Travel and Maps](https://idot.illinois.gov/travel-and-maps.html) |
| Wisconsin live traffic cameras, snapshots, incidents, construction, and signs | [511 Wisconsin](https://511wi.gov/) and [WisDOT traveler information](https://wisconsindot.gov/Pages/travel/511/511.aspx) |
| Minnesota live traffic cameras, snapshots, incidents, construction, signs, and road weather | [Minnesota 511](https://511mn.org/) and [MnDOT traveler information](https://www.dot.state.mn.us/information/traffic.html) |
| Iowa live traffic cameras, snapshots, incidents, construction, signs, and road weather | [Iowa DOT 511 data feeds](https://iowadot.gov/travel-tools/iowa-511/511-data-feeds), [Iowa 511](https://511ia.org/), and [Iowa DOT open data](https://data.iowadot.gov/) |
| Missouri live traffic cameras, snapshots, incidents, flooding, construction, and signs | [MoDOT Traveler Information Map](https://traveler.modot.org/) and the [official MoDOT traveler-information service](https://mapping.modot.org/arcgis/rest/services/TravelerInformation/TravelerInformationData/MapServer) |
| Arkansas traveler information | [IDrive Arkansas](https://www.idrivearkansas.com/) is linked as the official traveler-information source, but its data and camera media are not republished because of [ARDOT's acceptable-use policy](https://site.idrivearkansas.com/index.php/policies/acceptable-use) |
| Louisiana live traffic cameras, incidents, construction, and signs | [Louisiana 511](https://www.511la.org/) and [Louisiana DOTD traveler information](https://dotd.la.gov/about/office-of-operations/intelligent-transportation-systems/its-systems-integration/511-advanced-traveler-information-system/) |
| Oklahoma live traffic cameras, signs, Waze-backed traffic reports, construction, and road weather | [Oklahoma DOT OKTraffic](https://oktraffic.org/) and [ODOT current traffic conditions](https://oklahoma.gov/odot/travel/traffic/current-traffic-conditions.html) |
| Texas live traffic cameras, incidents, closures, flooding, road damage, and construction | [TxDOT DriveTexas](https://drivetexas.org/) and [TxDOT traveler information](https://www.txdot.gov/discover/texas-state-travel-guide-map.html) |
| New Mexico traffic-camera snapshots, incidents, closures, construction, signs, and road weather | [NMDOT NMRoads](https://nmroads.com/) and [NMDOT travel-information maps](https://www.dot.nm.gov/travel-information/maps/) |
| Arizona traffic-camera snapshots, incidents, roadwork, signs, and road weather | [ADOT AZ511](https://az511.gov/) and [ADOT traveler information](https://azdot.gov/travel-and-commute/traffic/travel-information) |
| California live traffic cameras, snapshots, incidents, closures, construction, signs, and traffic flow | [Caltrans QuickMap](https://quickmap.dot.ca.gov/) and [Caltrans travel information](https://dot.ca.gov/cttravel) |
| Nevada live traffic cameras, snapshots, incidents, closures, construction, signs, and road weather | [Nevada 511](https://www.nvroads.com/) and [Nevada DOT traffic cameras](https://www.dot.nv.gov/travel-info/road-conditions/traffic-cameras) |
| Oregon live traffic-camera snapshots, incidents, construction, and road weather | [Oregon DOT TripCheck](https://www.tripcheck.com/) and [ODOT Traveler Information](https://www.oregon.gov/ODOT/Maintenance/Pages/Traveler-Information.aspx) |
| Washington live traffic-camera snapshots, road alerts, construction, and road weather | [WSDOT real-time travel data](https://wsdot.com/travel/real-time/traffic-map), the [WSDOT camera/weather feature service](https://data.wsdot.wa.gov/arcgis/rest/services/TravelInformation/TravelInfoCamerasWeather/FeatureServer), and the [WSDOT road-alert feature service](https://data.wsdot.wa.gov/arcgis/rest/services/TravelInformation/TravelInfoRoadAlerts/FeatureServer) |
| Idaho live traffic cameras, snapshots, incidents, construction, signs, and road weather | [Idaho 511](https://511.idaho.gov/) and [Idaho Transportation Department travel information](https://itd.idaho.gov/travel/) |
| Utah live traffic cameras, snapshots, incidents, construction, signs, and road weather | [UDOT Traffic](https://udottraffic.utah.gov/) and [UDOT traveler information](https://connect.udot.utah.gov/current-conditions/) |
| Colorado live traffic cameras, snapshots, video, incidents, construction, signs, and road weather | [COtrip](https://www.cotrip.org/) and [Colorado DOT traveler information](https://www.codot.gov/travel) |
| Michigan traffic-camera snapshots, incidents, construction, and signs | [MDOT Mi Drive](https://mdotjboss.state.mi.us/MiDrive/map) and [Michigan DOT traveler information](https://www.michigan.gov/drive) |
| Wyoming traffic-camera snapshots, incidents, construction, signs, and road weather | [WYDOT 511 Map](https://map.wyoroad.info/511-map/) and [WYDOT traveler information](https://www.dot.state.wy.us/home/news_info/road--travel-information.html) |
| Montana traffic-camera snapshots, incidents, construction, signs, and road weather | [Montana 511](https://www.511mt.net/) and [Montana DOT traveler information](https://www.mdt.mt.gov/travinfo/) |
| North Dakota traffic-camera snapshots, alerts, construction, and road weather | [ND Roads](https://travel.dot.nd.gov/), [NDDOT road/weather resources](https://www.dot.nd.gov/travel-and-safety/road-conditions-weather-resources), and the [NDDOT web-map service catalog](https://www.dot.nd.gov/construction-and-planning/planning-process/gis-and-mapping/web-map-services) |
| South Dakota traffic-camera snapshots, advisories, restrictions, construction, and road weather | [SD511](https://www.sd511.org/) and [South Dakota DOT traveler information](https://dot.sd.gov/travelers/travelers/road-conditions) |
| Nebraska traffic-camera snapshots, incidents, construction, signs, and road weather | [Nebraska 511](https://www.511.nebraska.gov/) and [Nebraska DOT traveler information](https://dot.nebraska.gov/travel/) |
| Kansas traffic-camera snapshots, incidents, construction, signs, and road weather | [KanDrive](https://www.kandrive.gov/) and [Kansas DOT KanDrive information](https://www.ksdot.gov/travel/travel-conditions/kandrive) |
| Alaska traffic-camera snapshots, incidents, construction, signs, and road weather | [Alaska 511](https://511.alaska.gov/) and its [developer documentation](https://511.alaska.gov/developers/doc) |
| Hawaii cameras and active lane closures | [GoAkamai cameras](https://goakamai.org/cameras/) and [Hawaii DOT roadwork](https://hidot.hawaii.gov/highways/roadwork/); camera metadata and snapshots are cached by the AmericaMap server |
| Newfoundland and Labrador traffic-camera snapshots, incidents, and construction | [NL 511](https://511nl.ca/) |
| Nova Scotia traffic-camera snapshots, incidents, and construction | [Nova Scotia 511](https://511.novascotia.ca/) |
| New Brunswick traffic-camera snapshots, incidents, and construction | [New Brunswick 511](https://511.gnb.ca/) |
| Quebec camera locations and real-time road construction | [Données Québec traffic cameras](https://www.donneesquebec.ca/recherche/dataset/camera-de-circulation), [Données Québec roadwork](https://www.donneesquebec.ca/recherche/dataset/travaux-routiers), and [Québec 511](https://www.quebec511.info/) |
| Ontario traffic-camera snapshots, incidents, and construction | [Ontario 511](https://511on.ca/) and its [developer documentation](https://511on.ca/developers/doc) |
| Manitoba traffic-camera snapshots, incidents, and construction | [Manitoba 511](https://www.manitoba511.ca/) |
| Saskatchewan traffic-camera snapshots, incidents, and construction | [Saskatchewan Highway Hotline](https://hotline.gov.sk.ca/) |
| Alberta traffic-camera snapshots, incidents, construction, signs, and road weather | [Alberta 511](https://511.alberta.ca/) and its [developer documentation](https://511.alberta.ca/developers/doc) |
| British Columbia traffic-camera snapshots, incidents, and construction | [DriveBC](https://www.drivebc.ca/) and the official [DriveBC Open511 API](https://api.open511.gov.bc.ca/help) |
| Yukon traffic-camera snapshots, incidents, construction, signs, and road weather | [Yukon 511](https://511yukon.ca/) |
| Northwest Territories traffic-camera snapshots, road conditions, and advisories | [DriveNWT](https://drivenwt.ca/) and the [Government of Northwest Territories highway conditions service](https://www.inf.gov.nt.ca/en/services/highway-conditions-map-drivenwt) |
| Prince Edward Island | [Prince Edward Island 511](https://511.gov.pe.ca/map) cameras and roadwork |
| Nunavut | Boundaries, traffic flow, aircraft, radar where available, and ASOS weather are active; the jurisdictional road/camera feed remains empty-safe until a reusable official public source is available |
| Canadian weather alerts | [Environment and Climate Change Canada GeoMet weather alerts](https://api.weather.gc.ca/collections/weather-alerts?f=html) |
| Traffic flow | State 511 / Iteris tiles |
| Weather stations | [Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/) |
| Road sensors | FDOT, [GDOT RWIS via NOAA/NWS](https://www.weather.gov/ffc/gdot_rwis), [Mississippi DOT Traffic](https://www.mdottraffic.com/default.aspx?fullsite=1), DelDOT, PennDOT 511PA, [New England 511](https://www.newengland511.org/), [Ohio DOT OHGO](https://www.ohgo.com/), [INDOT TrafficWise](https://511in.org/), [Illinois DOT](https://idot.illinois.gov/travel-and-maps.html), [Wisconsin RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=WI_RWIS), [Minnesota 511 RWIS](https://511mn.org/), [Iowa RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=IA_RWIS), [Oklahoma DOT OKTraffic RWIS](https://oktraffic.org/), [NMDOT NMRoads RWIS](https://nmroads.com/), [Arizona DOT RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=AZ_RWIS), [California DOT RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=CA_RWIS), [Nevada DOT RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=NV_RWIS), [Oregon DOT TripCheck RWIS](https://www.tripcheck.com/), [WSDOT road weather](https://data.wsdot.wa.gov/arcgis/rest/services/TravelInformation/TravelInfoCamerasWeather/FeatureServer/1), [Idaho RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=ID_RWIS), [Utah RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=UT_RWIS), [Colorado DOT COtrip RWIS](https://www.cotrip.org/), [Wyoming RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=WY_RWIS), [Montana RWIS via Iowa Environmental Mesonet](https://mesonet.agron.iastate.edu/sites/networks.php?network=MT_RWIS), [ND Roads environmental sensor sites](https://www.dot.nd.gov/travel-and-safety/road-conditions-weather-resources), [SD511 road weather](https://www.sd511.org/), [Nebraska RWIS](https://mesonet.agron.iastate.edu/sites/networks.php?network=NE_RWIS), [Kansas RWIS](https://mesonet.agron.iastate.edu/sites/networks.php?network=KS_RWIS), [Arkansas RWIS](https://mesonet.agron.iastate.edu/sites/networks.php?network=AR_RWIS), [Connecticut RWIS](https://mesonet.agron.iastate.edu/sites/networks.php?network=CT_RWIS), [Massachusetts RWIS](https://mesonet.agron.iastate.edu/sites/networks.php?network=MA_RWIS), [Maryland RWIS](https://mesonet.agron.iastate.edu/sites/networks.php?network=MD_RWIS), [New York RWIS](https://mesonet.agron.iastate.edu/sites/networks.php?network=NY_RWIS), and [South Carolina RWIS](https://mesonet.agron.iastate.edu/sites/networks.php?network=SC_RWIS) |
| Plate readers | [DeFlock](https://deflock.me/) and OpenStreetMap contributors |
| Fire / EMS dispatch | [PulsePoint](https://www.pulsepoint.org/) public web feed, with its participating-agency directory refreshed daily for every supported state and D.C. |
| Georgia wildfires | [Georgia Forestry Commission Public Viewer](https://georgiafc.firesponse.com/public/) |
| Alabama wildfires | [Alabama Forestry Commission](https://forestry.alabama.gov/Pages/Maps/Wildfires.aspx) |
| Mississippi wildfires | [NIFC Wildland Fire Interagency Geospatial Services](https://www.nifc.gov/fire-information/maps) |
| South Carolina wildfires | [South Carolina Forestry Commission Public Viewer](https://scfc.firesponse.com/public/) |
| North Carolina wildfires | [North Carolina Forest Service Public Viewer](https://ncfspublic.firesponse.com/) |
| Tennessee wildfires | [Tennessee Division of Forestry](https://www.tn.gov/tnwildlandfire/suppression/current-wildfires.html) |
| Kentucky wildfires | [Kentucky Division of Forestry](https://eec.ky.gov/Natural-Resources/Forestry/Pages/default.aspx) |
| Virginia wildfires | [Virginia Department of Forestry](https://www.dof.virginia.gov/wildland-prescribed-fire/wildfire-suppression/) and [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| West Virginia wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Maryland wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Washington, D.C. wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Delaware wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Pennsylvania wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| New Jersey wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Connecticut wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Rhode Island wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Massachusetts wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| New Hampshire wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Vermont wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Maine wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| New York wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Ohio wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Indiana wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Illinois wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Wisconsin wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Minnesota wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Iowa wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Missouri wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Arkansas wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Louisiana wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Oklahoma wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Texas wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| New Mexico wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Arizona wildfires | [Arizona Interagency Wildfire Prevention](https://wildlandfire.az.gov/wildfire-situation) and [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| California wildfires | [CAL FIRE active incidents](https://www.fire.ca.gov/incidents) and [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Nevada wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Oregon wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Washington wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Idaho wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Utah wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Colorado wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Michigan wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Wyoming wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Montana wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| North Dakota wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| South Dakota wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Nebraska wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Kansas wildfires | [NIFC WFIGS](https://www.nifc.gov/fire-information/maps) |
| Phoenix Fire active incidents | [City of Phoenix Fire incident data](https://www.phoenix.gov/administration/departments/fire/data/incident-data.html) |
| Washington, D.C. public-safety alerts | [D.C. HSEMA AlertDC](https://trainingtrack.hsema.dc.gov/NRss/RssFeed) |
| New York City public-safety alerts | [NYC Emergency Management / Notify NYC](https://a858-nycnotify.nyc.gov/notifynyc/) |
| Weather alerts | [NOAA/National Weather Service](https://www.weather.gov/) |
| Florida outages | [Duke Energy public outage data](https://services3.arcgis.com/oX5r75R7mapdoI2F/ArcGIS/rest/services/Duke_Energy_Distribution_Outages_Public/FeatureServer), [Tampa Electric](https://www.tampaelectric.com/poweroutages/), [Keys Energy](https://powerstatus.keysenergy.com/), JEA, Lakeland Electric, OUC, and SECO Energy |
| Georgia outages | [Georgia Power Outage Map](https://outagemap.georgiapower.com/) |
| Alabama outages | [Alabama Power Outage Map](https://outagemap.alabamapower.com/) |
| Mississippi outages | [Mississippi Power](https://outagemap.mississippipower.com/) and [Entergy Mississippi](https://www.etrviewoutage.com/map?state=MS) |
| South Carolina outages | [Dominion Energy South Carolina](https://outagemap.dominionenergysc.com/) |
| North Carolina outages | [NC Department of Public Safety](https://www.ncdps.gov/power-outages) and [Duke Energy public outage data](https://services3.arcgis.com/oX5r75R7mapdoI2F/ArcGIS/rest/services/Duke_Energy_Distribution_Outages_Public/FeatureServer) |
| Tennessee outages | [Nashville Electric Service](https://www.nespower.com/outages/) and [Knoxville Utilities Board](https://www.kub.org/outage/map) |
| Kentucky outages | [LG&E and Kentucky Utilities](https://stormcenter.lge-ku.com/) |
| Virginia outages | [Dominion Energy Virginia](https://outagemap.dominionenergy.com/external/default.html?lv=true) |
| West Virginia outages | [Appalachian Power](https://outagemap.appalachianpower.com/?c=External&o=District) |
| Maryland outages | [Maryland Emergency Management statewide outage map](https://mdgeodata.md.gov/PowerOutages/) |
| Washington, D.C. outages | [Pepco Outage Map](https://outagemap.pepco.com/) |
| Delaware outages | [Delmarva Power Outage Map](https://outagemap.delmarva.com/) |
| Pennsylvania outages | [Pennsylvania Emergency Management county power-outage status](https://services2.arcgis.com/xtuWQvb2YQnp0z3F/ArcGIS/rest/services/Pennsylvania_County_Power_Outage_Status/FeatureServer) |
| New Jersey outages | [PSE&G](https://outagecenter.pseg.com/external/default.html) and [JCP&L](https://outages-nj.firstenergycorp.com/) |
| Connecticut outages | [Eversource Connecticut](https://outagemap.eversource.com/external/default.html) |
| Rhode Island outages | [Rhode Island Energy](https://outagemap.rienergy.com/) |
| Massachusetts outages | [Eversource](https://outagemap.eversource.com/external/default.html) and [National Grid](https://outagemap.ma.nationalgridus.com/) |
| New Hampshire outages | [Eversource](https://outagemap.eversource.com/external/default.html) and [New Hampshire Electric Cooperative](https://nhec.outagemap.coop/) |
| Vermont outages | [VTOutages statewide utility data](https://vtoutages.org/) with [VTrans town polygons](https://maps.vtrans.vermont.gov/arcgis/rest/services/VTrans511/511lookup/FeatureServer/16) |
| Maine outages | [Versant Power Storm Center](https://kubra.io/stormcenter/views/05bfafbb-0ad1-4ff1-8287-d32fd1ed7fce) |
| New York outages | [Con Edison](https://outagemap.coned.com/external/default.html), [Orange & Rockland](https://outagemap.oru.com/external/default.html), [National Grid New York](https://outagemap.ny.nationalgridus.com/), [Central Hudson](https://outagemap.cenhud.com/), and [PSEG Long Island](https://outagemap.psegliny.com/) |
| Ohio outages | [AEP Ohio](https://outagemap.aepohio.com/), [FirstEnergy Ohio](https://outages-oh.firstenergycorp.com/), [AES Ohio](https://myprofile.aes-ohio.com/Outages/Outages.html), and [Duke Energy Ohio](https://outagemaps.duke-energy.com/#/current-outages/ohky) |
| Indiana outages | [AES Indiana](https://myaccount.aesindiana.com/outages/outagemap.html), [Duke Energy Indiana](https://outagemaps.duke-energy.com/#/current-outages/in), [Indiana Michigan Power](https://outagemap.indianamichiganpower.com/), and [NIPSCO](https://www.nipsco.com/outages/power-outages) |
| Illinois outages | [ComEd](https://www.comed.com/outages/experiencing-an-outage/outage-map) and [Ameren Illinois](https://outagemap.ameren.com/?c=External&o=StateZIP) |
| Wisconsin outages | [We Energies](https://www.we-energies.com/outagesummary/view/outagegrid), [Wisconsin Public Service](https://www.wisconsinpublicservice.com/outagesummary/view/outagegrid), and [Madison Gas and Electric](https://mge.smartcmobile.com/Outage/) |
| Minnesota outages | [Xcel Energy](https://www.outagemap-xcelenergy.com/outagemap/) and [Minnesota Power](https://mnpower.com/OutageCenter/OutageMap) |
| Iowa outages | [MidAmerican Energy](https://www.midamericanenergy.com/OutageWatch/dsk.html) and the [Iowa Association of Electric Cooperatives statewide outage map](https://www.iowarec.org/outages) |
| Missouri outages | [Ameren Missouri](https://outagemap.ameren.com/?c=External&o=StateZIP) and [Evergy](https://outagemap.evergy.com/) |
| Arkansas outages | [Entergy Arkansas](https://www.etrviewoutage.com/map?state=AR) and [SWEPCO](https://outagemap.swepco.com/) |
| Louisiana outages | [Entergy Louisiana](https://www.etrviewoutage.com/map?state=LA), [Cleco](https://myaccount.cleco.com/portal/#/previewoutage), and [SWEPCO](https://outagemap.swepco.com/) |
| Oklahoma outages | [Public Service Company of Oklahoma](https://outagemap.psoklahoma.com/) and [OG&E System Watch](https://www.oge.com/wps/portal/ord/outages/systemwatch/) |
| Texas outages | [Oncor](https://stormcenter.oncor.com/external/default.html), [AEP Texas](https://outagemap.aeptexas.com/), [Texas-New Mexico Power](https://outagemap.tnmp.com/), [Entergy Texas](https://www.etrviewoutage.com/map?state=TX), and [CenterPoint Energy](https://tracker.centerpointenergy.com/map/) |
| New Mexico outages | [PNM Outage Map](https://outagemap.pnm.com/) |
| Arizona outages | [Arizona Public Service Outage Map](https://outagemap.aps.com/outageviewer/) |
| California outages | [PG&E Outage Map](https://pgealerts.alerts.pge.com/outage-tools/outage-map/) |
| Nevada outages | [NV Energy Outage Map](https://www.nvenergy.com/outages-and-emergencies/view-current-outages) |
| Oregon outages | [Pacific Power Outage Map](https://www.pacificpower.net/outages-safety.html?source=APP) |
| Washington outages | [Pacific Power Outage Map](https://www.pacificpower.net/outages-safety.html?source=APP) |
| Idaho outages | [Rocky Mountain Power Outage Map](https://www.rockymountainpower.net/outages-safety.html?source=APP) |
| Utah outages | [Rocky Mountain Power Outage Map](https://www.rockymountainpower.net/outages-safety.html?source=APP) |
| Colorado outages | [Xcel Energy Colorado Outage Map](https://co.my.xcelenergy.com/s/outage-safety/outage-map) |
| Michigan outages | [Indiana Michigan Power Outage Map](https://outagemap.indianamichiganpower.com/) |
| Wyoming outages | Live [Rocky Mountain Power](https://www.rockymountainpower.net/outages-safety.html?source=APP) and [Montana-Dakota Utilities](https://customer.montana-dakota.com/outage-map) data |
| Montana outages | Live [NorthWestern Energy](https://www.northwesternenergy.com/outages/outage-map) and [Montana-Dakota Utilities](https://customer.montana-dakota.com/outage-map) data |
| North Dakota outages | Live [Otter Tail Power](https://outages.otpco.com/) and [Montana-Dakota Utilities](https://customer.montana-dakota.com/outage-map) data |
| South Dakota outages | Live [NorthWestern Energy](https://www.northwesternenergy.com/outages/outage-map), [Otter Tail Power](https://outages.otpco.com/), and [Montana-Dakota Utilities](https://customer.montana-dakota.com/outage-map) data |
| Nebraska outages | Live [Lincoln Electric System](https://www.les.com/outage-center) data and the official [Omaha Public Power District](https://www.oppd.com/outages/) and [Nebraska Public Power District](https://www.nppd.com/outages) outage centers |
| Kansas outages | [Evergy Outage Map](https://outagemap.evergy.com/) |
| Weather radar | [RainViewer](https://www.rainviewer.com/) and NOAA NWS |
| Aircraft | [adsb.lol](https://adsb.lol/) |
| State boundary | [U.S. Census Bureau cartographic boundary files](https://www.census.gov/geographies/mapping-files/time-series/geo/cartographic-boundary.html) and TIGERweb |
| Base map | [OpenFreeMap](https://openfreemap.org/) dark vector style and OpenStreetMap data |
| 3D buildings | [OpenFreeMap](https://openfreemap.org/) vector tiles and OpenStreetMap building data rendered with [MapLibre GL JS](https://maplibre.org/maplibre-gl-js/docs/) |

## License

GPLv3. See [LICENSE](LICENSE).
