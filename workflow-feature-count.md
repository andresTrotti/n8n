# Workflow: Feature Count

## Objective
Goal: measure the use of fans, lights, thermostats and other features across fireplaces.

## Fireplace types
- Fireplace with lights
- Fireplace with lights and fan
- Fireplace with no light, no fan

## Workflow description

### Schedule Trigger
- Trigger: Manual (run on-demand).

### Vars and Redis
- Where the data comes from: values are read from Redis (or workflow variables containing Redis results).

### Digest data
- Purpose: determine for each `serial` whether the fireplace has `feature_light`, `feature_thermostat`, and `feature_fan`.

### Example code used in the Digest node

```javascript
var digested = {};

for (const dataItem of $input.first().json.data) {
    const separator = dataItem.indexOf(':');
    const serial = dataItem.substr(0, separator); 
    const strstatus = dataItem.substr(separator + 1); 

    const status = JSON.parse(strstatus);

    const digestedFeatures = {
        feature_light: status.feature_light,
        feature_thermostat: status.feature_thermostat,
        feature_fan: status.feature_fan
    }

    digested[serial] = digestedFeatures;
}

return digested;
```

### Counting (optional)
- By adding a counting node you can retrieve totals of fireplaces that have the named features (e.g., total with lights, total with fan, etc.).

## Convert to file and Save in Docker
- Create a file with the digested data and save it locally inside the container to persist it (example path inside container): `/data/digested/<YYYY-MM-DD>-features.json`.

Example Node.js snippet to write the digested object to a file:

```javascript
const fs = require('fs');
const outDir = '/data/digested';
fs.mkdirSync(outDir, { recursive: true });
const outPath = `${outDir}/${new Date().toISOString().slice(0,10)}-features.json`;
fs.writeFileSync(outPath, JSON.stringify(digested, null, 2));
```

- Docker considerations:
  - Mount a host directory into the container, e.g. `-v /var/lib/myapp/digested:/data/digested` so files persist outside container.
  - Ensure the container process has write permissions for the mounted directory.

## Execute a restart of the batch cleaning last data
- After saving the file, trigger a restart of the batch cleaning service that processes the last data.

Examples:
- Restart a Docker container named `batch-cleaner` on the host:

```bash
# on host
docker restart batch-cleaner
```

- From inside a container with access to the Docker socket (not recommended for production):

```bash
curl --unix-socket /var/run/docker.sock -X POST "http://localhost/containers/batch-cleaner/restart"
```

Security note: mounting the Docker socket into a container grants full control of the Docker daemon. Prefer safer orchestration or an authenticated API.
