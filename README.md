# Simple Self-consumption DESS

This Node-RED flow runs on a Victron GX and plans the battery from **EnergyZero** day-ahead prices (Dutch market, no API key):

- **Charge** to 100% during the cheapest block of `chargeHours` consecutive hours (default 3).
- In every other hour (or quarter, with 15-minute prices):
  - **Self-consume** if the price is higher than what the stored energy cost plus the battery cost. The battery follows the live consumption and never exports.
  - Otherwise **hold**: the battery does not discharge and the grid covers the load.

Prices can be hourly or per 15 minutes (`priceInterval`). The plan is written into the GX's Dynamic ESS schedule slots, with DESS in **mode 4 (Node-RED)**. No VRM account, token or API is used.

## What is in this repository

- `victron-nodered-simple-dess.json` — importable Node-RED flow.
- `README.md` — this file.

## Prerequisites

On the GX / Venus OS device:

1. Node-RED running on the GX with access to local dbus.
2. `node-red-contrib-victron` installed (provides `victron-output-custom`).
3. The GX set up as ESS. Victron advises *Optimized (without BatteryLife)* for Dynamic ESS.
4. GX timezone set to `Europe/Amsterdam` (Venus OS: *Settings → Date & Time → Time zone*); the flow uses the host's local time.

## Import

1. In Node-RED: **Menu → Import → paste/select** `victron-nodered-simple-dess.json`.
2. Import it as a **Flow**.
3. Configure (below) and **Deploy**.

## Configure

Open the function node **`Edit me: site configuration (On Start tab)`**, switch to its **On Start** tab and edit the `cfg` object. The *On Message* tab only re-applies these settings; there is nothing to edit there.

```js
// Charge window
priceInterval: 60,               // 60 = hourly prices, 15 = quarter-hour prices
chargeHours: 3,                  // length of the charge window (hours it takes to fill the battery)
targetSoc: 100,                  // SOC to charge to in the charge window
priceProperty: 'all_in_with_vat',// EnergyZero price array to use
refreshFromHour: 15,             // local hour from which tomorrow's prices are fetched

// After the charge window
b_cycle_cost: 0.01,              // battery cost per kWh cycled (EUR/kWh)
roundTripEfficiency: 1,          // e.g. 0.85 to include charge/discharge losses
minSoc: 10,                      // shown as target in self-consume hours (see below)

// Battery, also written to the GX
b_max: 10,                       // usable battery capacity (kWh)
tb_max: 5,                       // max battery CHARGE power (kW)
fb_max: 5,                       // max battery DISCHARGE power (kW)
feed_in_possible: true,          // allow solar feed-in to the grid

// GX schedule slots
scheduleSlots: 48,               // 48 on current Venus OS, 4 on older versions
writeTargetSoc: true             // false on Venus OS versions without Schedule/N/TargetSoc
```

Notes:

- `priceInterval`: `60` plans per hour with EnergyZero's hourly prices. `15` uses the quarter-hour prices (`INTERVAL_QUARTER`) and plans per 15 minutes, so the flow can also use cheap or expensive quarters within an hour. Changing it clears the cached prices and upcoming charge windows and fetches them again; ended windows stay as the price reference.
- `chargeHours`: the flow picks tomorrow's cheapest block of this many consecutive hours (lowest average price, ties go to the earliest). `1` = the single cheapest hour. With `priceInterval: 15` the block can start at any quarter (e.g. 02:45–05:45), and fractions such as `2.5` or `2.75` work (rounded to whole quarters).
- `tb_max` is the charge rate the GX uses in the charge window (**T**o **B**attery; `fb_max` = **F**rom **B**attery). If a full charge takes about 3 hours, `chargeHours: 3` fits.
- `b_max`, `tb_max` and `fb_max` are written to the GX (`/Settings/DynamicEss/BatteryCapacity`, `BatteryChargeLimit`, `BatteryDischargeLimit`). The GX uses them to turn a target SOC into a charge rate (capacity × SOC difference / time left in the slot, capped by the limit). If `BatteryCapacity` is 0 or wrong, scheduled charging is too slow or does not happen.
- `b_cycle_cost` and `roundTripEfficiency` set the self-consume threshold, see *How it works*. With `roundTripEfficiency: 1` the rule is exactly "price > charge price + battery cost". At 0.85, a kWh from the battery effectively costs `charge price / 0.85`.
- `minSoc`: in self-consume hours the GX ignores the target SOC. The value only shapes the plan graph, so set it to your ESS minimum SOC to get a realistic line instead of 0%.
- `priceProperty`: `all_in_with_vat` is the consumer all-in price including VAT, used for both the charge price and the prices per period.
- `refreshFromHour`: EnergyZero publishes tomorrow's prices in the early afternoon. Until then it returns an error, and the flow retries every hour.
- After editing, click **Deploy**. The *On Start* code runs on every deploy and restart, so you do not need to press **Reload config**. That button only re-applies the deployed settings, e.g. after flow context was cleared.

