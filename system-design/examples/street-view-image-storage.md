# Taxi Street View Image Storage

## 1. Goal and scope

Build a system that accepts road imagery captured by taxis, stores it with reliable location and capture metadata, processes it for downstream use, and serves approved imagery to mapping systems and users.

This design covers:

- Camera capture and taxi-side software
- Resumable uploads over unreliable mobile networks
- Durable image and metadata storage
- Asynchronous image processing and downstream distribution
- Location-based image lookup and user-facing delivery
- Device authentication, authorization, privacy, and operations

### Initial assumptions

- Images are captured at a configurable interval or distance threshold.
- A taxi may be offline for periods and must buffer captures locally.
- Image processing is asynchronous; a short delay before imagery is searchable is acceptable.
- A sample sizing assumption is 10,000 taxis, 10,000 images per taxi per day, and 1 MB per image: about 100 million images and 100 TB of raw image data per day. This is illustrative and excludes derivatives, replication, and metadata. Capture rate, image size, and retention policy are major cost drivers.
- The initial product is a chronological/geospatial image archive. Image ranking and richer user experiences can be layered on later.

## 2. Functional and system requirements

### Functional

- Register and authenticate each taxi capture device.
- Ingest images and capture metadata.
- Resume uploads after disconnects and safely retry requests.
- Store original and processed image versions.
- Validate images, remove duplicates, blur sensitive visual details, and map-match captures.
- Notify downstream image-understanding and map-generation systems.
- Find suitable imagery near a location and heading.
- Support retention, deletion, and access control.

### System qualities

- Durable ingestion: a device removes its local copy only after server acknowledgement.
- High write throughput and elastic processing capacity.
- Eventual consistency between upload and search availability.
- Protection of image data, device credentials, and location metadata.
- Isolation between raw imagery and approved user-facing imagery.

## 3. High-level architecture

~~~text
 Taxi camera(s) + sensors
           │ capture + metadata
           ▼
 Capture software ──► In-taxi edge computer
                           ├─ encrypted local spool
                           ├─ device identity / health
                           └─ resumable uploader
                                      │
                                      │ mTLS, upload manifest
                                      ▼
                         API gateway / upload service
                           ├─ authenticate device
                           ├─ authorize upload
                           ├─ create upload session
                           └─ publish completion event
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
             Raw object storage                  Event queue / log
                                                        │
                                                        ▼
                                                Processing workers
                                        ├─ checksum / format validation
                                        ├─ duplicate and quality checks
                                        ├─ privacy blurring
                                        ├─ GPS validation / map matching
                                        ├─ thumbnails and derivatives
                                        └─ processing events
                                              │             │
                                              ▼             ▼
                                     Processed object     Metadata and
                                        storage           geo index
                                                           │
                             ┌─────────────────────────────┴────────────┐
                             ▼                                          ▼
                    Map / ML consumers                         Image query API
                                                                        │
                                                                       CDN
                                                                        │
                                                                     Clients
~~~

Object storage holds image bytes. Metadata storage and a geospatial index locate the objects. Queues separate the upload path from processing and downstream jobs.

## 4. Taxi hardware

A practical taxi installation contains:

- **Camera(s):** Road-facing, calibrated, rugged cameras. One or more cameras may be used depending on desired coverage.
- **GNSS receiver:** Location and time source. Higher-accuracy equipment can help in urban canyons.
- **IMU:** Measures motion and orientation to estimate the camera direction and support map matching.
- **In-taxi edge computer:** Rugged ARM or x86 computer that buffers captures, manages device credentials, and uploads data. It may be integrated into the camera if that device has sufficient compute and storage.
- **Local storage:** Encrypted SSD or other rugged storage sized for the expected offline period.
- **Connectivity:** Cellular modem and antennas; depot Wi-Fi can provide a bulk-upload option.
- **Power and environmental protection:** Vehicle power conditioning, safe shutdown behavior, and hardware rated for vibration and temperature.
- **Secure element / TPM:** Protects the device private key and supports secure boot where available.

## 5. Taxi software

Separate capture from uploading so a slow network cannot block the camera.

### Camera capture service

