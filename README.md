# NeuroFleet-Guardian (NFG)

                  **Preventative Fleet Safety & Intelligence Platform**
                  
NeuroFleet-Guardian is a smart fleet safety and intelligence platform built to protect long-haul trucking operations by strengthening driver well-being, vehicle security, and real-time incident response through data-driven monitoring and analytics

> **IMPORTANT**: This is a demonstration MVP using **100% synthetic data**. No real biometrics, no real people. This is NOT a surveillance system — it's a preventative safety system designed with driver dignity, transparency, and consent first.

---

## Features

### Core Capabilities

- **Fatigue Prediction**: Continuous risk scoring (0-1) with 10-20 minute prediction horizon
- **Critical Event Detection**:
  - Microsleep episodes
  - Possible accidents/collisions
  - Abnormal or forced stops
  - Hijacking/coercion patterns
- **Fleet Intelligence**: Route-based danger sharing and automatic reroute suggestions
- **Automated Responses**:
  - Micro-break recommendations
  - Pull-over alerts
  - Driver swap coordination
  - Emergency escalation
- **Real-time Monitoring**: WebSocket-based live dashboard with map view

---

## Architecture

```
┌──────────────┐
│  Simulators  │──MQTT──┐
│  (20 trucks) │        │
└──────────────┘        ▼
                 ┌──────────────┐
                 │ MQTT Broker  │
                 └──────┬───────┘
                        │
            ┌───────────┴──────────┐
            ▼                      ▼
    ┌──────────────┐      ┌──────────────┐
    │ ML Service   │◄─────│ Orchestrator │
    │ (PyTorch)    │      │ (FastAPI)    │
    └──────────────┘      └──────┬───────┘
                                 │ WebSocket
                                 ▼
                          ┌──────────────┐
                          │  Dashboard   │
                          │  (React)     │
                          └──────────────┘
```

See [docs/architecture.md](docs/architecture.md) for detailed flow.

---

## Quick Start

### Prerequisites

- Docker & Docker Compose
- 8GB RAM minimum
- 4 CPU cores recommended
- Ports 1883, 3000, 8000, 8001 available

### Installation

```bash
# Clone repository
git clone https://github.com/your-org/neurofleet-guardian.git
cd neurofleet-guardian

# Build all services
make build

# Start the platform
make up

# Wait ~30 seconds for services to initialize
```

### Access Points

- **Dashboard**: http://localhost:3000
- **Fleet API**: http://localhost:8000/docs
- **ML API**: http://localhost:8001/docs
- **MQTT Broker**: mqtt://localhost:1883

### Run Simulation

In a new terminal:

```bash
# Run 20 trucks with event injection
make simulate

# Or manually:
python simulator/simulator.py \
  --number_of_trucks 20 \
  --frequency_hz 1 \
  --inject_events fatigue,microsleep,hijack \
  --output_mode mqtt
```

Within 30-60 seconds, you should see:
- Trucks appearing on the dashboard map
- Fatigue risk scores updating in real-time
- Alerts generated for high-risk trucks
- Event flags for injected incidents

---

## Training the Model

The ML service works with a pre-initialized model, but you can train it with synthetic data:

```bash
# Generate synthetic training dataset
cd ml-service
python dataset_generator.py --samples 100000 --output data/training.parquet

# Train model
python train.py --data data/training.parquet --epochs 20 --batch_size 64

# Model saved to: models/fatigue_model.pth
```

### Training Metrics

The training script outputs:
- AUC-ROC for fatigue, microsleep, anomaly
- Precision/Recall/F1
- Confusion matrices
- Calibration plots

---

## Testing

```bash
# Run all tests
make test

# Individual test suites
pytest simulator/tests/ -v
pytest ml-service/tests/ -v
pytest fleet-orchestrator/tests/ -v

# Integration test
make test-integration
```

---

## Project Structure

