# Ghost-Kitchen Event **Simulator**

Generate lifelike orders and driver-tracking events for a fictional Casper's “ghost kitchen” location and stream them into a Delta table for analytics demos, mapping, or real-time dashboards.

---

## Quick start (Databricks)

1. Clone this repo as a **Git Folder** into your workspace.
2. Review the default configuration in `config.json`. There are sensible defaults. Make changes as needed (detailed instructions below)
   - Destination `catalog` must already exist and anything already in the `schema` **will be overwritten**
3. Open `simulator` notebook and hit **Run All**
4. Query the table! 

   ``` sql
   SELECT
      count(*),
      date_format(ts, 'MMM dd') as day
   FROM
      {catalog}.{schema}.events
   WHERE
      event_type = 'order_created'
   GROUP BY
      date_format(ts, 'MMM dd')
   ORDER BY
      date_format(ts, 'MMM dd')
   ```

This will populate the following entites in the configured `catalog.schema`

| Name | Type | Notes | 
| - | - | - |
| `brand`| Table | Brands at this location |
| `menu` | Table |  Menus at this location (relation: belongs to brand) | 
| `category` | Table |  Categories for menus (appetizer, mains, etc) (relation: belongs to menu, brand)
| `item` | Table |  Specific items, prices, etc. (relation: belongs to category, menu, brand)
| `events` | Table |  Live event data from generator producing event types listed above | 
| `cache` | Volume |  Cache for routing data | 

Note the foreign key relationships between tables. You can view them in the entity relationship diagram available in the catalog explorer.

Following events are produced:

1. `created `
2. `gk_started`
3. `gk_finished`
4. `gk_ready`
5. `driver_arrived`
6. `driver_picked_up` 
7. `driver_ping`(*)
8. `delivered`

## Configuration

Tips for configuring your ideal data

#### Where is the data?

| Key                            | What it points to                           | Typical change                                               |
| ------------------------------ | ------------------------------------------- | ------------------------------------------------------------ |
| **`catalog / schema / table`** | Where the simulator writes its dimension data (menus brands, etc.) and events Delta table. | Point them at your Unity Catalog namespace or a dev sandbox. |

#### When, how fast, and how many orders?

| Block                 | Setting                       | What it means                                                                                                                                 | Try this                                                                        |
| --------------------- | ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Simulation window** | `start_ts`, `end_ts`          | The real-world period you want to *pretend* happened. Back-fill covers the span **up to `now`**, and real-time streaming picks up from there. | Run a whole month (`2025-05-01` → `2025-05-31`) to make brand momentum visible. |
| **Speed-up**          | `speed_up`                    | Wall-clock compression. `60` ⇒ 1 simulated minute takes 1 real second.                                                                        | Demo in front of an audience? Bump to `600` for “one day in 2½ minutes.”        |
| **Demand ramp**       | `orders_day_1`, `orders_last` | Linear growth target across the window.                                                                                                       | Double them to stress-test downstream pipelines.                                |
| **Noise**             | `noise_pct`                   | ±X % jitter on daily demand to keep dashboards lively.                                                                                        | Set to `0` for boring graphs                                  |


#### Inside the kitchen

| Block                                               | Setting                                                                                                                                    | What it tweaks                                                                                                   | Rule of thumb                                                    |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| **Service-stage mean/stdev**<br>`svc = { cs, sf, fr, rp }` | Mean/stdev (minutes) for four kitchen stages:<br>• **cs** chef-start<br>• **sf** cooking<br>• **fr** boxing<br>• **rp** ready→pickup lag | Cut the means in half to turn your ghost kitchen into a “dark microwave” (fast)                                    |                                                                  |
| **Driver arrival**                                  | `alpha`, `beta`                                                                                                                            | Beta(α, β) controls *shape* of arrival within the chosen interval. Higher α skews later, higher β skews earlier. | `alpha:5, beta:1` → drivers hover outside until the last minute. |
|                                                     | `after_ready_pct`                                                                                                                          | Probability the driver shows up **after** the kitchen marks the order ready.                                     | Lower to `0.2` if you want lots of drivers waiting around.       |

#### Brand level story lines

| Block       | Setting                                             | Behaviour                                                                                                                   | What to expect                                                                                                                   |
| ----------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Buckets** | `brand_momentum = { improving / flat / declining }` | Deterministic split of your brands into three camps. Must sum to 1.                                                         | `0.5 / 0.3 / 0.2` makes the market more competitive.                                                                             |
| **Rates**   | `momentum_rates = { growth / decline }`             | Compound change **per 30 days**. The factor is applied exponentially each day and re-normalised so total orders stay fixed. | **Small window?** Juice `growth` to `0.5` and `decline` to `0.4` or extend the `start_ts–end_ts` span to see visible divergence. |

#### Data quality issues?

| Block    | What it simulates                                                                        | How to use                                                      |
| -------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| **`dq`** | Field-level dropouts or nulls per event type. E.g. `0.01` ⇒ 1 % of rows lose that field. | Crank `driver_ping.loc_lat` to `0.25` to test GPS-gap handling. |

#### Where is the kitchen? Where are my customers?

| Block         | Setting                    | Impact                                                                                                             |
| ------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Geography** | `gk_location`, `radius_mi` | Kitchen anchor and customer catchment zone. Changing it triggers an automatic rebuild of the OSM road graph cache. |
|               | `driver_mph`               | Determines ETAs and ping cadence realism. Rural demo? Drop to `18`.                                                |
| | `ping_sec` | Interval between driver_ping events. Default `60`. Lower means more location data | 

#### Operational

| Setting                      | Why it exists                                              | Default                            |
| ---------------------------- | ---------------------------------------------------------- | ---------------------------------- |
| `batch_rows / batch_seconds` | Trade-off between write latency and Delta commit overhead. | `10 / 1 s` is safe for most tests. |
| `random_seed`                | Reproduce a run exactly.                                   | `42`                               |