- Controls the camera through its driver or vendor SDK.
- Triggers captures by configured time interval or distance traveled.
- Encodes the image in the configured format and resolution.
- Assigns a stable capture ID.
- Synchronizes the image timestamp with GNSS and IMU readings.
- Attaches location, heading, orientation, calibration version, and checksum.
- Hands a completed capture to the edge agent.

### Edge agent

Runs as a background service on the in-taxi computer, typically embedded Linux. It starts on boot and has no passenger-facing UI.

- **Spool manager:** Writes image and metadata to durable encrypted storage. Uses temporary files and atomic completion markers so a restart cannot mistake a partial file for a complete capture.
- **Upload manager:** Requests upload sessions, sends resumable chunks, retries transient errors with backoff, and limits bandwidth when necessary.
- **Acknowledgement handler:** Marks an image uploaded only after durable server acknowledgement. Applies local cleanup policy only after acknowledgement.
- **Device security:** Protects credentials, validates server certificates, and uses encrypted storage.
- **Health monitor:** Reports camera, disk, GNSS, temperature, power, and network status.
- **Configuration and update manager:** Receives signed capture-policy updates and signed software updates.
- **Optional edge processing:** May resize or reject clearly unusable captures. Privacy blurring can happen on device, centrally, or in both places depending on policy and compute budget.

A suitable implementation is a long-running C++ or Rust service for production device software. Python can be useful for prototypes and test utilities.

### Local capture state

~~~text
CAPTURED → STORED → UPLOADING → SERVER_ACKNOWLEDGED → LOCAL_CLEANUP
~~~

The local queue must recover after process or vehicle power failure. If storage approaches its limit, alert the backend and apply an explicit capture-rate or retention policy; do not silently discard unuploaded images.

## 6. Device identity and authentication

### Recommended baseline

Give every taxi edge device a unique X.509 certificate and private key. Use mutual TLS (mTLS) for device authentication. The private key should be generated and held in a TPM or secure element when possible.

The certificate identifies a device; backend policy determines what that device is allowed to do. It should not grant broad access by itself.

### Enrollment and certificate lifecycle

1. During installation, the device generates a key pair. The private key stays on the device.
2. The device submits a certificate signing request through a controlled enrollment process.
3. The fleet certificate authority issues a device certificate and associates its device identity with the registered taxi/camera assignment.
4. The backend trusts the fleet CA and maps the certificate identity to a device registry record.
5. Rotate certificates before expiration. Disable or revoke the device identity if hardware is stolen, compromised, or decommissioned.

Avoid a shared fleet-wide secret. Do not place long-lived cloud credentials in the vehicle.

### Connection authentication

1. The taxi opens a TLS connection and validates the backend’s server certificate.
2. The backend requests a client certificate.
3. The taxi presents its device certificate and proves possession of the corresponding private key during the TLS handshake.
4. The backend validates the certificate chain, validity, device status, and authorization policy.
5. The device may request an upload session only for its authorized identity and assigned capture stream.

A copied public certificate is not sufficient to impersonate a device; the client must also prove possession of the private key.

### OAuth option

OAuth 2.0 is useful when APIs need short-lived tokens with scopes and audiences. A certificate can authenticate the taxi to the OAuth token endpoint; the endpoint then issues a short-lived token such as one scoped to image upload. Where supported, bind the token to the certificate to reduce the risk of a stolen bearer token being replayed.

For a first version, mTLS plus device-registry authorization is simpler. Add OAuth when centralized token issuance and fine-grained scopes across several services are useful. Do not treat OAuth and mTLS as competing concepts: mTLS proves device identity at the connection; OAuth can express API permissions.

## 7. Upload API and flow

Example API:

~~~http
POST /v1/upload-sessions
Authorization: device identity established by mTLS
Content-Type: application/json

{
  "captureId": "device-481-camera-1-seq-89325",
  "capturedAt": "2026-10-31T18:30:00Z",
  "latitude": 37.0,
  "longitude": -122.0,
  "headingDegrees": 90,
  "imageSizeBytes": 1048576,
  "checksum": "sha256:..."
}
~~~

The response contains an upload session and a short-lived, narrowly scoped object-storage URL.

### What the upload URL is

The URL is a temporary signed permission, created by the backend using its object-storage credentials. It authorizes a specific operation—normally an HTTP PUT to one exact object key—until it expires. The taxi does not receive cloud account keys or permission to browse the storage bucket.

