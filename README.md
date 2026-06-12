# Border Common Operating Picture System - BCOP Lite

## 1. Project Purpose

BCOP Lite is a backend-first command and incident management system for authorized border-security operators.

The system's purpose is to integrate signals from existing hardware, edge processing nodes, and field/operator inputs into one central operational picture(main server).

The main goal is:

> Convert raw device data into verified operational incidents.

Core mental model:

- Device = source
- Edge Gateway = local processor
- DeviceEvent = detected signal
- Incident = verified operational case
- Evidence = supporting media/document
- Dashboard = operational view
- Report = official record

BCOP Lite is not a public alert app, public border map, vigilante tool, facial recognition system, or drone-control platform.

## 2. System Scope

### MVP Scope

The first backend milestone should build the event pipeline:

1. Device registry
2. Gateway registry
3. Heartbeat API
4. Event ingestion API
5. Event storage
6. Event list/detail API
7. Admin support
8. Fake gateway simulator

The MVP should not build AI, radar processing, drone control, WebSockets, PostGIS, incident workflow, or advanced dashboard features yet.

### Future Scope

After the pipeline works, add:

- Device health history
- Incident workflow
- Incident evidence
- Operator roles and permissions
- Audit logs
- Realtime dashboard alerts
- Map dashboard
- AI edge service
- PostGIS geospatial queries
- MinIO or S3-compatible media storage

## 3. High-Level Architecture

### Layer 1: Device Registry

The central backend stores all hardware and field-input source records.

Supported device types:

- CCTV
- PTZ
- THERMAL
- RADAR
- DRONE
- ACOUSTIC
- FIELD_REPORT_APP

The backend needs to know:

- Which devices exist
- Where each device is installed
- Which border sector each device belongs to
- Which edge gateway handles each device
- Whether each device is active, inactive, offline, or in error state
- Which protocol each device uses
- Stream URL or API details where applicable

### Layer 2: Edge Gateway

An Edge Gateway is a local processing node near devices.

It may be:

- Mini PC
- Local server
- Rugged edge box
- NVR-adjacent machine
- GPU/CPU processing box

The Edge Gateway is responsible for:

- Reading camera streams locally
- Decoding video frames
- Running AI or motion detection locally
- Applying local rules
- Keeping a short rolling video buffer
- Creating normalized events
- Sending only important events to the central backend
- Sending heartbeat and health status to the backend

The central backend should not receive or process all raw camera footage continuously.

Correct data flow:

```text
Camera/Device -> Local Edge Gateway -> Event/Clip/Metadata -> Central Backend
```

Reason:

One 1080p CCTV stream may use around 2-8 Mbps. At 4 Mbps:

- 4 Mbps = about 0.5 MB/s
- Per day = about 43 GB per camera
- 100 cameras = about 4.3 TB/day
- 30 days = about 129 TB/month

Centralizing all raw video is expensive, slow, bandwidth-heavy, and unreliable.

### Layer 3: Event Ingestion API

The Edge Gateway sends normalized events to the central backend.

Example event payload:

```json
{
  "gateway_id": "gw_benapole_01",
  "device_id": "cam_12",
  "event_type": "suspicious_group_movement",
  "confidence": 0.82,
  "severity": "high",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "occurred_at": "2026-06-12T02:10:00",
  "snapshot_url": "snapshot_123.jpg",
  "clip_url": "clip_123.mp4"
}
```

The backend validates:

- Gateway is authenticated
- Gateway exists
- Device exists
- Device belongs to the gateway
- Device and gateway belong to the expected sector
- Event type is valid
- Confidence is between 0.0 and 1.0
- Severity is valid
- Location is valid
- Timestamp is valid

After validation, the backend saves the event and makes it available to dashboard and alert workflows.

### Layer 4: Event Fusion and Incident Creation

A `DeviceEvent` is not an `Incident`.

DeviceEvent:

- Raw signal or detection
- May be wrong
- May be duplicated
- May need human review

Incident:

- Verified operational case
- Created by an operator or trusted rule engine
- Can contain multiple evidence records
- Can be assigned, resolved, or marked as false alarm

Example:

- Camera detects 6 people
- Radar detects movement nearby
- Thermal camera confirms heat signatures
- Patrol report arrives from same area

The system can combine these signals and raise confidence. An operator may then convert one or more events into an incident.

### Layer 5: Dashboard, Alert, and Workflow

The dashboard should eventually show:

- Map of sectors
- Devices
- Gateways
- Active alerts
- Device health
- Incident timeline
- Evidence clips
- Operator actions
- Event-to-incident workflow

Future workflow:

```text
new event
-> reviewing
-> ignored or converted_to_incident
-> incident open
-> assigned
-> responding
-> resolved or false_alarm
-> report generated
```

## 4. Recommended Tech Stack

Backend:

- Django
- Django REST Framework
- PostgreSQL
- PostGIS later for geospatial queries
- Redis later
- Celery later
- Django Channels or WebSocket later

Media storage:

- Local storage for MVP
- MinIO or S3-compatible storage later

AI/Edge service:

- Python service
- OpenCV
- FFmpeg
- FastAPI optional
- YOLO or RT-DETR later
- Fake event generator for MVP

Frontend/dashboard:

- React/Next.js or similar
- Leaflet or MapLibre for map

Deployment:

- On-premise or private cloud preferred
- Public SaaS should not be assumed for sensitive border data

## 5. Core Backend Models

### 5.1 BorderSector

Represents a logical border area.

Fields:

- `id`
- `name`
- `district`
- `upazila`
- `description`
- `risk_level`
- `boundary_polygon`, optional later with GIS/PostGIS
- `created_at`
- `updated_at`

Suggested risk levels:

- `low`
- `medium`
- `high`
- `critical`

Relationships:

- One `BorderSector` has many `EdgeGateways`
- One `BorderSector` has many `Devices`
- One `BorderSector` has many `DeviceEvents`
- One `BorderSector` has many `Incidents`

### 5.2 EdgeGateway

Represents a local processing node.

Fields:

- `id`
- `external_id`, for example `gw_benapole_01`
- `sector_id`
- `name`
- `location_lat`
- `location_lng`
- `ip_address`
- `status`
- `last_heartbeat_at`
- `auth_token_hash`
- `metadata`
- `created_at`
- `updated_at`