## One-time DESS mode switch

After deploying, check the `Rules plan (per period)` debug output (press **`Run now (test)`**), then:

1. Click **`Set DESS mode = 4 (Node-RED)`**.
2. Confirm the `Set DynamicEss/Mode` node status shows `4 (Node-RED)`.

Do not enable Dynamic ESS in VRM at the same time: VRM's Auto mode (1) writes the same schedule slots. To stop using this flow, click **`Revert DESS mode = 1 (VRM Auto)`** first, then disable the flow.

> Do not leave DESS in mode 4 while this flow is disabled. The GX keeps following the last written plan until it runs out (at most 48 hours), and then falls back to self-consumption.

## How it works

### Every hour

1. `Every hour` (at minute 0) triggers `Load cached cheapest hour`. It fetches EnergyZero prices when needed:
   - **tomorrow's prices**, from `refreshFromHour`, if tomorrow has no charge window yet;
   - **today's prices**, if the current period has no price (e.g. after a restart).

   Fetches are retried on each hourly run (at most once per 30 minutes, also when you press the buttons):

   ```text
   https://public.api.energyzero.nl/public/v1/prices?energyType=ENERGY_TYPE_ELECTRICITY&date=DD-MM-YYYY&interval=INTERVAL_HOUR      (INTERVAL_QUARTER with priceInterval: 15)
   ```

2. The response covers two or three days. All prices are stored in flow context (`cheapestHourDessPrices`). The cheapest block of `chargeHours` hours on the requested date is stored as that day's charge window (`cheapestHourDessTargets`). Ended windows are kept for 48 hours as the price reference. If prices are not available yet, the cached plan is used unchanged.
3. The `Rules planner` decides the mode per period (hour or quarter), from now until the last known price (up to 48 hours). Consecutive periods with the same settings are merged into one GX slot, e.g. *self-consume 17:15–21:45* or a whole charge window. That way the 48 slots cover the full horizon, also with 15-minute prices (one slot per quarter would only reach 12 hours ahead).
4. `Map plan to dbus writes` writes the changes to the GX schedule slots.

### Rules

| Period | Condition | Slot | What the GX does |
|---|---|---|---|
| charge | inside the charge window | `Strategy` 2, `TargetSoc` = `targetSoc` | charges from the grid at up to `tb_max` |
| self-consume | price > charge price / `roundTripEfficiency` + `b_cycle_cost` | `Strategy` 1, `Restrictions` 1 (no battery → grid), `TargetSoc` = `minSoc` | battery covers the live consumption, never exports; the target is ignored |
| hold | otherwise (price equal or lower) | `Strategy` 2, `TargetSoc` 100, `Restrictions` 2 (no grid → battery) | battery does not discharge, grid covers the load; surplus solar can still charge it |

- *Charge price* is the average price of the most recent charge window that has ended, i.e. what the energy in the battery cost. Before the first window has ended, the upcoming window is used.
- A charge window is written as one slot, so the "time left in the slot" the GX uses for the charge rate is the rest of the window: it spreads the charge over the window (capped at `tb_max`) instead of aiming for `targetSoc` within each hour or quarter.
- The 100% in hold hours is required, not cosmetic. With a target below the actual SOC, the GX would discharge towards it.
- The behaviour per strategy follows Victron's GX code (`venus-dynamicess`).
- The planner charges from the grid in the window regardless of the solar forecast. In sunny months a lower `targetSoc` may be better.

The `Rules plan (per period)` debug node shows the plan as text, one line per hour or quarter:

```text
25 01:00  self   0.2600 (threshold 0.2300)
25 02:00  charge 0.2200
25 12:00  hold   0.2300 (threshold 0.2300)
```

### Writing to the GX