The URL is **not** a public URL for viewing the image. The bucket stays private. Treat the upload URL like a short-lived bearer secret: anyone who obtains it may be able to perform its permitted upload until it expires. The edge agent must not log it or include it in ordinary health telemetry.

Do not assume a signed URL is automatically single-use. Depending on the storage provider and request conditions, it may be reusable until expiry; a repeated PUT might replace the object at that key. Use a unique key per upload session, bind completion to the expected capture ID, size, and checksum, and use conditional-create or object-versioning controls where supported. If a URL leaks, expire or cancel the session, reject its completion, and quarantine or delete any unexpected object.

### How the backend creates and the taxi uses a signed URL

The backend does not encrypt the image with the URL. It asks the storage provider's SDK to sign a narrowly defined storage request using backend credentials (preferably a short-lived cloud role). For an S3-style SigV4 URL, the signature is a keyed cryptographic MAC over a canonical request and credential/time scope. The request includes the HTTP method, bucket and object key, expiry, and any headers or query parameters that must match. The URL carries the signature and signing metadata as query parameters. Storage reconstructs the request when it arrives and checks the signature, expiry, credential permissions, and bucket policy. HTTPS/TLS separately encrypts the request and image bytes in transit.

Backend implementation outline:

1. Authenticate the taxi and validate the upload session against the catalog.
2. Generate an opaque object key on the server, such as `raw/region-a/2026/10/31/<random-id>`. Do not accept a caller-supplied bucket path.
3. Use the cloud SDK with the backend's least-privilege role to sign exactly `PUT` for that key, with a short expiry and only the required headers. Include a checksum header when the provider supports signed checksum validation.
4. Return the URL, required method and headers, expiry, and upload ID to the taxi. Keep the cloud credentials on the server; never return them to the device.
5. The taxi sends the bytes directly to the storage endpoint with that exact method and signed headers. It should send the URL unchanged and must not log it.
6. The taxi calls the authenticated completion API. The backend independently checks the stored object, expected length, checksum, and session/device relationship before accepting it.

Conceptually, the SDK call looks like this (provider-specific names and options differ):

~~~text
signed_url = storage_sdk.sign(
    operation = PUT_OBJECT,
    object_key = server_generated_key,
    expires_in = short_duration,
    required_headers = { content_type, checksum }
)
~~~

Example device request:

~~~http
PUT <signed-url>
Content-Type: image/jpeg
<provider checksum header>: <checksum value>

<raw image bytes>
~~~

The content type and any signed headers must match what the backend included when signing; changing the URL, method, path, or signed header can make validation fail. Keep device clocks synchronized because some providers validate signing timestamps. Use the provider SDK rather than implementing signature cryptography by hand.

For multipart uploads, the backend starts the multipart session and controls its upload ID. It returns a signed URL per part (each scoped to the object, upload ID, and part number), or another provider-supported resumable session. The taxi uploads parts and reports their identifiers/checksums; the backend validates the part list and completes the multipart upload. Abort incomplete sessions after a retention window to avoid paying for abandoned parts.

Revoking a database upload session prevents the completion API from accepting it, but may not immediately invalidate a URL already issued to object storage. Treat it as usable until its expiry unless the provider offers a policy/credential-level revocation mechanism. This is why URL lifetime should be short and the object key should be unique to the session.

### Upload session sequence

1. The edge agent authenticates to the Upload API with mTLS and sends capture metadata, expected size, content type, and checksum.
2. The API checks the device identity, upload policy, limits, and capture ID. It creates a PENDING upload record and assigns an opaque, unique object key.
3. The API returns an upload ID, the exact HTTP method and required headers, a signed URL, and its expiry. The permission should be limited to that object key and operation; enforce size, content type, and checksum constraints where the storage provider supports them.
4. The edge agent sends the image bytes directly to private object storage over HTTPS. The application servers do not proxy the large image body.
5. The edge agent calls a completion endpoint over mTLS. For multipart uploads, the backend verifies the uploaded parts and finalizes the object through the storage API.
6. The backend checks object existence, size, checksum, upload ID, and device ownership. It marks the capture RECEIVED, records an ingestion event, and returns a durable acknowledgement.
7. Only after that acknowledgement may the edge agent clean up its local copy. Processing then continues asynchronously.