Suggested statuses:

- `online`
- `offline`
- `error`
- `maintenance`

Relationships:

- One `EdgeGateway` belongs to one `BorderSector`
- One `EdgeGateway` has many `Devices`
- One `EdgeGateway` sends many `DeviceEvents`
- One `EdgeGateway` sends many `DeviceHealthLogs` later

Important design decision:

Use a database primary key internally, but expose `external_id` for gateway communication. This allows user-friendly IDs like `gw_01` without locking database structure to external naming.

### 5.3 Device

Represents CCTV, PTZ, thermal, radar, drone, acoustic sensor, or field report source.

Fields:

- `id`
- `external_id`, for example `cam_01`
- `gateway_id`
- `sector_id`
- `name`
- `type`
- `protocol`
- `stream_url`
- `location_lat`
- `location_lng`
- `status`
- `installed_at`
- `metadata`
- `created_at`
- `updated_at`

Suggested device types:

- `CCTV`
- `PTZ`
- `THERMAL`
- `RADAR`
- `DRONE`
- `ACOUSTIC`
- `FIELD_REPORT_APP`

Suggested protocols:

- `RTSP`
- `ONVIF`
- `API`
- `MANUAL`
- `UPLOAD`

Suggested statuses:

- `active`
- `inactive`
- `offline`
- `error`

Relationships:

- One `Device` belongs to one `EdgeGateway`
- One `Device` belongs to one `BorderSector`
- One `Device` creates many `DeviceEvents`
- One `Device` may provide many `IncidentEvidence` records later

Validation rule:

The device sector should match the gateway sector unless there is a specific operational reason to allow otherwise.

### 5.4 DeviceEvent

Represents a machine or system detected signal.

This is the most important MVP model.

Fields:

- `id`
- `device_id`
- `gateway_id`
- `sector_id`
- `event_type`
- `confidence`
- `severity`
- `location_lat`
- `location_lng`
- `occurred_at`
- `status`
- `metadata`
- `snapshot_url`
- `clip_url`
- `created_at`

Suggested event types:

- `person_detected`
- `group_movement`
- `zone_crossing`
- `boat_detected`
- `vehicle_detected`
- `possible_gunshot`
- `radar_movement`
- `suspicious_activity`

Suggested severities:

- `low`
- `medium`
- `high`
- `critical`

Suggested event statuses:

- `new`
- `reviewing`
- `ignored`
- `converted_to_incident`

Validation rules:

- `confidence` must be between `0.0` and `1.0`
- `occurred_at` must be a valid timestamp
- `device_id` must identify an existing device
- `gateway_id` must identify an existing gateway
- The device must belong to the gateway
- The event sector should be derived from the device/gateway instead of blindly trusted from request payload

Important:

`DeviceEvent` is not confirmed truth. It is a detected signal.

### 5.5 DeviceHealthLog

Not required in the first milestone, but should be added soon after.

Tracks device and gateway health.

Fields:

- `id`
- `device_id`, nullable
- `gateway_id`
- `status`
- `cpu_usage`
- `memory_usage`
- `disk_usage`
- `network_status`
- `message`
- `reported_at`
- `created_at`

Purpose:

The command center must know whether cameras and gateways are actually working.

### 5.6 Incident

Not required in the first milestone.

Represents a verified operational case created by an operator or trusted rule engine.

Fields:

- `id`
- `sector_id`
- `title`
- `incident_type`
- `severity`
- `status`
- `confidence_score`
- `location_lat`
- `location_lng`
- `started_at`
- `resolved_at`
- `created_from_event_id`, nullable
- `assigned_to`, nullable
- `summary`
- `created_by`
- `created_at`
- `updated_at`

Suggested incident types:

- `push_in_attempt`
- `firing`
- `suspicious_crossing`
- `smuggling`
- `injured_person`
- `river_incident`
- `crowd_gathering`
- `unknown`

Suggested incident statuses:

- `open`
- `assigned`
- `responding`
- `resolved`
- `false_alarm`

Important:

An `Incident` should be created only after human verification or strong multi-source confidence.

### 5.7 IncidentEvidence

Not required in the first milestone.

Represents evidence attached to an incident.

Fields:

- `id`
- `incident_id`
- `evidence_type`
- `file_url`
- `source_type`
- `device_id`, nullable
- `uploaded_by`, nullable
- `captured_at`
- `metadata`
- `created_at`

Suggested evidence types:

- `image`
- `video`
- `audio`
- `document`
- `note`

Suggested source types:

- `camera`
- `drone`
- `patrol_app`
- `operator_upload`
- `radar`
- `acoustic`

### 5.8 User / Operator

Use Django's user system or a custom user model.

Roles:

- `admin`
- `operator`
- `verifier`
- `patrol_officer`
- `viewer`

Suggested fields:

- `id`
- `name`
- `email`
- `phone`
- `role`
- `assigned_sector`
- `is_active`
- `created_at`

Access control is important because this is an authorized-use system.

## 6. MVP API Design

Base path:

```text
/api/
```

### 6.1 Border Sector APIs

```text
GET /api/sectors/
POST /api/sectors/
GET /api/sectors/{id}/
PATCH /api/sectors/{id}/
DELETE /api/sectors/{id}/
```

Purpose:

Create and manage border sectors.

Create payload:

```json
{
  "name": "Benapole Sector",
  "district": "Jashore",
  "upazila": "Sharsha",
  "description": "High activity border sector",
  "risk_level": "high"
}
```

### 6.2 Gateway APIs

```text
GET /api/gateways/
POST /api/gateways/
GET /api/gateways/{id}/
PATCH /api/gateways/{id}/
DELETE /api/gateways/{id}/
POST /api/gateways/{id}/heartbeat/
```

Purpose:

Create and manage edge gateways. Receive heartbeat updates.

Create payload:

```json
{
  "external_id": "gw_benapole_01",
  "sector": "sector_uuid_or_id",
  "name": "Benapole Gateway 01",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "ip_address": "192.168.1.20",
  "status": "offline",
  "metadata": {
    "hardware": "mini_pc",
    "gpu": "none"
  }
}
```

Heartbeat payload:

```json
{
  "status": "online",
  "cpu_usage": 62,
  "memory_usage": 71,
  "disk_usage": 55,
  "devices": [
    {
      "device_id": "cam_01",
      "status": "online"
    },
    {
      "device_id": "cam_02",
      "status": "offline"
    }
  ]
}
```

Heartbeat behavior:

- Authenticate gateway
- Update gateway status
- Update `last_heartbeat_at`
- Optionally update linked device statuses
- Later, create `DeviceHealthLog` records

### 6.3 Device APIs

```text
GET /api/devices/
POST /api/devices/
GET /api/devices/{id}/
PATCH /api/devices/{id}/
DELETE /api/devices/{id}/
```

Purpose:

Create and manage devices.

Create payload:

```json
{
  "external_id": "cam_01",
  "gateway": "gateway_uuid_or_id",
  "sector": "sector_uuid_or_id",
  "name": "North Fence Camera 01",
  "type": "CCTV",
  "protocol": "RTSP",
  "stream_url": "rtsp://example.local/stream1",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "status": "active",
  "installed_at": "2026-06-12T10:00:00",
  "metadata": {
    "resolution": "1080p",
    "night_vision": true
  }
}
```

Validation behavior:

- Device external ID should be unique per system or unique per gateway
- Gateway must exist
- Sector must exist
- Device sector should match gateway sector
- Protocol must be valid for the selected device type where possible

### 6.4 Event Ingestion API

```text
POST /api/events/
```

Purpose:

Receive normalized events from edge gateways.

Payload:

```json
{
  "gateway_id": "gw_01",
  "device_id": "cam_01",
  "event_type": "group_movement",
  "confidence": 0.82,
  "severity": "high",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "occurred_at": "2026-06-12T03:10:20",
  "snapshot_url": "snapshot.jpg",
  "clip_url": "clip.mp4",
  "metadata": {
    "object_count": 6,
    "direction": "towards_border",
    "time_context": "night"
  }
}
```

Expected behavior:

- Authenticate gateway
- Look up gateway by `gateway_id`
- Look up device by `device_id`
- Confirm device belongs to gateway
- Derive sector from device/gateway
- Validate event type
- Validate confidence
- Validate severity
- Save event with status `new`
- Return created event response

Suggested success response:

```json
{
  "id": "event_uuid_or_id",
  "status": "new",
  "message": "Event accepted"
}
```

### 6.5 Event List APIs

```text
GET /api/events/
GET /api/events/{id}/
PATCH /api/events/{id}/
```

Purpose:

Allow authorized operators to view and review device events.

Filters:

- `sector`
- `device`
- `gateway`
- `severity`
- `status`
- `event_type`
- `occurred_at_after`
- `occurred_at_before`

Example:

```text
GET /api/events/?severity=high&status=new&event_type=group_movement
```

Review behavior:

- Operators may change event status from `new` to `reviewing`
- Operators may mark event as `ignored`
- Later, operators may convert event to incident

## 7. Serializer and Validation Design

### BorderSectorSerializer

Responsible for:

- Validating sector details
- Returning sector fields for list/detail APIs

### EdgeGatewaySerializer

Responsible for:

- Validating gateway setup
- Showing sector relationship
- Keeping gateway metadata flexible

### DeviceSerializer

Responsible for:

- Validating device setup
- Ensuring gateway and sector consistency
- Supporting optional stream URL

### DeviceEventSerializer

Responsible for:

- Returning event details to operators
- Supporting event review status changes

### EventIngestionSerializer

Separate from the normal `DeviceEventSerializer`.

Responsible for:

- Accepting external IDs from gateways
- Looking up real gateway and device records
- Validating gateway-device relationship
- Deriving sector
- Creating a valid `DeviceEvent`

This separation keeps external ingestion concerns away from internal admin/dashboard representation.

### GatewayHeartbeatSerializer

Responsible for:

- Validating gateway status
- Validating optional resource usage values
- Validating device status updates
- Updating gateway and device status records

## 8. View and API Structure

Recommended DRF structure:

- `BorderSectorViewSet`
- `EdgeGatewayViewSet`
- `DeviceViewSet`
- `DeviceEventViewSet`
- Custom action: `EdgeGatewayViewSet.heartbeat`

Recommended API routing:

```text
/api/sectors/
/api/gateways/
/api/gateways/{id}/heartbeat/
/api/devices/
/api/events/
```

The event ingestion endpoint may use the same `/api/events/` POST route, but with ingestion-specific validation.

Alternative:

```text
POST /api/ingest/events/
```

For MVP, using `POST /api/events/` is acceptable if permissions and serializer selection are clear.

## 9. Authentication and Authorization

### MVP Authentication

Use two access paths:

1. Admin/operator access through Django admin or normal authenticated users
2. Gateway access through gateway token authentication

Gateway requests should include a token header, for example:

```text
Authorization: GatewayToken <token>
```

or:

```text
X-Gateway-Token: <token>
```

Recommended MVP behavior:

- Store only a hash of the gateway token
- Show the raw token only once when generated
- Require token for heartbeat
- Require token for event ingestion
- Ensure the token belongs to the gateway submitting the event

### Future Authorization

Add role-based access control:

- Admin can manage sectors, gateways, devices, users
- Operator can review events and manage incidents
- Verifier can verify incidents
- Patrol officer can submit reports/evidence
- Viewer can only read permitted dashboard data

### Audit Logging

All important actions should eventually be logged:

- Gateway created
- Device created
- Heartbeat received
- Event received
- Event status changed
- Incident created
- Incident assigned
- Incident resolved
- Evidence uploaded
- User login or permission-sensitive action

## 10. Security and Safety Rules

Do not build:

- Public border live map
- Public alert system
- Public patrol tracking
- Face recognition
- Nationality detection
- "Enemy detected" feature
- Targeting or vigilante features
- Unauthorized drone control

Required security direction:

- Authorized operators only
- Authenticated gateways only
- Role-based access control
- Audit logs
- Gateway token authentication
- Event validation
- Media access control
- Private/on-premise deployment preference

## 11. Fake Gateway Simulator

The fake gateway simulator is part of the MVP because the pipeline should be tested before AI is added.

Simulator responsibilities:

1. Use an existing gateway external ID, for example `gw_01`
2. Send heartbeat every 10 seconds
3. Simulate device statuses
4. Send fake `DeviceEvent` every 30-60 seconds
5. Print API responses and errors to the console