```
neurofleet-guardian/
├── README.md                    # This file
├── ETHICS.md                    # Ethics & privacy guidelines
├── docker-compose.yml           # Service orchestration
├── Makefile                     # Build & run commands
│
├── simulator/                   # Synthetic telemetry generator
│   ├── simulator.py            # Main simulator
│   ├── event_injector.py       # Critical event injection
│   └── tests/
│
├── ml-service/                  # PyTorch ML inference
│   ├── app.py                  # FastAPI service
│   ├── model.py                # LSTM architecture
│   ├── train.py                # Training script
│   ├── dataset_generator.py    # Synthetic data generation
│   └── tests/
│
├── fleet-orchestrator/          # Fleet management & coordination
│   ├── app.py                  # Main service
│   ├── decision_engine.py      # Decision logic
│   ├── models.py               # Data models
│   └── tests/
│
├── dashboard/                   # React web interface
│   ├── src/
│   │   ├── App.js              # Main app
│   │   ├── components/         # UI components
│   │   └── services/           # API clients
│   └── public/
│
└── docs/
    └── architecture.md          # Detailed architecture
```

---

## Configuration

### Environment Variables

**ML Service** (ml-service/.env):
```bash
MODEL_PATH=/app/models/fatigue_model.pth
DEVICE=cpu
LOG_LEVEL=INFO
```

**Fleet Orchestrator** (fleet-orchestrator/.env):
```bash
STORE_PII=false                 # CRITICAL: Keep false for privacy
ALERT_COOLDOWN_SECONDS=300
ML_SERVICE_URL=http://ml-service:8001
MQTT_BROKER=mqtt-broker
```

**Dashboard** (dashboard/.env):
```bash
REACT_APP_API_URL=http://localhost:8000
REACT_APP_WS_URL=ws://localhost:8000/ws
```

---

## Security & Privacy

### Data Protection

- **STORE_PII=false** by default (NEVER enable with real data)
- Driver IDs are hashed (SHA256)
- No raw audio or images stored
- TLS configuration available for production

### Anonymization

```python
def anonymize_telemetry(data):
    """Anonymize sensitive fields"""
    if not STORE_PII:
        data['driver_id'] = hashlib.sha256(data['driver_id'].encode()).hexdigest()[:16]
        data.pop('raw_audio', None)
        data.pop('face_image', None)
    return data
```

See [ETHICS.md](ETHICS.md) for full privacy guidelines.

---

## Dashboard Usage

### Fleet Overview

- **Map View**: See all trucks in real-time with fatigue indicators
- **Fleet List**: Grid of truck cards with status and quick actions
- **Alerts**: Active alerts sorted by severity

### Truck Status Colors

- **Green** (active): Normal operation, fatigue < 30%
- **Yellow** (warning): Elevated fatigue 30-60%
- **Orange** (alert): High fatigue 60-80%
- **Red** (emergency): Critical fatigue >80% or event detected

### Manual Actions

Fleet managers can trigger:
- **Micro Break**: 10-minute rest stop
- **Pull Over**: Immediate 30-minute rest
- **Driver Swap**: Coordinate driver change
- **Reroute**: Suggest alternate route
- **Emergency**: Alert authorities

---

## ML Model Details

### Architecture

- **Model**: LSTM (2 layers, 64 hidden units)
- **Input**: 60-second rolling window of 15 features
- **Output**: 3 risk scores (fatigue, microsleep, anomaly)
- **Framework**: PyTorch
- **Inference**: <100ms on CPU

### Features Used

**Driver Biometrics** (synthetic):
- Blink rate, eye closure duration
- Head tilt, steering grip pressure
- Heart rate, HRV, stress score
- Reaction delay

**Vehicle Telemetry**:
- Speed, harsh braking, lane deviation

**Context**:
- Time of day, weather, traffic density
- Time on route

### Decision Logic

```python
if fatigue_risk > 0.8 or "microsleep" in events:
    action = "EMERGENCY_PULL_OVER"
elif fatigue_risk > 0.6:
    action = "SCHEDULED_REST"
elif fatigue_risk > 0.4:
    action = "MICRO_BREAK"
else:
    action = "CONTINUE_MONITORING"
```

---

## 🛠️ Development

### Adding a New Feature

1. **Update Simulator**: Add new telemetry field in `simulator/simulator.py`
2. **Update ML Service**: Add feature to `FeatureExtractor` in `ml-service/app.py`
3. **Retrain Model**: Generate new dataset and retrain
4. **Update Orchestrator**: Modify decision logic if needed
5. **Update Dashboard**: Add UI components for new data