```text
com.victronenergy.settings
  /Settings/DynamicEss/Schedule/<slot>/Start, Duration, Soc, TargetSoc,
                                       AllowGridFeedIn, Restrictions, Strategy, Flags
  /Settings/DynamicEss/BatteryCapacity, BatteryChargeLimit, BatteryDischargeLimit
```

The GX uses whichever slot contains the current time; slot order does not matter. A planned slot stays in the GX slot it already occupies (also when the current slot just continues, e.g. *self-consume until 21:45* planned an hour ago); new ones go into free slots. `Soc` and `TargetSoc` get the same value: current Venus OS uses `TargetSoc` when it is above 0, older versions use `Soc`.

**Flash wear.** The GX stores these settings in a file via `localsettings`. Writing an unchanged value does nothing there, and changes are saved to flash at most once per 2-second burst. So the flow:

- only sends values that differ from what it wrote before;
- leaves slots whose period has already ended untouched (the GX ignores them);
- disables unused slots with a single `Duration = 0` write;
- only disables a slot before rewriting it if it still holds a current or future period, so the GX never acts on a half-written slot;
- writes at 20 values per second, so a batch of changes falls into a few 2-second save windows.

In a two-day simulation with real prices, an hourly run without new prices wrote nothing, and tomorrow's prices arriving caused 5 (hourly) or 33 (15-minute) writes, one settings-file save. Everything is sent once after each deploy or restart, which also disables slots left over from VRM Auto mode, and again every 6 hours; unchanged values in those repeats are ignored by the GX.

## Debug nodes

- `Selected charge window` — the charge window found by each price refresh (`msg.found` is `null` if prices were not available yet).
- `Rules plan (per period)` — mode, price and threshold for every planned hour or quarter.
- `Schedule writes` — each dbus path/value written (only changes: usually nothing, a few hundred after a deploy).

## Schedule viewer

<img width="1043" height="612" alt="dess_schedule" src="https://github.com/user-attachments/assets/39230276-2fb4-463b-b8fe-a94e4a3cf6d1" />

Group **7 — Schedule viewer** serves a read-only page with the current plan. Open it at the same address as the Node-RED editor, with `/dess` appended (e.g. `https://venus.local:1881/dess`; if `httpNodeRoot` is set in `settings.js`, it goes in front of `/dess`).

- A bar chart of the price per hour or quarter, coloured by mode (charge / self-consume / hold), with the self-consume threshold as a dashed line, day separators and a "now" marker. Hover over a bar for its details.
- Summary tiles (current mode, battery SoC, current price vs. threshold, next charge window, planned hours per mode and the number of GX slots used) and a table you can expand.
- `/dess?format=json` returns the same data as JSON, including `soc: { value, at }`.
- `/dess?format=ha` returns a compact JSON for Home Assistant, see below.

The battery SoC comes from the GX (`com.victronenergy.system` → `/Dc/Battery/Soc`) via the `Battery SoC` node, which only sends changes (rounded to 0.1%). The tile therefore shows the time of the *last change*, not of the last check. After a deploy it says "waiting for the GX" until the first value arrives. If the GX reports no valid SoC (e.g. the battery monitor is offline), the tile shows "unknown since …" instead of 0%.

The page shows what the `Rules planner` stored in flow context (`cheapestHourDessPlan`) on its last run, so it is at most an hour old, and it reloads every 5 minutes. It writes nothing to the GX. After a deploy it says "No plan yet" until the next run (or press `Run now (test)`). The endpoint has no login of its own; it follows `httpNodeAuth` in `settings.js`, if set.

### Home Assistant (apexcharts-card)

`/dess?format=ha` returns a compact version for a Home Assistant REST sensor:

```json
{ "mode": "self", "price": 0.26, "threshold": 0.23, "soc": 55.3,
  "next_charge_start": "2026-11-26T01:00:00.000Z", "next_charge_end": "2026-11-26T04:00:00.000Z", "next_charge_price": 0.21,
  "planned_at": "2026-11-25T18:00:00.000Z", "interval": 15, "horizon_hours": 33,
  "periods": [[1795631400000, 0.26, 0.23, "self"], [1795632300000, 0.22, null, "charge"], ...] }
```

