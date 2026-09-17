# STVE - Sensor Trust Verification Engine

**Sensor Trust Verification Engine for Precision Agriculture**

## Quick Links

- **Quick start**: `QUICK_START.md`
- **Backend**: `backend/`
- **Frontend dashboard**: `frontend/`
- **License**: `LICENSE`

---

## Problem Statement

Large-scale precision agriculture deployments rely on networks of thousands of soil sensors to measure critical parameters like moisture, temperature, and electrical conductivity. These measurements drive automated irrigation and fertilization systems that optimize resource usage and crop yields.

However, sensor networks suffer from **silent failures**:
- Sensors can become stuck, reporting unchanged values despite environmental changes
- Physical damage or calibration drift can cause sudden spikes or anomalous readings
- Faulty sensors distort the data fed into automation systems
- **Even a 5% sensor failure rate** can lead to incorrect irrigation decisions, causing water waste, nutrient imbalance, and reduced crop productivity

The fundamental challenge is that **automation systems trust all sensor data equally**, lacking the ability to distinguish reliable readings from faulty ones.

---

## Solution Overview

**STVE** introduces a **Trust Layer** between raw sensor data and decision-making systems. Rather than blindly trusting all sensors, STVE continuously validates sensor behavior and assigns a dynamic **Trust Score** to each device.

### Key Innovation

STVE acts as a **reliability firewall**:
1. Sensors send readings → STVE ingests and analyzes them
2. Trust Engine evaluates each sensor's behavior using statistical methods
3. Faulty sensors are flagged and isolated **before** they influence decisions
4. Maintenance tickets are automatically generated for anomalous sensors
5. Only trusted sensor data flows to automation systems

This approach prevents faulty sensors from corrupting irrigation decisions, reducing resource waste and improving operational reliability.

---

## Core Modules Explanation

### 1. Sensor Registry Module

**Purpose**: Centralized management of sensor metadata and network topology.

**Functionality**:
- Registers new sensors with unique IDs, zone assignments, and geographic coordinates
- Maintains sensor metadata (type, installation date, location)
- Initializes each sensor with a baseline Trust Score of 100
- Provides lookup capabilities for sensor information retrieval

**Why It Matters**: Knowing which sensors belong to which zones enables cross-sensor validation and zone-level anomaly detection.

---

### 2. Data Ingestion Module

**Purpose**: Accept and store real-time sensor readings.

**Functionality**:
- Receives sensor readings via REST API (moisture, temperature, EC)
- Timestamps each reading for temporal analysis
- Stores readings in the database for historical tracking
- Triggers Trust Engine evaluation after each new reading

**Design Choice**: Immediate trust evaluation ensures rapid detection of sensor failures, preventing bad data from accumulating.

---

### 3. Trust Engine Module

**Purpose**: Core intelligence that evaluates sensor reliability.

**What it does (v2)**: STVE uses a diagnostic, confidence-based approach instead of a fixed pass/fail rule set.

**Key features**:
- Computes **parameter-level trust** (moisture / temperature / EC / pH) to isolate which probe is faulty
- Separates checks into:
  - **Temporal stability** (SPIKE / STATIC / DRIFT)
  - **Cross-sensor agreement** within a zone (ZONE_MISMATCH vs FIELD_EVENT)
  - **Physical plausibility** (IMPOSSIBLE_VALUE / WEATHER_MISMATCH)
- Handles **missing data / connectivity** and marks sensors **SENSOR_OFFLINE**
- Tracks **health trend** over recent trust scores (stable / improving / degrading)
- Produces **root cause codes**, **severity**, and a **human-readable diagnostic**

**Trust scoring (high level)**:
- Each parameter gets a trust score `T_param` in the range **0.0–1.0**
- Final sensor trust is the average across parameters
- Status/labels are derived from bands (Healthy / Warning / Anomalous)

**Auto-ticket generation**:
- A ticket is created when status is **Anomalous**
- Tickets are **not** created for **FIELD_EVENT** (interpreted as a real-world change affecting the whole zone)

---

