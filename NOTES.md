# Tanzania Software Logistics — Step-by-Step Notes
Stack: Python + Django + DRF + PostgreSQL + JS + Leaflet
Goal: Get a job in Tanzania as Supply Chain Systems Dev / ERP / Analyst

## Why this works in Tanzania
- Dar Port (CT2, Navis N4 TOS) + TRA TANCIS/TANESW Single Window = all digital now
- 27.7M tonnes 2024/25, +15%. Kwala Dry Port, SGR, Isaka/Mwanza corridor
- Gap: 87% no real-time tracking after gate-out, 73% still use phone/WhatsApp for scheduling
- Hiring: Maersk, Cargill, Coca-Cola, mining, NGOs (FHI 360 eLMIS/DHIS2). They ask for: SAP/Business Central/Odoo + SQL/PowerBI + Python/Postgres/APIs

Your pitch: "Python Fullstack who understands port clearance + inventory + tracking dashboards"

---

## PART 0: Setup (Do once)

1. Install: Python 3.12+, PostgreSQL 15+, Git, VS Code, Docker Desktop (later)
2. Create DB:
```sql
CREATE DATABASE logistics_db;
CREATE USER logistics_user WITH PASSWORD 'pass123';
GRANT ALL PRIVILEGES ON DATABASE logistics_db TO logistics_user;
```
3. Start Django project in this folder:
```bash
python -m venv venv
source venv/bin/activate
pip install django djangorestframework psycopg2-binary django-filter celery redis pillow
django-admin startproject config .
python manage.py startapp shipping
python manage.py startapp tracking
```
4. `config/settings.py`: add `shipping, tracking, rest_framework` to INSTALLED_APPS, set DATABASES to postgres logistics_db
5. Git:
```bash
git init
git add .
git commit -m "init: django + postgres setup"
```

Checklist: [ ] venv works [ ] admin loads [ ] postgres connects

---

## PART 1: Logistics Fundamentals (Week 1-2, 1hr/day)

Learn just enough to talk to employers:
- Flow: Supplier -> Vessel -> Dar Port discharge (berth 8-11 CT2) -> Yard -> Customs TANCIS/TANESW -> Gate-out -> Truck -> ICD/CFS Kwala/Kurasini -> Warehouse -> Customer
- Terms to memorize: Incoterms (FOB/CIF), manifest, TANCIS ref, demurrage (port free days ~7), storage, landed cost = freight + duty + transport, cycle count, inventory accuracy, dwell time = gate_out - discharge, turnaround
- Action: Watch 1x APICS CSCP intro + 1x TANCIS clearance flow on YouTube. Write 1-page in Swahili/English explaining Dar clearance steps.

---

## PART 2: Core Project - Mini Container Tracking System (Week 3-5)

This is your #1 portfolio piece. Replaces WhatsApp scheduling.

### Step 1: Models (Day 1-2)
Edit `shipping/models.py` and `tracking/models.py` — use schema from chat (Vessel, Container, ClearingJob, GateEvent, Device, TrackingPing, Alert).

Run:
```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
```

Seed 5 test containers in admin:
- MSKU1234567 - in yard (discharge 2 days ago)
- MSKU1234568 - in_transit to Kwala
- MSKU1234569 - in_transit to Kurasini
- TGHU9876543 - discharged today
- MSKU9999999 - in yard 8 days (demurrage risk)

### Step 2: Admin + Gate Flow (Day 3-4)
- Register all models in `admin.py`
- Logic: when GateEvent `gate_out` created -> set Container.status=in_transit + set gate_out_date=now. When `icd_in` -> status=ICD_arrived
- Add property on Container:
```python
@property
def dwell_days(self):
    end = self.gate_out_date or timezone.now()
    return (end - self.discharge_date).days
@property
def is_demurrage_risk(self):
    return self.dwell_days > 7 and self.status in ['discharged','yard']
```