For large images or poor connections, use multipart upload: upload numbered parts independently and resume only missing parts. If a signed URL expires, the device asks the Upload API for a replacement URL for the same authorized session; it does not need to restart completed parts. The session has an overall expiry and can be aborted to remove incomplete parts.

### Connectivity and upload policy

Assume cellular connectivity will be intermittent, slow, or unavailable for hours. The system should be store-and-forward: capture and durable local storage must continue without a network, and the device deletes a capture only after the backend acknowledges durable receipt.

- Size encrypted local storage for the expected capture volume over the target offline window, with headroom for delayed depot uploads. Monitor free space and estimate time until full.
- Resume uploads by part or chunk and retry with exponential backoff plus jitter. Keep capture IDs and upload sessions stable across retries so reconnects do not create duplicate logical captures.
- Request fresh signed URLs or OAuth access tokens when they expire. Expiry should limit exposure, but should not force an image or already completed parts to be uploaded again.
- Adapt upload concurrency and bandwidth to cellular quality, data cost, and vehicle operation. Prioritize older buffered captures or strategically valuable coverage; use depot Wi-Fi for bulk backlog draining when available.
- Define behavior before the local spool fills: alert the backend when possible, reduce capture rate or image size according to policy, and preserve the most valuable pending captures. Never silently delete unacknowledged images.
- Track queue depth, oldest pending capture age, upload throughput, retry counts, and estimated time to spool exhaustion. These reveal whether the fleet is keeping up with its capture rate.

Brief outages are handled by retrying and resuming. Long outages are primarily a local capacity and prioritization problem; no network protocol can upload data while the taxi is disconnected.

Signed URLs are limited bearer capabilities, so keep expirations short, restrict each URL to one object and one method, use unique object keys, keep storage private, redact URLs from logs, and rate-limit session creation. A leaked URL should not grant read, list, or delete access. Completion is authenticated separately with mTLS and must be idempotent.

### End-to-end write path

1. Capture service records an image and synchronized metadata.
2. Edge agent writes both to its durable local spool.
3. When connected, it requests an upload session using its authenticated device identity.
4. Upload service validates authorization and returns the expected object key and short-lived upload permission.
5. Edge agent uploads the image, resuming interrupted multipart uploads.
6. Completion service verifies object existence, expected size, checksum, capture ID, and device ownership.
7. It records the capture as received and emits an ingestion event.
8. The taxi receives durable acknowledgement and may later clean up its local copy.

The capture ID remains stable across retries. Re-requesting a URL or repeating completion for the same session must not create a second logical image. If stricter inspection is required before storage, the API can proxy uploads, at the cost of more application bandwidth and compute.

## 8. Data model

### Capture metadata

~~~text
capture_id                 stable unique ID generated by device
device_id                  registered camera/edge identity
vehicle_assignment_id      assignment reference, access controlled
captured_at                UTC capture time
latitude / longitude       capture position
heading                    travel or camera heading
camera_orientation         calibrated orientation
calibration_version        camera calibration reference
image_dimensions           width and height
image_format               encoded format
image_size_bytes           expected object length
checksum                   content checksum
raw_object_key             restricted raw object reference
processed_object_keys      approved derivative references
upload_status              received / failed / quarantined
processing_status          pending / processed / rejected
privacy_status             unprocessed / blurred / approved
map_data_version           map-matching context/version
~~~

### Image identity

A stable capture ID can be built from a provisioned device ID, camera ID, and monotonic capture sequence. The ID and checksum support retries, deduplication, and reconciliation after interrupted uploads.

Treat the capture ID as the image's durable logical identity. It is generated once at capture time and does not change when the image is copied, blurred, resized, re-encoded, moved to another storage tier, or replicated to another region. Do not use a signed URL, CDN URL, or object-store path as the image's identity: those are access or storage details that can change or expire.

Keep a separate asset/version record for every stored byte representation. For example, one capture may have an original object, a blurred display image, and several thumbnails. Each asset record points to an immutable object version and records its role, processing version, checksum, size, format, and lifecycle state. A `capture_id` can therefore resolve to multiple assets, while a particular `asset_id` identifies one exact representation.

Example logical records:

~~~text
Capture
  capture_id: cap_01J...
  captured_at: 2026-10-31T18:30:00Z
  location: { latitude: ..., longitude: ... }
  heading: ...
  status: READY
  current_display_asset_id: asset_blurred_v3

Asset
  asset_id: asset_blurred_v3
  capture_id: cap_01J...
  role: DISPLAY
  object_key: processed/region-a/2026/10/31/ab/cap_01J/blur-v3.webp
  object_version: provider-version-id
  checksum: sha256:...
  processing_version: blur-pipeline-v3
  state: AVAILABLE
~~~

The object key is an internal locator. Keep it in a controlled metadata/catalog service and construct it server-side; clients and downstream systems should normally exchange `capture_id` or `asset_id` instead. This indirection lets storage layout, provider, and processing versions change without changing consumer references.

Keep personally identifying driver or vehicle ownership information in a separately controlled registry. Most image-processing consumers need capture ID and location, not operator identity.

## 9. Storage and indexing

### Image objects

Use durable object storage for large image files. Example key shape:

~~~text
/{region}/{capture-date}/{device-shard}/{capture-id}/{version}
~~~

Use distinct versions for raw, privacy-processed, thumbnail, and other derivatives. Avoid placing sensitive identifiers in object keys. Raw images live in a restricted bucket or namespace with stricter access controls.

### Metadata store

Store capture metadata separately from image bytes. Support:

- Lookup by capture ID
- Batch retrieval by capture IDs
- Processing and privacy status updates
- References to object versions
- Audit and deletion state

### Server-side metadata storage

The Upload API writes metadata to a durable database; it never puts the image bytes in that database. Keep a **capture catalog** as the authoritative record and a **geospatial search index** as a derived index optimized for nearby-image queries.

For an initial deployment, a relational database such as PostgreSQL can hold upload sessions, capture records, asset records, and an event outbox. Transactions and constraints make it convenient to enforce unique capture IDs and consistent state changes. At fleet scale, choose a sharded or distributed database based on measured write rate, query patterns, and operational expertise; a key-value or wide-column store can work well for high-volume capture/asset lookups. The geospatial index may be a database geo-index or a dedicated search service. There is no single database choice required by the design.

Illustrative tables / records:

~~~text
upload_session(
  upload_id, device_id, capture_id, state, expected_size, expected_checksum,
  expires_at, created_at
)

capture(
  capture_id, device_id, captured_at, latitude, longitude, heading,
  capture_metadata, ingestion_state, privacy_state, created_at, updated_at
)

asset(
  asset_id, capture_id, role, object_key, object_version, checksum,
  size_bytes, format, processing_version, state, created_at
)

outbox_event(
  event_id, aggregate_id, event_type, payload, created_at, published_at
)
~~~

The exact schema can be normalized across relational tables or represented as keyed records, but preserve the distinctions: one capture can have several asset versions, and one device can produce many captures. Enforce uniqueness on `capture_id` and on each `asset_id`; use idempotency keys so upload retries resolve to the existing logical record.

Write flow: create a pending upload session when issuing upload permission; after the object store confirms the completed object and checksum/size checks pass, atomically mark the capture received, add its original asset record, and insert an outbox event. A separate publisher sends outbox events to the queue and marks them published. This avoids losing the processing notification if the database commit succeeds but queue publication fails. Workers update processing state and add derivative asset records idempotently; the geo index is updated from those events only when the capture is eligible for search.

The capture catalog is the source of truth. The queue and geo index may lag briefly and must be rebuildable or repairable from catalog records/events. Periodically reconcile catalog references against object storage, and track unreferenced objects for safe cleanup after a grace period. Store location and vehicle-linked metadata with strict access controls because it can reveal sensitive movement patterns.

Keep capture metadata and asset metadata distinct. Capture metadata describes when and where the camera took a picture. Asset metadata describes the exact bytes created from it. Use immutable object keys per asset version where possible; never overwrite the only copy of an image in place. Update the catalog's pointer to the currently approved display asset when a newer processing version is ready.

### Geospatial index

Index processed images by a geographic cell such as S2 or H3, then filter by time, heading, quality, and capture status. A nearby-image query searches the target cell and neighboring cells, calculates exact distance, and selects suitable imagery. The index stores references and searchable attributes, not image bytes.

#### Recommended implementation

For an initial version:

1. Keep image bytes in object storage and capture/asset metadata in PostgreSQL.
2. Store each capture coordinate as a spatial point and create a GiST index with PostGIS.
3. Query by radius with `ST_DWithin`, then rank matches by exact distance, heading, image quality, time, and map match.
4. Add ordinary database indexes for common filters such as `privacy_status`, `captured_at`, and road/map version. Add composite indexes only after checking real query plans.
5. Publish captures to general search only after processing and privacy checks succeed; keep raw captures out of user-facing search.

This provides one authoritative metadata system and avoids adding geohash/H3 plumbing before there is a measured need.

#### Scaling the geospatial read path

The sizing assumption in this design (100 million captures/day) averages about 1,157 captures/second, before peaks and retries. Benchmark realistic ingest and read patterns before selecting a distributed design.

If a single Postgres/PostGIS deployment no longer meets write, storage, or query objectives:

- Partition catalog data by capture time and/or region according to retention and query patterns.
- Use a durable event stream to feed a horizontally scalable search/index projection.
- Consider H3/S2 cells as routing keys for regional partitions or candidate groups, with additional time/shard dimensions to avoid hot cells.
- Use a geo-capable search tier for high-volume nearby queries with many filters; retain a canonical catalog and rebuildable index.
- Preserve precise coordinates and run an exact radius/distance check after cell-based candidate retrieval.
- Measure skew: downtown cells can be far hotter than rural areas. Use adaptive cell resolution or secondary partitioning rather than relying on one fixed geohash prefix.

Every maintained projection adds write fan-out, lag, repair logic, and operating cost. Add a cell index or dedicated geo-search database only to solve a measured need.

Example flow:

~~~text
Image bytes ───────────────► Object storage

Capture metadata ──────────► Authoritative catalog (Postgres/PostGIS initially)
                                   │
                                   ├─ spatial index for nearby candidate lookup
                                   └─ outbox/event stream
                                          │
                                          ├─► optional search projection (Elasticsearch/OpenSearch)
                                          └─► optional H3/S2 analytics or partition keys

Nearby query ─► geo candidate search ─► exact distance + status/heading/quality filters
                                      ─► capture_id / asset_id ─► authorized image read
~~~

### Capacity and retention

The sample assumption above yields about 100 TB/day of raw data. Before production, measure real image sizes, capture policies, compression, and coverage requirements. Use lifecycle rules for raw and derivative data, and define retention separately for each version. Restrict replication and backup to the required durability and recovery targets because each increases storage cost.

## 10. Asynchronous processing

Workers consume ingestion events and:

1. Verify object existence, format, and checksum.
2. Reject corrupted or incomplete images into a quarantine state.
3. Detect exact and near-duplicate captures.
4. Run image quality checks.
5. Blur faces and license plates before general access.
6. Validate GPS and map-match the capture to a road segment.
7. Generate thumbnails or display derivatives.
8. Update metadata and geospatial indexes.
9. Emit processing-complete or rejection events for downstream consumers.

Use a transactional outbox or equivalent mechanism so a durable upload record cannot be committed without eventually publishing its event. Workers should be idempotent and retry transient failures. Persistently failing items go to a dead-letter queue for investigation.

## 11. Read path and downstream consumers

Downstream systems can consume processing events or query authorized batches of capture IDs. This keeps map generation and image-understanding workloads independent from taxi upload traffic.

Example nearby-image query:

~~~http
GET /v1/images/nearby?lat=37.0&lon=-122.0&radius=50&heading=90
~~~

The image query service:

1. Searches nearby geospatial cells.
2. Filters to processed, approved, accessible images.
3. Selects candidates by distance, heading match, quality, and recency.
4. Returns metadata and short-lived image URLs.

### How consumers reference an image

Consumers use stable IDs to ask the platform for an image, rather than building object-store paths themselves:

~~~text
Geospatial index → capture_id
Capture catalog → capture metadata + eligible asset_id
Asset catalog → private object key + version/checksum
Access service → short-lived read URL or proxied response
~~~

For example, the nearby-image API returns a `capture_id`, relevant capture metadata, and either an `asset_id` or a short-lived display URL. A map-generation worker can request a batch of capture IDs, while a browser or mobile client receives only an approved display asset URL. Raw imagery uses a separate privileged access path. A URL is temporary permission to fetch bytes; it is not a durable reference to the image.