Example fake heartbeat:

```json
{
  "status": "online",
  "cpu_usage": 62,
  "memory_usage": 71,
  "disk_usage": 55,
  "devices": [
    {
      "device_id": "cam_01",
      "status": "online"
    }
  ]
}
```

Example fake event:

```json
{
  "gateway_id": "gw_01",
  "device_id": "cam_01",
  "event_type": "group_movement",
  "confidence": 0.82,
  "severity": "high",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "occurred_at": "2026-06-12T03:10:20",
  "snapshot_url": "snapshot.jpg",
  "clip_url": "clip.mp4",
  "metadata": {
    "object_count": 6,
    "direction": "towards_border",
    "time_context": "night"
  }
}
```

Later, this simulator can be replaced with:

- RTSP stream reader
- Frame extractor
- AI detector
- Local rule engine
- Event sender

## 12. Admin Support

The Django admin should allow authorized admins to manage:

- Border sectors
- Edge gateways
- Devices
- Device events

Admin list pages should show useful operational fields:

Border sectors:

- Name
- District
- Upazila
- Risk level

Gateways:

- External ID
- Name
- Sector
- Status
- Last heartbeat

Devices:

- External ID
- Name
- Type
- Gateway
- Sector
- Status

Device events:

- Event type
- Severity
- Confidence
- Status
- Device
- Gateway
- Sector
- Occurred at

## 13. Filtering and Query Behavior

Event list filters should support:

- Sector
- Device
- Gateway
- Severity
- Status
- Event type
- Date range

Default event ordering:

```text
occurred_at descending
```

Recommended dashboard query:

```text
GET /api/events/?status=new&severity=high
```

Recommended sector query:

```text
GET /api/events/?sector={sector_id}
```

Recommended date range query:

```text
GET /api/events/?occurred_at_after=2026-06-12T00:00:00&occurred_at_before=2026-06-13T00:00:00
```

## 14. Data Lifecycle

MVP:

- Events are stored in PostgreSQL
- Snapshot and clip URLs are stored as strings
- Actual media can be local files or placeholder URLs

Future:

- Store media in MinIO or S3-compatible storage
- Add retention policies
- Add evidence immutability rules for official incidents
- Add report generation
- Add audit trail for evidence access

## 15. Development Milestones

### Milestone 1: Backend Pipeline MVP

Build:

- Django project
- DRF setup
- PostgreSQL configuration
- `BorderSector` model
- `EdgeGateway` model
- `Device` model
- `DeviceEvent` model
- Serializers
- ViewSets
- API routes
- Heartbeat endpoint
- Event ingestion endpoint
- Event filtering
- Django admin registration
- Fake gateway simulator

Success criteria:

- Admin can create a sector
- Admin can create a gateway
- Admin can create a device
- Fake gateway can send heartbeat
- Fake gateway can send fake events
- Backend saves events
- Operator can list and filter events through API

### Milestone 2: Health and Incident Workflow

Build:

- `DeviceHealthLog`
- `Incident`
- `IncidentEvidence`
- Event-to-incident conversion
- Incident status workflow
- Basic role-based permissions

Success criteria:

- Heartbeat history is stored
- Operator can convert event to incident
- Incident can be assigned and resolved
- Evidence can be attached to incident

### Milestone 3: Realtime Dashboard

Build:

- WebSocket or Django Channels
- Realtime new-event notifications
- Gateway/device health updates
- Dashboard-ready summary APIs

Success criteria:

- Dashboard can receive new event alerts without polling
- Operators can see active sector/device health

### Milestone 4: Edge AI Integration

Build:

- RTSP reader
- Frame extractor
- AI detector
- Rule engine
- Local video buffer
- Clip extraction
- Event sender

Success criteria:

- Edge service watches real or test video stream
- Edge service sends real detection events
- Backend receives events using the same ingestion API

## 16. Important Design Principles

### Build the Pipeline Before AI

Do not build AI first.

Correct order:

```text
Device Registry
-> Gateway Registry
-> Heartbeat API
-> Event Ingestion API
-> Event Store
-> Event List API
-> Dashboard-ready data
-> AI and intelligence
```

### Keep Edge and Central Responsibilities Separate

Edge Gateway responsibilities:

- Raw stream reading
- Local processing
- AI/motion detection
- Rule engine
- Local video buffer
- Event creation
- Event sending
- Heartbeat sending

Central Backend responsibilities:

- Device registry
- Gateway registry
- Authentication
- Event ingestion
- Event storage
- Incident workflow
- Dashboard API
- Alert workflow
- Evidence management
- Report generation
- Audit logs

### DeviceEvent Is Not Incident

Never treat machine detection as confirmed truth.

`DeviceEvent` means:

- Something was detected
- It may require review
- It may be ignored
- It may later become part of an incident

`Incident` means:

- Verified operational case
- Human-reviewed or strongly confirmed
- Has workflow status
- Can have evidence and official report

## 17. MVP Implementation Defaults

Recommended defaults for the first implementation:

- Django + Django REST Framework
- PostgreSQL
- Separate `external_id` fields for gateways and devices
- Gateway token authentication for ingestion and heartbeat
- Django admin for internal management
- DRF filters for event list
- JSON metadata fields for flexible device/event information
- Local media URLs or placeholder URLs
- No PostGIS in MVP
- No WebSocket in MVP
- No AI in MVP

## 18. Acceptance Criteria for Backend MVP

The first backend version is successful when:

- The system can store sectors, gateways, devices, and events
- A gateway can authenticate and send heartbeat
- A gateway can authenticate and send a device event
- Invalid gateway/device/event payloads are rejected
- The backend confirms the device belongs to the gateway
- Events are saved with status `new`
- Events can be listed, filtered, and viewed
- Admin can inspect core records
- A fake gateway script can continuously produce test traffic

## 19. Notes for Future Engineers

This project should be built as a professional backend system, not as a random feature collection.

When adding features, preserve these boundaries:

- Devices produce signals
- Edge gateways process locally
- The central backend stores and coordinates
- Operators verify
- Incidents are official workflow records
- Evidence supports incidents
- Sensitive data stays private and access-controlled

The long-term quality of the system depends more on clean boundaries, validation, auditability, and correct workflow than on early AI complexity.
# Border Common Operating Picture System - BCOP Lite