### 4. Maintenance Automation Module

**Purpose**: Manage maintenance workflows for faulty sensors.

**Functionality**:
- Auto-generates tickets when sensors enter Anomalous state
- Prevents duplicate tickets for the same sensor
- Categorizes issues by severity (Critical, High, Medium, Low)
- Tracks ticket lifecycle: Open → InProgress → Resolved
- Provides filtering and querying capabilities for maintenance teams

**Value**: Converts anomaly detection into actionable maintenance tasks, closing the loop between detection and repair.

---

### 5. Dashboard & Visualization Module

**Purpose**: Real-time monitoring and system health overview.

**Features**:
- **Summary Cards**: Total sensors, Healthy/Warning/Anomalous counts
- **Sensor Table**: Live view of all sensors with current trust scores and readings
- **Trust Score Distribution Chart**: Visual breakdown of sensor health across the network
- **Maintenance Page**: Centralized list of all open, in-progress, and resolved tickets

**Design Philosophy**: At-a-glance visibility into system health enables proactive management and rapid issue identification.

---

## Architecture Overview

```
┌─────────────┐
│   Sensors   │  (Soil moisture, temperature, EC sensors)
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Backend API    │  (Express.js REST endpoints)
│  /api/readings  │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│  Trust Engine   │  (Evaluates sensor behavior)
│  - Temporal     │
│  - Cross-zone   │
│  - Physical     │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│   Database      │  (PostgreSQL + Prisma ORM)
│  - Sensors      │
│  - Readings     │
│  - TrustScores  │
│  - Tickets      │
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│   Dashboard     │  (Next.js frontend)
│  - Live metrics │
│  - Trust charts │
│  - Tickets view │
└─────────────────┘
```

**Data Flow**:
1. Sensors push readings to `/api/readings` endpoint
2. Backend stores reading in database
3. Trust Engine evaluates the sensor using recent history + zone neighbours
4. A confidence-based trust score (0.0–1.0), root causes, severity, and diagnostic are stored
5. If status is Anomalous, a maintenance ticket is generated (except for `FIELD_EVENT`)
6. Frontend polls `/api/dashboard/summary` and `/api/sensors` for updates
7. Operators view sensor health and maintenance tasks in real-time

---

## Trust Score Logic

The backend trust engine (`backend/src/modules/trustEngine.js`) computes a **confidence score** instead of applying fixed penalties.

### 1) Parameter trust (0.0–1.0)

For each parameter (`moisture`, `temperature`, `ec`, `ph`):

- **Temporal score**: detects
  - `STATIC` (frozen probe via low range across recent history)
  - `DRIFT` (slow trend via linear regression slope)
  - `SPIKE` (large change vs rolling mean)
- **Cross score**: compares against zone neighbours
  - `ZONE_MISMATCH` when this sensor deviates but neighbours are stable
  - `FIELD_EVENT` when neighbours also changed (interpreted as real conditions)
- **Physical score**: checks plausibility / context
  - `IMPOSSIBLE_VALUE` for out-of-bounds values
  - `WEATHER_MISMATCH` for context mismatches (e.g., very high moisture with no rain/irrigation)

The three components are combined with weights:
```
T_param = 0.3 * Temporal + 0.5 * Cross + 0.2 * Physical
```

### 2) Sensor trust (0.0–1.0)

The overall sensor trust is the average across parameters:
```
SensorTrust = (T_moisture + T_temperature + T_ec + T_ph) / 4
```

### 3) Status mapping

The trust engine interprets the final score into:

- **Healthy**: Highly Reliable / Reliable
- **Warning**: Uncertain
- **Anomalous**: Unreliable / Anomaly

### 4) Offline handling

If a sensor is not transmitting recent data (timestamp too old) or has too many null readings, it is marked:

- `SENSOR_OFFLINE`
- Trust score is stored as `0.0`
- Severity is `Critical`

### 5) Root causes, severity, and health trend