The platform should resolve IDs and enforce access each time a consumer requests bytes. It can then change the underlying bucket, region, CDN, or derivative without changing the consumer's saved `capture_id`. Keep object-store keys out of public APIs, and avoid exposing raw asset IDs where they could reveal internal storage structure.

### State and publication

An image moves through states such as `PENDING_UPLOAD → RECEIVED → PROCESSING → READY`, or to `QUARANTINED` / `REJECTED` when checks fail. `RECEIVED` means the original bytes and metadata are durably recorded; it does not mean the image is safe or available to general consumers. Publish it to the geospatial search index only after required quality, location, and privacy checks pass. If a newer derivative is being generated, keep serving the previous approved asset until the replacement is validated, then atomically update the catalog pointer.

Object storage and the metadata database do not share one transaction. Make processing idempotent and reconcile both directions: find catalog records whose objects are missing, and stored objects with no valid catalog record. Use lifecycle cleanup for abandoned multipart uploads and orphaned temporary objects. Deletion must follow the references from a capture to every asset version, index entry, cache, and downstream copy.

Serve only approved derivatives to general clients. Use a CDN with signed, expiring URLs or an authenticated image proxy. Keep raw object paths inaccessible to clients.

## 12. Privacy, security, and deletion

- Encrypt network traffic with modern TLS and encrypt local spool and object storage.
- Protect device private keys in secure hardware where available.
- Use least-privilege access policies for raw imagery, processed imagery, and metadata.
- Blur faces and license plates before general-purpose use.
- Audit access to raw images and location data.
- On deletion or retention expiry, remove objects, derivatives, metadata indexes, caches, and downstream copies according to policy.
- On unfollowed/reassigned devices or revoked credentials, deny new uploads and clean up stale sessions.
- Use signed short-lived upload permissions; never expose permanent object-store credentials to taxis.

## 13. Reliability and operations

- **Network outages:** Buffer locally and resume uploads when connectivity returns.
- **Backpressure:** Monitor queue depth and processing delay; scale workers independently from upload services.
- **Retries:** Use exponential backoff with jitter and bounded retry policies.
- **Idempotency:** Stable capture IDs and checksums prevent retries from creating duplicate logical records.
- **Device health:** Monitor last-seen time, camera status, disk capacity, GNSS quality, temperature, and upload backlog.
- **Service metrics:** Track upload success, checksum failures, queue age, image-query latency, storage growth, privacy-processing status, and deletion completion.
- **Regional resilience:** Place ingestion near the fleet and replicate data according to recovery objectives.
- **Configuration safety:** Roll out signed capture-policy and software changes gradually, with health checks and rollback.

## 14. Key design choices and tradeoffs

| Choice | Benefit | Cost / tradeoff |
| --- | --- | --- |
| Direct resumable upload to object storage | Avoids routing large image bodies through API servers | Needs upload-session and cleanup handling |
| Async queue-driven processing | Upload remains responsive during processing spikes | Searchability is delayed |
| Separate image objects and metadata index | Scales blob storage and query workloads independently | Must reconcile metadata and object lifecycle |
| Per-device X.509 identity | Strong device identity; supports revocation per taxi | Requires enrollment, rotation, and fleet PKI |
| OAuth short-lived scoped tokens (optional) | Centralized API scopes and token lifecycle | Adds an authorization service and token operations |
| Hybrid raw/processed storage | Protects raw data while serving safe derivatives | Additional storage and lifecycle complexity |

## 15. FAQ and design clarifications

### Does the taxi request a presigned URL for every image?

In the straightforward design, yes: each image has its own upload session, unique object key, and short-lived presigned URL. The taxi may request sessions for a small batch to reduce API round trips, but each image still gets a URL scoped to its own object. Multipart uploads use separate scoped permissions for the parts of one image. Do not reuse a URL across different images.

### What if someone changes the object ID in a copied URL?

The object key is part of the signed request. Changing the key changes the request, so storage should reject it because the signature no longer matches. Someone who copies the complete URL can still use the permission it grants until expiry; for an S3 presigned `PUT`, that may let them upload to the signed key, potentially replacing an object already at that key. Use a unique, unpredictable key per session, a short expiry, and verify the completed object's size and checksum before accepting it. Treat the full URL as a bearer secret.