## 1. Project Purpose

BCOP Lite is a backend-first command and incident management system for authorized border-security operators.

The system's purpose is to integrate signals from existing hardware, edge processing nodes, and field/operator inputs into one central operational picture.

The main goal is:

> Convert raw device data into verified operational incidents.

Core mental model:

- Device = source
- Edge Gateway = local processor
- DeviceEvent = detected signal
- Incident = verified operational case
- Evidence = supporting media/document
- Dashboard = operational view
- Report = official record

BCOP Lite is not a public alert app, public border map, vigilante tool, facial recognition system, or drone-control platform.

## 2. System Scope

### MVP Scope

The first backend milestone should build the event pipeline:

1. Device registry
2. Gateway registry
3. Heartbeat API
4. Event ingestion API
5. Event storage
6. Event list/detail API
7. Admin support
8. Fake gateway simulator

The MVP should not build AI, radar processing, drone control, WebSockets, PostGIS, incident workflow, or advanced dashboard features yet.

### Future Scope

After the pipeline works, add:

- Device health history
- Incident workflow
- Incident evidence
- Operator roles and permissions
- Audit logs
- Realtime dashboard alerts
- Map dashboard
- AI edge service
- PostGIS geospatial queries
- MinIO or S3-compatible media storage

## 3. High-Level Architecture

### Layer 1: Device Registry

The central backend stores all hardware and field-input source records.

Supported device types:

- CCTV
- PTZ
- THERMAL
- RADAR
- DRONE
- ACOUSTIC
- FIELD_REPORT_APP

The backend needs to know:

- Which devices exist
- Where each device is installed
- Which border sector each device belongs to
- Which edge gateway handles each device
- Whether each device is active, inactive, offline, or in error state
- Which protocol each device uses
- Stream URL or API details where applicable

### Layer 2: Edge Gateway

An Edge Gateway is a local processing node near devices.

It may be:

- Mini PC
- Local server
- Rugged edge box
- NVR-adjacent machine
- GPU/CPU processing box

The Edge Gateway is responsible for:

- Reading camera streams locally
- Decoding video frames
- Running AI or motion detection locally
- Applying local rules
- Keeping a short rolling video buffer
- Creating normalized events
- Sending only important events to the central backend
- Sending heartbeat and health status to the backend

The central backend should not receive or process all raw camera footage continuously.

Correct data flow:

```text
Camera/Device -> Local Edge Gateway -> Event/Clip/Metadata -> Central Backend
```

Reason:

One 1080p CCTV stream may use around 2-8 Mbps. At 4 Mbps:

- 4 Mbps = about 0.5 MB/s
- Per day = about 43 GB per camera
- 100 cameras = about 4.3 TB/day
- 30 days = about 129 TB/month

Centralizing all raw video is expensive, slow, bandwidth-heavy, and unreliable.

### Layer 3: Event Ingestion API

The Edge Gateway sends normalized events to the central backend.

Example event payload:

```json
{
  "gateway_id": "gw_benapole_01",
  "device_id": "cam_12",
  "event_type": "suspicious_group_movement",
  "confidence": 0.82,
  "severity": "high",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "occurred_at": "2026-06-12T02:10:00",
  "snapshot_url": "snapshot_123.jpg",
  "clip_url": "clip_123.mp4"
}
```

The backend validates:

- Gateway is authenticated
- Gateway exists
- Device exists
- Device belongs to the gateway
- Device and gateway belong to the expected sector
- Event type is valid
- Confidence is between 0.0 and 1.0
- Severity is valid
- Location is valid
- Timestamp is valid

After validation, the backend saves the event and makes it available to dashboard and alert workflows.

### Layer 4: Event Fusion and Incident Creation

A `DeviceEvent` is not an `Incident`.

DeviceEvent:

- Raw signal or detection
- May be wrong
- May be duplicated
- May need human review

Incident:

- Verified operational case
- Created by an operator or trusted rule engine
- Can contain multiple evidence records
- Can be assigned, resolved, or marked as false alarm

Example:

- Camera detects 6 people
- Radar detects movement nearby
- Thermal camera confirms heat signatures
- Patrol report arrives from same area

The system can combine these signals and raise confidence. An operator may then convert one or more events into an incident.

### Layer 5: Dashboard, Alert, and Workflow

The dashboard should eventually show:

- Map of sectors
- Devices
- Gateways
- Active alerts
- Device health
- Incident timeline
- Evidence clips
- Operator actions
- Event-to-incident workflow

Future workflow:

```text
new event
-> reviewing
-> ignored or converted_to_incident
-> incident open
-> assigned
-> responding
-> resolved or false_alarm
-> report generated
```

## 4. Recommended Tech Stack

Backend:

- Django
- Django REST Framework
- PostgreSQL
- PostGIS later for geospatial queries
- Redis later
- Celery later
- Django Channels or WebSocket later

Media storage:

- Local storage for MVP
- MinIO or S3-compatible storage later

AI/Edge service:

- Python service
- OpenCV
- FFmpeg
- FastAPI optional
- YOLO or RT-DETR later
- Fake event generator for MVP

Frontend/dashboard:

- React/Next.js or similar
- Leaflet or MapLibre for map

Deployment:

- On-premise or private cloud preferred
- Public SaaS should not be assumed for sensitive border data

## 5. Core Backend Models

### 5.1 BorderSector

Represents a logical border area.

Fields:

- `id`
- `name`
- `district`
- `upazila`
- `description`
- `risk_level`
- `boundary_polygon`, optional later with GIS/PostGIS
- `created_at`
- `updated_at`

Suggested risk levels:

- `low`
- `medium`
- `high`
- `critical`

Relationships:

- One `BorderSector` has many `EdgeGateways`
- One `BorderSector` has many `Devices`
- One `BorderSector` has many `DeviceEvents`
- One `BorderSector` has many `Incidents`

### 5.2 EdgeGateway

Represents a local processing node.

Fields:

- `id`
- `external_id`, for example `gw_benapole_01`
- `sector_id`
- `name`
- `location_lat`
- `location_lng`
- `ip_address`
- `status`
- `last_heartbeat_at`
- `auth_token_hash`
- `metadata`
- `created_at`
- `updated_at`

Suggested statuses:

- `online`
- `offline`
- `error`
- `maintenance`

Relationships:

- One `EdgeGateway` belongs to one `BorderSector`
- One `EdgeGateway` has many `Devices`
- One `EdgeGateway` sends many `DeviceEvents`
- One `EdgeGateway` sends many `DeviceHealthLogs` later

Important design decision:

Use a database primary key internally, but expose `external_id` for gateway communication. This allows user-friendly IDs like `gw_01` without locking database structure to external naming.

### 5.3 Device

Represents CCTV, PTZ, thermal, radar, drone, acoustic sensor, or field report source.

Fields:

- `id`
- `external_id`, for example `cam_01`
- `gateway_id`
- `sector_id`
- `name`
- `type`
- `protocol`
- `stream_url`
- `location_lat`
- `location_lng`
- `status`
- `installed_at`
- `metadata`
- `created_at`
- `updated_at`

Suggested device types:

- `CCTV`
- `PTZ`
- `THERMAL`
- `RADAR`
- `DRONE`
- `ACOUSTIC`
- `FIELD_REPORT_APP`

Suggested protocols:

- `RTSP`
- `ONVIF`
- `API`
- `MANUAL`
- `UPLOAD`

Suggested statuses:

- `active`
- `inactive`
- `offline`
- `error`

Relationships:

- One `Device` belongs to one `EdgeGateway`
- One `Device` belongs to one `BorderSector`
- One `Device` creates many `DeviceEvents`
- One `Device` may provide many `IncidentEvidence` records later

Validation rule:

The device sector should match the gateway sector unless there is a specific operational reason to allow otherwise.

### 5.4 DeviceEvent

Represents a machine or system detected signal.

This is the most important MVP model.

Fields:

- `id`
- `device_id`
- `gateway_id`
- `sector_id`
- `event_type`
- `confidence`
- `severity`
- `location_lat`
- `location_lng`
- `occurred_at`
- `status`
- `metadata`
- `snapshot_url`
- `clip_url`
- `created_at`

Suggested event types:

- `person_detected`
- `group_movement`
- `zone_crossing`
- `boat_detected`
- `vehicle_detected`
- `possible_gunshot`
- `radar_movement`
- `suspicious_activity`

Suggested severities:

- `low`
- `medium`
- `high`
- `critical`

Suggested event statuses:

- `new`
- `reviewing`
- `ignored`
- `converted_to_incident`

Validation rules:

- `confidence` must be between `0.0` and `1.0`
- `occurred_at` must be a valid timestamp
- `device_id` must identify an existing device
- `gateway_id` must identify an existing gateway
- The device must belong to the gateway
- The event sector should be derived from the device/gateway instead of blindly trusted from request payload

Important:

`DeviceEvent` is not confirmed truth. It is a detected signal.

### 5.5 DeviceHealthLog

Not required in the first milestone, but should be added soon after.

Tracks device and gateway health.

Fields:

- `id`
- `device_id`, nullable
- `gateway_id`
- `status`
- `cpu_usage`
- `memory_usage`
- `disk_usage`
- `network_status`
- `message`
- `reported_at`
- `created_at`

Purpose:

The command center must know whether cameras and gateways are actually working.

### 5.6 Incident

Not required in the first milestone.

Represents a verified operational case created by an operator or trusted rule engine.

Fields:

- `id`
- `sector_id`
- `title`
- `incident_type`
- `severity`
- `status`
- `confidence_score`
- `location_lat`
- `location_lng`
- `started_at`
- `resolved_at`
- `created_from_event_id`, nullable
- `assigned_to`, nullable
- `summary`
- `created_by`
- `created_at`
- `updated_at`

Suggested incident types:

- `push_in_attempt`
- `firing`
- `suspicious_crossing`
- `smuggling`
- `injured_person`
- `river_incident`
- `crowd_gathering`
- `unknown`

Suggested incident statuses:

- `open`
- `assigned`
- `responding`
- `resolved`
- `false_alarm`

Important:

An `Incident` should be created only after human verification or strong multi-source confidence.

### 5.7 IncidentEvidence

Not required in the first milestone.

Represents evidence attached to an incident.

Fields:

- `id`
- `incident_id`
- `evidence_type`
- `file_url`
- `source_type`
- `device_id`, nullable
- `uploaded_by`, nullable
- `captured_at`
- `metadata`
- `created_at`

Suggested evidence types:

- `image`
- `video`
- `audio`
- `document`
- `note`

Suggested source types:

- `camera`
- `drone`
- `patrol_app`
- `operator_upload`
- `radar`
- `acoustic`

### 5.8 User / Operator

Use Django's user system or a custom user model.

Roles:

- `admin`
- `operator`
- `verifier`
- `patrol_officer`
- `viewer`

Suggested fields:

- `id`
- `name`
- `email`
- `phone`
- `role`
- `assigned_sector`
- `is_active`
- `created_at`

Access control is important because this is an authorized-use system.

## 6. MVP API Design

Base path:

```text
/api/
```

### 6.1 Border Sector APIs

```text
GET /api/sectors/
POST /api/sectors/
GET /api/sectors/{id}/
PATCH /api/sectors/{id}/
DELETE /api/sectors/{id}/
```

Purpose:

Create and manage border sectors.

Create payload:

```json
{
  "name": "Benapole Sector",
  "district": "Jashore",
  "upazila": "Sharsha",
  "description": "High activity border sector",
  "risk_level": "high"
}
```

### 6.2 Gateway APIs

```text
GET /api/gateways/
POST /api/gateways/
GET /api/gateways/{id}/
PATCH /api/gateways/{id}/
DELETE /api/gateways/{id}/
POST /api/gateways/{id}/heartbeat/
```

Purpose:

Create and manage edge gateways. Receive heartbeat updates.

Create payload:

```json
{
  "external_id": "gw_benapole_01",
  "sector": "sector_uuid_or_id",
  "name": "Benapole Gateway 01",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "ip_address": "192.168.1.20",
  "status": "offline",
  "metadata": {
    "hardware": "mini_pc",
    "gpu": "none"
  }
}
```

Heartbeat payload:

```json
{
  "status": "online",
  "cpu_usage": 62,
  "memory_usage": 71,
  "disk_usage": 55,
  "devices": [
    {
      "device_id": "cam_01",
      "status": "online"
    },
    {
      "device_id": "cam_02",
      "status": "offline"
    }
  ]
}
```

