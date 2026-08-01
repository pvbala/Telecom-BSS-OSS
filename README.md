# Telecom BSS/OSS Prototype

A Django-based prototype covering 15 core telecom modules (8 BSS + 7 OSS) for two
products: **Mobile Prepaid SIM** and **Home Fiber Broadband**.

This is deliberately built as a single modular monolith rather than microservices —
one Django project, one Django "app" per module, one PostgreSQL database. Network
behavior (provisioning, activation, usage, faults) is simulated rather than
integrated with real network elements, since this is a prototype.

## Modules

**BSS:** `customers`, `catalog`, `orders`, `billing`, `payments`, `crm`,
`comm_inventory`, `tickets`

**OSS:** `service_inventory`, `resource_inventory`, `provisioning`, `activation`,
`usage`, `fault_mgmt`, `assurance`

## Quick start (Docker — recommended)

```bash
docker compose up --build
```

Then, in a second terminal:

```bash
docker compose exec web python manage.py createsuperuser
docker compose exec web python manage.py seed_demo_data
```

Visit:
- Admin (all 15 modules' CRUD screens): http://localhost:8000/admin/
- Service Assurance dashboard: http://localhost:8000/admin/assurance/dashboard/

## Quick start (without Docker, using SQLite)

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
python manage.py migrate
python manage.py createsuperuser
python manage.py seed_demo_data
python manage.py runserver
```

## Demo flow

1. **Go to `/admin/`** and log in. You'll see all 15 modules listed as
   individually manageable sections.
2. **Add a customer** under Customers.
3. **Go to Orders → Add order**, pick the customer and a price plan
   (e.g. "Value Pack" for prepaid SIM, or "Home 100" for fiber).
4. **Select the order in the Orders list** and use the admin action dropdown:
   - "Advance selected orders: -> VALIDATED"
   - "Advance selected orders: -> PROVISIONED (assigns resource)" — this
     reserves a free MSISDN/fiber port from Resource Inventory
   - "Advance selected orders: -> ACTIVE (activates service)" — this creates
     a live entry in Service Inventory
5. **Add a Balance** (Billing) for the order if it's a prepaid SIM, so usage
   simulation has something to deduct from.
6. **Run the simulator** to generate usage + occasional faults:
   ```bash
   python manage.py simulate_network_events          # single tick
   python manage.py simulate_network_events --loop --interval 10   # continuous
   ```
   This will create Usage Records, deduct prepaid balance, and — roughly 30% of
   ticks — raise a Fault on an assigned resource. Major/critical faults
   auto-open a Trouble Ticket.
7. **Check `/admin/assurance/dashboard/`** to see active services, open
   faults, and open tickets in one view.
8. **Simulate a payment**: go to Payments, add one linked to an order/invoice,
   then use the "Simulate gateway: mark SUCCESS" admin action.

## Notes on design choices

- **State machine**: `Order.transition_to()` enforces valid status transitions
  (`CREATED → VALIDATED → PROVISIONED → ACTIVE → COMPLETED`, with `FAILED` as
  an escape hatch). Every transition is logged to `OrderStatusHistory`.
- **Resource allocation**: `resource_inventory.NetworkResource` models the
  MSISDN pool (prepaid SIM) and fiber port pool, with status
  `FREE → RESERVED → ASSIGNED`.
- **Prepaid vs. fiber billing differs on purpose**: prepaid balance is
  decremented per usage event; fiber usage is logged but not billed per-use
  (flat monthly `Invoice` instead).
- **Mock payment gateway** (`payments/services.py`) deterministically succeeds
  or fails — swap this out for a real gateway integration later without
  touching any other module.
- **Reporting**: Django Admin gives list/filter/search/CSV export
  (via `django-import-export`) for free on every module. The one custom page
  is the Service Assurance dashboard, which aggregates Fault + Ticket +
  Service Inventory data.

## Extending this later

- Split modules into separate services once real scale/ownership boundaries
  emerge — the current app-per-module structure makes that split easier, not
  harder.
- Replace the mock payment gateway and network adapters with real
  integrations behind the same function signatures.
- Add DRF serializers/viewsets per module if you need a customer-facing
  self-care API (buy SIM, check balance) beyond the admin screens.