### What is a geospatial index?

A geospatial index is a search structure for finding records by geographic location efficiently. Instead of checking every capture's latitude and longitude, the system groups locations into geographic cells (for example, using H3 or S2), searches the cells near a requested point, and then calculates exact distance for the candidates. The index holds locations and capture references, not image bytes. It is updated only when a capture is eligible for search, then results resolve through `capture_id` to approved asset metadata and image access.

### How are presigned URLs different from image references?

`capture_id` is the durable logical identity of the capture; `asset_id` identifies a particular representation such as the original or blurred image. The object key is an internal storage locator. A presigned URL is temporary permission to perform one storage operation and expires, so it must not be saved as the permanent image reference. Consumers use IDs and ask the platform to resolve them to an authorized asset or temporary URL.

## 16. Likely interviewer follow-up questions

Use these to practice explaining the design and its tradeoffs. An interviewer may choose a few and then keep drilling into whichever assumptions you make.

### Requirements and capacity

- How many taxis, images per taxi, and bytes per image are we designing for? What are the average and peak upload rates?
- What freshness target do downstream systems need: seconds, minutes, or hours?
- How long must raw images and approved derivatives be retained?
- What availability, durability, and recovery objectives are required?
- Which assumptions dominate storage, network, and processing costs?

### Taxi capture and upload

- How does the edge device avoid losing or duplicating captures after a crash or power loss?
- How much local storage is needed for a taxi offline for a day or a week?
- How are uploads prioritized when a taxi reconnects with a large backlog?
- How do retries, idempotency, and multipart resume work after the URL or OAuth token expires?
- What happens when the device clock is wrong, GPS is missing, or capture metadata is malformed?
- How do we prevent one faulty or compromised taxi from exhausting upload capacity?

### Authentication and upload security

- How are devices enrolled, certificates rotated, revoked, and recovered after hardware replacement?
- When is mTLS alone enough, and when do OAuth scopes add value?
- What exactly can a presigned URL do, and how do we limit damage if it leaks?
- How do we prevent overwrites, replayed completion requests, and uploads that do not match the expected checksum or size?
- How are credentials and signed URLs kept out of logs and telemetry?

### Storage and metadata

- Why store image bytes in object storage instead of a relational database or file system?
- What is the authoritative metadata record, and how do we reconcile it with object storage?
- How do we handle an object upload that succeeds while the metadata write or event publication fails?
- How are duplicate captures detected, and should identical bytes share storage?
- How do object versioning, lifecycle tiers, replication, and deletion affect cost and recovery?
- How do we migrate storage providers or change object-key layouts without breaking consumers?

### Processing and data quality

- How do we make processing workers idempotent and safely retry failed jobs?
- What happens when blurring, map matching, or image understanding is delayed or fails?
- How do we ensure raw images cannot accidentally appear in user-facing results?
- Can a new processing pipeline version regenerate derivatives, and how are versions tracked?
- How do we handle late, out-of-order, duplicate, or incorrect GPS observations?

### Geospatial and read path

- How do we choose a geographic cell size and search neighboring cells near cell boundaries?
- How do we rank images by distance, heading, quality, recency, and coverage?
- What happens when many captures cluster in a busy city cell?
- How fresh can the geospatial index be, and how do we rebuild or repair it?
- How are image reads authorized, cached, and served through a CDN without exposing raw objects?

### Privacy, operations, and failure handling

- What privacy rules govern capture, retention, blurring, access, and deletion?
- How do deletion requests propagate to derivatives, indexes, caches, backups, and downstream consumers?
- What metrics and alerts reveal that uploads or processing are falling behind?
- What are the system's behavior and recovery plan during a regional storage or queue outage?
- How do we detect and respond to a compromised device or leaked upload URL?
- Which parts of the design would you simplify for an MVP, and what would trigger scaling them up?

## 17. Interview summary

The taxi captures an image with synchronized location and orientation, writes it to encrypted local storage, and uploads it resumably using a unique device identity. The backend verifies and stores the image, emits an event, and processes it asynchronously. Metadata and a geospatial index make approved derivatives discoverable to map and user-facing systems. The main design challenges are upload reliability, data volume, privacy processing, geospatial lookup, and lifecycle management.

