# Ghost-Kitchen Event **Simulator**

Generate lifelike order, kitchen, and driver-tracking events for a fictional
Casper's “ghost kitchen” location then stream them into a Delta table for analytics demos, mapping, or real-time dashboards.

---

## Quick start (Databricks)

1. Clone this repo as a **Git Folder** into your workspace.
2. In the notebook toolbar for `simulator` notebook create the widgets that drive the run (or create a job):

   | widget name | example value | description |
   |-------------|---------------|-------------|
   | `catalog`   | `main`        | Unity Catalog or hive metastore catalog |
   | `schema`    | `gk_sim`      | target schema (created if missing) |
   | `table`     | `events`      | Delta table name |
   | `gk_location` | `115 Penn St, El Segundo, CA 90245` | address of the kitchen |
   | `start_ts`  | `2025-05-12 00:00:00` | simulation window start |
   | `end_ts`    | `2025-05-14 00:00:00` | simulation window end |
   | `start_orders` | `100` | daily orders at `start_ts` |
   | `end_orders`   | `200` | daily orders at `end_ts` |
   | `speed_up_factor` | `60` | 1 sim-minute per real second (`1` = real time) |

3. **Run** the notebook.  
   *Back-fill* events are written in one batch, then the notebook stays
   running and *streams* live events into  
   `main.gk_sim.events` (or whatever you chose).

4. Query the table:

   ```sql
   SELECT event_type, COUNT(*) 
   FROM   main.gk_sim.events 
   GROUP  BY event_type;
   ```

## What the notebook 

1. Build/Cache a road graph around the kitchen via OSMnx.
2. Sample reachable customer nodes inside that radius.
3. Snapshot menu items from the item table so orders reference a fixed menu.
4. Volume model
   - linear growth from start_orders → end_orders
   - weekly lift (Fri/Sat busiest)
   - lunch & dinner sine-wave peaks
5. Order life-cycle
   ```
   created → gk_started → gk_finished → gk_ready
          → driver_picked_up → driver_ping* → delivered
   ```

   Note config in code for configuring mean/stddev for each event.
6. Routing with NetworkX shortest-path; driver pings follow the polyline.
7. Writer
   - real-time rows → asyncio queue → flush when buffer full (configurable)

## Event Schema

| column       | type      | notes                 |
| ------------ | --------- | --------------------- |
| `event_id`   | string    | UUID                  |
| `event_type` | string    | lifecycle step        |
| `ts`         | timestamp | microsecond precision |
| `gk_id`      | string    | ghost-kitchen run id  |
| `order_id`   | string    | UUID                  |
| `sequence`   | int       | per-order counter     |
| `body`       | string    | JSON payload          |
| `day`        | date      | partition column      |


## Customizing the simulation

| Want to change…                | Edit in `simulator.ipynb`                                               |
| ------------------------------ | ----------------------------------------------------------------------- |
| **Kitchen address / radius**   | Widgets `gk_location`, `GK_RADIUS_MI` constant                          |
| **Driver speed**               | `GK_DRIVER_MPH`                                                         |
| **Service-time distributions** | `CONFIG["svc"]`                                    |
| **Volume profile**             | `orders_per_day()`, `minute_weight_vector()`, or the weekly multipliers |
| **Driver Ping frequency**      | `PING_INTERVAL_SEC`                                                     |
| **Output table location**      | widgets `catalog`, `schema`, `table`                                    |