Each `periods` entry is `[start (ms), price, threshold or null, mode]`. `horizon_hours` is the number of whole hours from the start of the current hour to the end of the plan (about 9 in the morning, up to about 33 once tomorrow's prices are in), used as the chart span. 48 hours of quarters is about 7 KB, below Home Assistant's 16 KB limit for attributes stored in the recorder.

Sensor (`configuration.yaml`; use the same address as the page):

```yaml
rest:
  - resource: http://venus.local:1881/dess?format=ha
    scan_interval: 300
    sensor:
      - name: DESS plan
        unique_id: dess_plan
        value_template: "{{ value_json.mode }}"
        json_attributes: [price, threshold, soc, next_charge_start, next_charge_end, next_charge_price, planned_at, interval, horizon_hours, periods]
```

Card: price bars coloured by mode, plus the threshold. It needs [apexcharts-card](https://github.com/RomRider/apexcharts-card) and [config-template-card](https://github.com/iantrich/config-template-card) (both via HACS). apexcharts-card only takes a fixed `graph_span`, so config-template-card fills it in from `horizon_hours`; the chart then always ends where the plan ends instead of leaving the rest of a fixed span empty.

```yaml
type: custom:config-template-card
entities:
  - sensor.dess_plan
card:
  type: custom:apexcharts-card
  header:
    show: true
    title: DESS plan
  graph_span: ${states['sensor.dess_plan'].attributes.horizon_hours + 'h'}
  span:
    start: hour
  now:
    show: true
    label: now
  stacked: true
  yaxis:
    - decimals: 2
  apex_config:
    legend:
      show: true
  series:
    - entity: sensor.dess_plan
      name: Charge
      type: column
      color: "#2a78d6"
      data_generator: |
        return entity.attributes.periods.map(p => [p[0], p[3] === 'charge' ? p[1] : null]);
    - entity: sensor.dess_plan
      name: Self-consume
      type: column
      color: "#eb6834"
      data_generator: |
        return entity.attributes.periods.map(p => [p[0], p[3] === 'self' ? p[1] : null]);
    - entity: sensor.dess_plan
      name: Hold
      type: column
      color: "#1baf7a"
      data_generator: |
        return entity.attributes.periods.map(p => [p[0], p[3] === 'hold' ? p[1] : null]);
    - entity: sensor.dess_plan
      name: Threshold
      type: line
      curve: stepline
      stroke_width: 2
      color: "#8a8984"
      data_generator: |
        return entity.attributes.periods.map(p => [p[0], p[2]]);
```

The three mode series are stacked so each bar sits in its own time slot (only one of them has a value per period). `periods` only holds periods that have not ended, so the chart starts at the current hour.

## Troubleshooting

- **Warning `EnergyZero has no ... prices for YYYY-MM-DD yet`** — normal before tomorrow's prices are published. The flow retries every hour and keeps running meanwhile. If it persists into the evening, check the endpoint and the GX timezone. Use `Force price refresh` to retry immediately.
- **Rules planner: `no price for the current period yet`** — first run after a restart; today's prices are fetched on the same or the next run.
- **Battery discharges when it should hold** — check the `Rules plan (per period)` output for that hour or quarter. If it says `self`, the price was above the threshold. Raise `b_cycle_cost` or lower `roundTripEfficiency` to hold more often.
- **Battery does not reach `targetSoc` by the end of the charge window** — the GX charges at up to `tb_max`. Check that `tb_max` and `b_max` match reality, and increase `chargeHours` if charging takes longer. Confirm DESS mode is `4`.
- **Write errors on `Schedule/4/...` or `.../TargetSoc`** — older Venus OS with 4 slots and no `TargetSoc`: set `scheduleSlots: 4` and `writeTargetSoc: false`. With 4 slots the plan reaches less far ahead (until the fifth change of mode).
- **Plan graph shows a charge ramp at the start of every hold block** — the graph colours every step up to 100% as charging. In hold hours the GX is blocked from grid charging; the measured SOC line should stay flat there.
- **Prices or charge windows lost after a Node-RED restart** — they are kept in flow context, which is in memory by default. Today's prices are fetched again automatically. Until the next charge window has ended, the charge price comes from the upcoming window. Configure a persistent `contextStorage` in Node-RED `settings.js` to keep them.

## Safety and ownership

- In mode 4 this flow, not VRM, decides when the battery charges and discharges. You are responsible for the battery limits and costs you configure.
- Test with `Run now (test)` (or `Force price refresh` to also fetch prices) and the debug sidebar before relying on it.
- If you are unsure, use `Revert DESS mode = 1 (VRM Auto)` and disable this flow.