- **Root causes** are aggregated across temporal/cross/physical checks (e.g., `SPIKE`, `STATIC`, `DRIFT`, `ZONE_MISMATCH`, `WEATHER_MISMATCH`, `IMPOSSIBLE_VALUE`, `SENSOR_OFFLINE`)
- **Severity** is derived from root causes and trust score (Critical / High / Medium / Low / None)
- **Health trend** is computed from recent trust history (stable / improving / degrading) to spot gradual degradation

### Re-Evaluation
- Trust score is recalculated **on every new reading**
- Sensors can recover if behavior normalizes
- Historical trust scores are preserved for trend analysis

---

## How to Run

If you want the fastest path to a working demo, follow `QUICK_START.md`.

### Prerequisites
- **Node.js** (v18 or higher)
- **PostgreSQL** (v14 or higher)
- **pnpm** (or npm)

---

### Backend Setup

1. **Navigate to backend folder**:
   ```bash
   cd backend
   ```

2. **Install dependencies**:
   ```bash
   npm install
   ```

3. **Configure database**:
   - Create `backend/.env` and set:
     ```
     DATABASE_URL="postgresql://username:password@localhost:5432/STVE"
     ```

4. **Run migrations + generate Prisma client**:
   ```bash
   npx prisma migrate dev --name init
   npx prisma generate
   ```

5. **Start the backend server**:
   ```bash
   npm run dev
   ```

   The API will run on `http://localhost:5000`

### Backend Utility Scripts

Run these from `backend/`:

- **Seed sample data**: `npm run seed`
- **Upload dataset (if applicable)**: `npm run upload`
- **Verify ingested data**: `npm run verify`
- **Demo realtime mode**: `npm run realtime`
- **Continuous realtime demo**: `npm run realtime:continuous`
- **Reset realtime demo state**: `npm run realtime:reset`
- **Reset database**: `npm run db:reset`

---

### Frontend Setup

1. **Navigate to frontend folder**:
   ```bash
   cd frontend
   ```

2. **Install dependencies**:
   ```bash
   npm install
   ```

3. **Start the development server**:
   ```bash
   npm run dev
   ```

   The dashboard will be available at `http://localhost:3000`

### Frontend Configuration

If your frontend needs an explicit backend URL, set `frontend/.env.local`:
```
NEXT_PUBLIC_API_URL=http://localhost:5000
```

---

### Testing the System

1. **Register a sensor**:
   ```bash
   curl -X POST http://localhost:5000/api/sensors \
     -H "Content-Type: application/json" \
     -d '{
       "sensorId": "SENSOR-001",
       "zone": "Field-A",
       "type": "soil_moisture",
       "latitude": 37.7749,
       "longitude": -122.4194
     }'
   ```

2. **Send sensor readings**:
   ```bash
   curl -X POST http://localhost:5000/api/readings \
     -H "Content-Type: application/json" \
     -d '{
       "sensorId": "SENSOR-001",
       "moisture": 45.2,
       "temperature": 22.5,
       "ec": 1.2
     }'
   ```

3. **View dashboard**: Open `http://localhost:3000` to see sensor health and trust scores

4. **Check maintenance tickets**: Navigate to `http://localhost:3000/maintenance`

---

## API Endpoints

### Sensors
- `POST /api/sensors` - Register a new sensor
- `GET /api/sensors` - List all sensors with latest trust scores
- `GET /api/sensors/:id` - Get sensor details

### Readings
- `POST /api/readings` - Ingest a single sensor reading
- `POST /api/readings/batch` - Batch ingest multiple readings

### Dashboard
- `GET /api/dashboard/summary` - Get system overview (sensor counts, ticket stats)
- `GET /api/dashboard/zones` - Get zone-level statistics
- `GET /api/dashboard/activity` - Get recent activity feed

### Maintenance
- `GET /api/tickets` - List all maintenance tickets
- `GET /api/tickets?status=Open` - Filter tickets by status
- `PATCH /api/tickets/:id` - Update ticket status

---

## Project Structure