### Running in Development Mode

```bash
# Start only infrastructure
docker-compose up mqtt-broker

# Run services locally
cd ml-service && uvicorn app:app --reload --port 8001
cd fleet-orchestrator && uvicorn app:app --reload --port 8000
cd dashboard && npm start

# Run simulator
python simulator/simulator.py --output_mode mqtt
```

---

## Troubleshooting

### Services won't start

```bash
# Check service logs
docker-compose logs ml-service
docker-compose logs fleet-orchestrator

# Restart services
docker-compose restart
```

### Dashboard shows "Disconnected"

1. Check orchestrator is running: `curl http://localhost:8000/health`
2. Check WebSocket: Browser console should show connection errors
3. Verify no firewall blocking port 8000

### No trucks appearing

1. Ensure simulator is running: `docker-compose ps`
2. Check MQTT broker: `docker-compose logs mqtt-broker`
3. Verify simulator is publishing: Check simulator terminal output

### ML predictions seem random

This is expected if the model hasn't been trained. Run:
```bash
make train
```

---

## Makefile Commands

```bash
make build          # Build all Docker images
make up             # Start all services
make down           # Stop all services
make logs           # View all logs
make train          # Train ML model
make test           # Run all tests
make simulate       # Run truck simulator
make clean          # Clean up containers and volumes
```

---

## 🔮 Production Readiness

### Current Status: MVP / Demo

This MVP is designed for local testing and demonstration. For production deployment, see the Production TODOs below.

### Production TODOs

**Infrastructure**:
- [ ] Replace SQLite with PostgreSQL + TimescaleDB
- [ ] Add Redis for caching and pub/sub
- [ ] Implement Kafka for high-throughput telemetry
- [ ] Add load balancer for ML service
- [ ] Set up Kubernetes deployment

**Security**:
- [ ] Implement OAuth2 authentication
- [ ] Enable TLS for all services
- [ ] Add API rate limiting
- [ ] Implement audit logging
- [ ] Conduct security audit

**ML Improvements**:
- [ ] Deploy Transformer models for better accuracy
- [ ] Implement online learning / model updates
- [ ] Add federated learning across fleets
- [ ] Deploy edge inference devices
- [ ] Add model monitoring and drift detection

**Features**:
- [ ] Weather API integration
- [ ] Traffic API integration
- [ ] Predictive maintenance
- [ ] Route optimization
- [ ] Multi-fleet management
- [ ] Mobile driver app

**Monitoring**:
- [ ] Prometheus metrics
- [ ] Grafana dashboards
- [ ] Sentry error tracking
- [ ] Uptime monitoring

---

## License

MIT License - See LICENSE file for details

---

## Contributing

We welcome contributions! Please see CONTRIBUTING.md for guidelines.

**Key Areas**:
- ML model improvements
- Dashboard UX enhancements
- Integration with real fleet management systems
- Privacy-preserving techniques

---

## Ethics & Responsible AI

NeuroFleet Guardian is designed as a **preventative safety system**, not surveillance. Key principles:

1. **Consent First**: Drivers must explicitly consent and understand the system
2. **Transparency**: All data collection and decision logic must be transparent
3. **Driver Dignity**: Never punitive — designed to protect, not discipline
4. **Privacy**: Minimize data collection, anonymize everything, short retention
5. **Safe-Fail**: System failures must default to human judgment
6. **Oversight**: Human review of all critical decisions

See [ETHICS.md](ETHICS.md) for complete guidelines.

---

## Support

- **Issues**: https://github.com/your-org/neurofleet-guardian/issues
- **Discussions**: https://github.com/your-org/neurofleet-guardian/discussions
- **Email**: support@neurofleet.example.com

---

## Acknowledgments

Built with:
- PyTorch for ML inference
- FastAPI for backend services
- React + Leaflet for dashboard
- MQTT for telemetry streaming
- Docker for containerization

---

**Remember**: This system is designed to **protect drivers**, not surveil them. Always deploy with consent, transparency, and human oversight.