### Step 3: APIs with DRF (Day 5-6)
Install DRF, create `shipping/serializers.py`, `tracking/serializers.py`
Endpoints to build:
```
POST /api/pings/ {device_id, lat, lng, speed_kmh}
GET  /api/containers/?status=in_transit
GET  /api/pings/?container=MSKU1234567&last=50
POST /api/gate-event/ {container_no, event_type, location}
```
Test with Postman / curl.

### Step 4: GPS Simulator (Day 6)
Create `simulate_gps.py`:
- Route Dar Port (-6.843, 39.288) -> Chalinze (-6.65, 38.75) -> Kwala (-6.70, 38.70)
- Every 10 sec POST to /api/pings/
- Run: `python simulate_gps.py MSKU1234568 DEV001`

### Step 5: Frontend Pages (Week 4-5)
Templates + Leaflet.js (no React needed):
1. `/` Dashboard: 4 KPI cards (in yard, in_transit, avg dwell, demurrage risk) + live map + recent alerts
2. `/containers/` Table: search by container_no/TANCIS, filter by status
3. `/containers/<no>/` Detail: timeline of GateEvents, ClearingJob form, demurrage calc, mini-map of its pings
4. `/schedule/` Board: ordered by scheduled_pickup, assign truck/driver, Gate-Out button
5. `/tracking/` Full map: auto-refresh JS fetch /api/pings/ every 10s
6. `/reports/` Dwell time + demurrage cost table + Export CSV

Leaflet snippet:
```html
<link rel="stylesheet" href="https://unpkg.com/leaflet/dist/leaflet.css"/>
<div id="map" style="height:400px"></div>
<script src="https://unpkg.com/leaflet/dist/leaflet.js"></script>
<script>
var map = L.map('map').setView([-6.8, 39.28], 8);
L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
fetch('/api/pings/?container=MSKU1234568&last=50')
 .then(r=>r.json()).then(data=>{
   var pts = data.map(p=>[p.lat,p.lng]);
   L.polyline(pts).addTo(map);
   pts.forEach(pt=>L.marker(pt).addTo(map));
 });
</script>
```

Checklist: [ ] 5 containers seeded [ ] gate-out updates status [ ] ping API saves [ ] map shows moving truck [ ] dashboard KPIs correct [ ] <3s load

---

## PART 3: Project 2 - Mini WMS (Week 6-7)
Build after tracking works. Mimics Dynamics 365 Business Central jobs in Arusha.

Tables: Product/SKU, Warehouse (Dar, Kwala), InventoryLedger (receipt, transfer, adjustment, repack consumption/output/loss), PurchaseReceipt
Features: receive, transfer Dar->Kwala, cycle count, variance report, landed-cost allocation, stock accuracy %

This gives you ERP vocabulary for interviews.

---

## PART 4: Polish for Tanzania Jobs (Week 8)

1. Power BI: connect to Postgres, build 1 dashboard: dwell time trend, demurrage cost by shipping line, fill rate, stock variance
2. Dockerize: Dockerfile + docker-compose (web + db + redis)
3. Deploy free: Render / Railway, post live link on CV
4. GitHub README must have: problem (Tanzania context), architecture diagram, demo video link, how to run, TANCIS/TOS integration notes
5. 2-min demo video (English, Swahili subtitles): show schedule -> gate-out -> live map -> dashboard
6. CV line: "Built Django/Postgres container tracking (dashboard 2.1s, GPS 3s delay, 20 concurrent users) + WMS with landed cost"

---

## PART 5: Apply

Where: Ajira Yako, ZoomTanzania, LinkedIn (search: Dar es Salaam SCM Analytics, Business Central, eLMIS)
Target titles: IT System & Data Analyst (logistics), Supply Chain Analyst, ERP Support (Odoo/Dynamics), Systems Developer eLMIS/DHIS2
Next certs: PL-300 (Power BI), Odoo Functional, DHIS2 Fundamentals, CSCP

Interview prep: explain dwell time, demurrage calc, TANCIS flow, how TOS + ERP + GPS integrate via API.

---

## Daily routine
- 1hr logistics theory / Swahili port terms
- 2hr coding (follow steps above in order, don't skip)
- 15min commit + update this checklist

Next: Tell me when Step 1 models are done, I'll give serializers + views code.