```
STVE/
│
├── backend/
│   ├── src/
│   │   ├── modules/
│   │   │   ├── sensorRegistry.js    # Sensor management
│   │   │   ├── dataIngestion.js     # Reading intake
│   │   │   ├── trustEngine.js       # Core trust logic
│   │   │   ├── maintenance.js       # Ticket management
│   │   │   └── dashboard.js         # Summary stats
│   │   └── server.js                # Express app
│   ├── prisma/
│   │   └── schema.prisma            # Database schema
│   ├── package.json
│   └── .env
│
├── frontend/
│   ├── app/
│   │   ├── components/
│   │   │   └── Navigation.tsx       # Nav bar
│   │   ├── maintenance/
│   │   │   └── page.tsx             # Maintenance page
│   │   ├── page.tsx                 # Dashboard page
│   │   ├── layout.tsx               # Root layout
│   │   └── globals.css
│   ├── package.json
│   └── .env.local
│
└── README.md
```

---

## Future Scope

### 1. Machine Learning-Based Anomaly Detection
- **Current**: Confidence-based diagnostic engine (temporal + cross-sensor + physical checks)
- **Future**: Train ML models on historical sensor data to detect subtle drift patterns
- **Benefit**: Catch complex failures that simple variance/spike checks miss

### 2. Predictive Maintenance
- **Current**: Reactive ticket generation when sensors fail
- **Future**: Predict sensor failures before they occur based on degradation trends
- **Benefit**: Preventive replacement instead of reactive repairs

### 3. Integration with Real Irrigation Controllers
- **Current**: Standalone trust verification system
- **Future**: Direct API integration with irrigation controllers (e.g., Netafim, Jain)
- **Benefit**: Automatically exclude untrusted sensors from irrigation decisions

### 4. Weather-Based Validation Layer
- **Current**: Cross-sensor zone validation only
- **Future**: Compare sensor readings against local weather data (rainfall, evapotranspiration)
- **Benefit**: Detect sensors that contradict known weather events (e.g., no moisture increase after heavy rain)

### 5. Mobile App for Field Technicians
- **Current**: Web dashboard only
- **Future**: Native mobile app with offline ticket management and GPS-based sensor navigation
- **Benefit**: Enable technicians to resolve issues in areas with poor connectivity

### 6. Multi-Crop Zone Calibration
- **Current**: Single variance/threshold configuration
- **Future**: Crop-specific trust parameters (e.g., rice paddies vs. orchards have different moisture patterns)
- **Benefit**: Reduce false positives in heterogeneous farming environments

### 7. Blockchain-Based Sensor Audit Trail
- **Current**: Database-stored trust scores
- **Future**: Immutable ledger of sensor trust history for compliance and insurance claims
- **Benefit**: Prove sensor reliability to auditors and stakeholders

---

## Technology Choices

| Component | Technology | Reasoning |
|-----------|-----------|-----------|
| Backend | Node.js + Express | Fast prototyping, large ecosystem, non-blocking I/O for sensor streams |
| Database | PostgreSQL | ACID compliance for critical trust data, strong JSON support |
| ORM | Prisma | Type-safe database access, excellent migration tooling |
| Frontend | Next.js (App Router) | React framework with built-in routing, SSR capabilities |
| Styling | Tailwind CSS | Rapid UI development, utility-first approach |
| Charts | Recharts | React-native charting library, easy integration |

---

## Key Insights

1. **Trust is Dynamic**: Sensors don't fail permanently; they can recover. Continuous re-evaluation allows sensors to regain trust.

2. **Zone Context Matters**: A sensor reading that's anomalous in isolation might be normal for its zone. Cross-sensor comparison is critical.

3. **Prevention > Reaction**: Catching faulty sensors early prevents cascading errors in automation systems.

4. **Automation Needs Verification**: The shift from manual to automated agriculture demands robust data validation layers like STVE.

---

## License

MIT License - See LICENSE file for details

---

## Contributors

Built during a hackathon to demonstrate real-world IoT trust verification concepts.

---

## Contact

For questions or collaboration opportunities, please open an issue in this repository.

---

**STVE**: Because in precision agriculture, data you can trust is the foundation of decisions that matter.
