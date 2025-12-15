# Workflow: Test features usages

## Objective
Measure how often the `timer` and `thermostat` functions are used, and also record changes to `light` and `fanspeed`, aggregated per device (`serial`).

## Flow summary
- Trigger: scheduled to run hourly (hours 0–23).
- Data source: variables / Redis.
- Day boundary detection: when the trigger hour is 0, a new day is considered to have started.
- Count reports: count the number of updates from devices.
- Report threshold: the flow continues only when there are at least 50,000 updates.
- Digesting: aggregate records per `serial`, sort by timestamp, and compute unique transitions:
  - `timer`: counts only transitions to `"1"` (activations).
  - `thermostat`: counts only transitions to `"1"` (activations).
  - `light` and `fanspeed`: count unique changes (each change increments).

## Nodes / Steps

### Schedule Trigger
- Configuration: run once every hour for hours 0 through 23.
- Purpose: start the workflow and capture the current hour for day-boundary logic.

### Vars and Redis
- Purpose: read the raw data entries from Redis (or from workflow variables that contain Redis data).
- Expected format: an array of strings where each entry is `"<serial>:<json-string>"`. The JSON contains fields including `timestamp`, `timer`, `light`, `fanspeed`, and `thermostat`.

### "If" — Day Passed Check
- Logic: if the current trigger hour equals `0`, consider that a day has passed and take any day-rollover actions as needed.

### Count Reports
- Purpose: count the total number of updates received (across all serials) during the period.

### Check Report Limit
- Condition: proceed only if the total number of updates is >= 50,000. This threshold avoids processing when data volume is insufficient.

### Digesting (aggregation & transition counting)
This node aggregates Redis data by `serial`, orders records by `timestamp`, and computes per-serial transition counts. Important rules:

- Records are grouped by `serial`.
- Within each serial group, records are sorted ascending by `timestamp`.
- For `timer` and `thermostat`, only transitions to the string value `"1"` are counted (i.e., activations).
- For `light` and `fanspeed`, every change from the previous value is counted as a unique change.
- If there are no input records, the node returns a single object indicating "No Data" with zero counts and `empty: true`.

Example implementation used in the Digesting node:

```javascript
var aggregatedData = {};
var transitionCounts = [];

const redisData = $('Redis').first().json.data;

if (!redisData || redisData.length === 0) {
    // If there's no data, return an object with the "empty" indicator
    return [{
        serial: "No Data",
        timer_activation_count: 0,
        light_unique_changes: 0,
        fanspeed_unique_changes: 0,
        thermostat_activation_count: 0,
        empty: true 
    }];
}

// Loop 1: group data by 'serial' and extract relevant properties
for (const dataItem of redisData) {
    const separator = dataItem.indexOf(':');
    const serial = dataItem.substr(0, separator);
    const strstatus = dataItem.substr(separator + 1);

    const status = JSON.parse(strstatus);

    const relevantStatus = {
        timestamp: status.timestamp,
        timer: status.timer,
        light: status.light, 
        fanspeed: status.fanspeed,
        thermostat: status.thermostat
    };

    if (serial in aggregatedData) {
        aggregatedData[serial].push(relevantStatus);
    } else {
        aggregatedData[serial] = [relevantStatus];
    }
}

// Loop 2: compute unique transitions per 'serial'
for (const serial in aggregatedData) {
    const records = aggregatedData[serial];
    
    records.sort((a, b) => a.timestamp - b.timestamp);

    let timerChanges = 0;
    let lightChanges = 0; 
    let fanspeedChanges = 0; 
    let thermostatActivations = 0; 

    // Initialize 'last' values from the first record for this serial
    let lastTimer = records[0].timer;
    let lastLight = records[0].light; 
    let lastFanspeed = records[0].fanspeed; 
    let lastThermostat = records[0].thermostat;

    for (let i = 1; i < records.length; i++) {
        const currentRecord = records[i];

        // Timer: count only transitions to "1"
        if (currentRecord.timer !== lastTimer) {
            if (currentRecord.timer === "1") {
                timerChanges++;
            }
            lastTimer = currentRecord.timer; 
        }

        if (currentRecord.light !== lastLight) {
            lightChanges++;
            lastLight = currentRecord.light;
        }

        if (currentRecord.fanspeed !== lastFanspeed) {
            fanspeedChanges++;
            lastFanspeed = currentRecord.fanspeed;
        }
        
        // Thermostat: count only transitions to "1"
        if (currentRecord.thermostat !== lastThermostat) {
            if (currentRecord.thermostat === "1") {
                thermostatActivations++;
            }
            lastThermostat = currentRecord.thermostat;
        }
    }

    // Save results for this serial
    transitionCounts.push({
        serial: serial,
        timer_activation_count: timerChanges, 
        light_unique_changes: lightChanges,
        fanspeed_unique_changes: fanspeedChanges,
        thermostat_activation_count: thermostatActivations,
        empty: false 
    });
}

return transitionCounts;
```

## Convert to file and Save in Docker
- Create a file with the digested data and save it locally inside the container so it can be persisted to a host volume.
- Suggested path inside container: `/data/digested/<YYYY-MM-DD>-digested.json`.
- Example NodeJS snippet to write the output array to a file:

```javascript
const fs = require('fs');
const path = `/data/digested/${new Date().toISOString().slice(0,10)}-digested.json`;
fs.mkdirSync('/data/digested', { recursive: true });
fs.writeFileSync(path, JSON.stringify(transitionCounts, null, 2));
```

- Docker considerations:
  - Mount a host directory into the container, e.g. `-v /var/lib/myapp/digested:/data/digested` so files are persisted outside the container lifecycle.
  - Ensure the container process has write permissions for the mounted directory.

## Execute workflow by last node
- After generating and saving the digested file, execute the workflow's last node actions. If the last node must trigger a batch cleaning restart, run the restart command or API call from the workflow.
- Example approaches:
  - If you control a systemd or supervisord job for the batch cleaner, call into the host (via an API or over SSH) to restart it.
  - If the batch cleaner is another Docker container, you can restart it from within a containerized orchestration environment (Kubernetes job, Docker API) or call an external deployment API.

- Minimal example using Docker CLI on the host to restart a container named `batch-cleaner`:

```bash
# on host
docker restart batch-cleaner
```

- If you need the workflow to trigger the restart directly from inside the container and Docker socket is mounted, you can call:

```bash
curl --unix-socket /var/run/docker.sock -X POST "http://localhost/containers/batch-cleaner/restart"
```

- Security note: mounting the Docker socket into a container grants full control of the Docker daemon. Prefer exposing a limited API or using orchestration tooling where possible.

## Output
- The Digesting step returns an array of objects, each with:
  - `serial` (string)
  - `timer_activation_count` (integer)
  - `light_unique_changes` (integer)
  - `fanspeed_unique_changes` (integer)
  - `thermostat_activation_count` (integer)
  - `empty` (boolean) — true only when there was no data

## Notes and suggestions
- Confirm the expected types for the fields in Redis (strings vs numbers). The current logic compares to string values like `"1"`.
- Consider storing timestamps as numeric Unix epoch milliseconds for reliable sorting.
- If you want, I can:
  - Add examples of input data and expected output,
  - Move this file to a docs/workflows directory and add index entries,
  - Convert this into a shorter README or a more detailed design document.