Heartbeat behavior:

- Authenticate gateway
- Update gateway status
- Update `last_heartbeat_at`
- Optionally update linked device statuses
- Later, create `DeviceHealthLog` records

### 6.3 Device APIs

```text
GET /api/devices/
POST /api/devices/
GET /api/devices/{id}/
PATCH /api/devices/{id}/
DELETE /api/devices/{id}/
```

Purpose:

Create and manage devices.

Create payload:

```json
{
  "external_id": "cam_01",
  "gateway": "gateway_uuid_or_id",
  "sector": "sector_uuid_or_id",
  "name": "North Fence Camera 01",
  "type": "CCTV",
  "protocol": "RTSP",
  "stream_url": "rtsp://example.local/stream1",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "status": "active",
  "installed_at": "2026-06-12T10:00:00",
  "metadata": {
    "resolution": "1080p",
    "night_vision": true
  }
}
```

Validation behavior:

- Device external ID should be unique per system or unique per gateway
- Gateway must exist
- Sector must exist
- Device sector should match gateway sector
- Protocol must be valid for the selected device type where possible

### 6.4 Event Ingestion API

```text
POST /api/events/
```

Purpose:

Receive normalized events from edge gateways.

Payload:

```json
{
  "gateway_id": "gw_01",
  "device_id": "cam_01",
  "event_type": "group_movement",
  "confidence": 0.82,
  "severity": "high",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "occurred_at": "2026-06-12T03:10:20",
  "snapshot_url": "snapshot.jpg",
  "clip_url": "clip.mp4",
  "metadata": {
    "object_count": 6,
    "direction": "towards_border",
    "time_context": "night"
  }
}
```

Expected behavior:

- Authenticate gateway
- Look up gateway by `gateway_id`
- Look up device by `device_id`
- Confirm device belongs to gateway
- Derive sector from device/gateway
- Validate event type
- Validate confidence
- Validate severity
- Save event with status `new`
- Return created event response

Suggested success response:

```json
{
  "id": "event_uuid_or_id",
  "status": "new",
  "message": "Event accepted"
}
```

### 6.5 Event List APIs

```text
GET /api/events/
GET /api/events/{id}/
PATCH /api/events/{id}/
```

Purpose:

Allow authorized operators to view and review device events.

Filters:

- `sector`
- `device`
- `gateway`
- `severity`
- `status`
- `event_type`
- `occurred_at_after`
- `occurred_at_before`

Example:

```text
GET /api/events/?severity=high&status=new&event_type=group_movement
```

Review behavior:

- Operators may change event status from `new` to `reviewing`
- Operators may mark event as `ignored`
- Later, operators may convert event to incident

## 7. Serializer and Validation Design

### BorderSectorSerializer

Responsible for:

- Validating sector details
- Returning sector fields for list/detail APIs

### EdgeGatewaySerializer

Responsible for:

- Validating gateway setup
- Showing sector relationship
- Keeping gateway metadata flexible

### DeviceSerializer

Responsible for:

- Validating device setup
- Ensuring gateway and sector consistency
- Supporting optional stream URL

### DeviceEventSerializer

Responsible for:

- Returning event details to operators
- Supporting event review status changes

### EventIngestionSerializer

Separate from the normal `DeviceEventSerializer`.

Responsible for:

- Accepting external IDs from gateways
- Looking up real gateway and device records
- Validating gateway-device relationship
- Deriving sector
- Creating a valid `DeviceEvent`

This separation keeps external ingestion concerns away from internal admin/dashboard representation.

### GatewayHeartbeatSerializer

Responsible for:

- Validating gateway status
- Validating optional resource usage values
- Validating device status updates
- Updating gateway and device status records

## 8. View and API Structure

Recommended DRF structure:

- `BorderSectorViewSet`
- `EdgeGatewayViewSet`
- `DeviceViewSet`
- `DeviceEventViewSet`
- Custom action: `EdgeGatewayViewSet.heartbeat`

Recommended API routing:

```text
/api/sectors/
/api/gateways/
/api/gateways/{id}/heartbeat/
/api/devices/
/api/events/
```

The event ingestion endpoint may use the same `/api/events/` POST route, but with ingestion-specific validation.

Alternative:

```text
POST /api/ingest/events/
```

For MVP, using `POST /api/events/` is acceptable if permissions and serializer selection are clear.

## 9. Authentication and Authorization

### MVP Authentication

Use two access paths:

1. Admin/operator access through Django admin or normal authenticated users
2. Gateway access through gateway token authentication

Gateway requests should include a token header, for example:

```text
Authorization: GatewayToken <token>
```

or:

```text
X-Gateway-Token: <token>
```

Recommended MVP behavior:

- Store only a hash of the gateway token
- Show the raw token only once when generated
- Require token for heartbeat
- Require token for event ingestion
- Ensure the token belongs to the gateway submitting the event

### Future Authorization

Add role-based access control:

- Admin can manage sectors, gateways, devices, users
- Operator can review events and manage incidents
- Verifier can verify incidents
- Patrol officer can submit reports/evidence
- Viewer can only read permitted dashboard data

### Audit Logging

All important actions should eventually be logged:

- Gateway created
- Device created
- Heartbeat received
- Event received
- Event status changed
- Incident created
- Incident assigned
- Incident resolved
- Evidence uploaded
- User login or permission-sensitive action

## 10. Security and Safety Rules

Do not build:

- Public border live map
- Public alert system
- Public patrol tracking
- Face recognition
- Nationality detection
- "Enemy detected" feature
- Targeting or vigilante features
- Unauthorized drone control

Required security direction:

- Authorized operators only
- Authenticated gateways only
- Role-based access control
- Audit logs
- Gateway token authentication
- Event validation
- Media access control
- Private/on-premise deployment preference

## 11. Fake Gateway Simulator

The fake gateway simulator is part of the MVP because the pipeline should be tested before AI is added.

Simulator responsibilities:

1. Use an existing gateway external ID, for example `gw_01`
2. Send heartbeat every 10 seconds
3. Simulate device statuses
4. Send fake `DeviceEvent` every 30-60 seconds
5. Print API responses and errors to the console

Example fake heartbeat:

```json
{
  "status": "online",
  "cpu_usage": 62,
  "memory_usage": 71,
  "disk_usage": 55,
  "devices": [
    {
      "device_id": "cam_01",
      "status": "online"
    }
  ]
}
```

Example fake event:

```json
{
  "gateway_id": "gw_01",
  "device_id": "cam_01",
  "event_type": "group_movement",
  "confidence": 0.82,
  "severity": "high",
  "location_lat": 23.123,
  "location_lng": 88.456,
  "occurred_at": "2026-06-12T03:10:20",
  "snapshot_url": "snapshot.jpg",
  "clip_url": "clip.mp4",
  "metadata": {
    "object_count": 6,
    "direction": "towards_border",
    "time_context": "night"
  }
}
```

Later, this simulator can be replaced with:

- RTSP stream reader
- Frame extractor
- AI detector
- Local rule engine
- Event sender

## 12. Admin Support

The Django admin should allow authorized admins to manage:

- Border sectors
- Edge gateways
- Devices
- Device events

Admin list pages should show useful operational fields:

Border sectors:

- Name
- District
- Upazila
- Risk level

Gateways:

- External ID
- Name
- Sector
- Status
- Last heartbeat

Devices:

- External ID
- Name
- Type
- Gateway
- Sector
- Status

Device events:

- Event type
- Severity
- Confidence
- Status
- Device
- Gateway
- Sector
- Occurred at

## 13. Filtering and Query Behavior

Event list filters should support:

- Sector
- Device
- Gateway
- Severity
- Status
- Event type
- Date range

Default event ordering:

```text
occurred_at descending
```

Recommended dashboard query:

```text
GET /api/events/?status=new&severity=high
```

Recommended sector query:

```text
GET /api/events/?sector={sector_id}
```

Recommended date range query:

```text
GET /api/events/?occurred_at_after=2026-06-12T00:00:00&occurred_at_before=2026-06-13T00:00:00
```

## 14. Data Lifecycle

MVP:

- Events are stored in PostgreSQL
- Snapshot and clip URLs are stored as strings
- Actual media can be local files or placeholder URLs

Future:

- Store media in MinIO or S3-compatible storage
- Add retention policies
- Add evidence immutability rules for official incidents
- Add report generation
- Add audit trail for evidence access

## 15. Development Milestones

### Milestone 1: Backend Pipeline MVP

Build:

- Django project
- DRF setup
- PostgreSQL configuration
- `BorderSector` model
- `EdgeGateway` model
- `Device` model
- `DeviceEvent` model
- Serializers
- ViewSets
- API routes
- Heartbeat endpoint
- Event ingestion endpoint
- Event filtering
- Django admin registration
- Fake gateway simulator

Success criteria:

- Admin can create a sector
- Admin can create a gateway
- Admin can create a device
- Fake gateway can send heartbeat
- Fake gateway can send fake events
- Backend saves events
- Operator can list and filter events through API

### Milestone 2: Health and Incident Workflow

Build:

- `DeviceHealthLog`
- `Incident`
- `IncidentEvidence`
- Event-to-incident conversion
- Incident status workflow
- Basic role-based permissions

Success criteria:

- Heartbeat history is stored
- Operator can convert event to incident
- Incident can be assigned and resolved
- Evidence can be attached to incident

### Milestone 3: Realtime Dashboard

Build:

- WebSocket or Django Channels
- Realtime new-event notifications
- Gateway/device health updates
- Dashboard-ready summary APIs

Success criteria:

- Dashboard can receive new event alerts without polling
- Operators can see active sector/device health

### Milestone 4: Edge AI Integration

Build:

- RTSP reader
- Frame extractor
- AI detector
- Rule engine
- Local video buffer
- Clip extraction
- Event sender

Success criteria:

- Edge service watches real or test video stream
- Edge service sends real detection events
- Backend receives events using the same ingestion API

## 16. Important Design Principles

### Build the Pipeline Before AI

Do not build AI first.

Correct order:

```text
Device Registry
-> Gateway Registry
-> Heartbeat API
-> Event Ingestion API
-> Event Store
-> Event List API
-> Dashboard-ready data
-> AI and intelligence
```

### Keep Edge and Central Responsibilities Separate

Edge Gateway responsibilities:

- Raw stream reading
- Local processing
- AI/motion detection
- Rule engine
- Local video buffer
- Event creation
- Event sending
- Heartbeat sending

Central Backend responsibilities:

- Device registry
- Gateway registry
- Authentication
- Event ingestion
- Event storage
- Incident workflow
- Dashboard API
- Alert workflow
- Evidence management
- Report generation
- Audit logs

### DeviceEvent Is Not Incident

Never treat machine detection as confirmed truth.

`DeviceEvent` means:

- Something was detected
- It may require review
- It may be ignored
- It may later become part of an incident

`Incident` means:

- Verified operational case
- Human-reviewed or strongly confirmed
- Has workflow status
- Can have evidence and official report

## 17. MVP Implementation Defaults

Recommended defaults for the first implementation:

- Django + Django REST Framework
- PostgreSQL
- Separate `external_id` fields for gateways and devices
- Gateway token authentication for ingestion and heartbeat
- Django admin for internal management
- DRF filters for event list
- JSON metadata fields for flexible device/event information
- Local media URLs or placeholder URLs
- No PostGIS in MVP
- No WebSocket in MVP
- No AI in MVP

## 18. Acceptance Criteria for Backend MVP

The first backend version is successful when:

- The system can store sectors, gateways, devices, and events
- A gateway can authenticate and send heartbeat
- A gateway can authenticate and send a device event
- Invalid gateway/device/event payloads are rejected
- The backend confirms the device belongs to the gateway
- Events are saved with status `new`
- Events can be listed, filtered, and viewed
- Admin can inspect core records
- A fake gateway script can continuously produce test traffic

## 19. Notes for Future Engineers

This project should be built as a professional backend system, not as a random feature collection.

When adding features, preserve these boundaries:

- Devices produce signals
- Edge gateways process locally
- The central backend stores and coordinates
- Operators verify
- Incidents are official workflow records
- Evidence supports incidents
- Sensitive data stays private and access-controlled

The long-term quality of the system depends more on clean boundaries, validation, auditability, and correct workflow than on early AI complexity.
