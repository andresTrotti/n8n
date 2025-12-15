# Workflow: Feature Count
<img width="1033" height="457" alt="Untitled 2" src="https://github.com/user-attachments/assets/ee839050-2616-402e-a6f4-73bc6b0d1a9f" />

## Objective
Goal: measure the use of fans, lights, thermostats and other features across fireplaces.
- Fireplace with lights
- Fireplace with lights and fan
- Fireplace with no light, no fan

## Workflow description

### Schedule Trigger
<img width="134" height="116" alt="Captura de pantalla 2025-12-15 a la(s) 3 47 19 p  m" src="https://github.com/user-attachments/assets/9dee2ed5-cdc4-4566-9d6b-5ad15aab3d71" />

- Trigger: Manual (run on-demand).

### Vars and Redis
<img width="235" height="109" alt="Captura de pantalla 2025-12-15 a la(s) 3 47 32 p  m" src="https://github.com/user-attachments/assets/327d8318-33e5-440f-bf6a-8c293a3fb183" />

- Where the data comes from: values are read from Redis (or workflow variables containing Redis results).

### Digest data
<img width="122" height="110" alt="Captura de pantalla 2025-12-15 a la(s) 3 47 46 p  m" src="https://github.com/user-attachments/assets/13b139cc-05c6-43dd-a395-9d39665c2663" />

- Purpose: determine for each `serial` whether the fireplace has `feature_light`, `feature_thermostat`, and `feature_fan`.


### Counting (optional)

<img width="155" height="95" alt="Captura de pantalla 2025-12-15 a la(s) 3 49 03 p  m" src="https://github.com/user-attachments/assets/8ebd3ca7-fe4b-4ec6-9933-57ab20198233" />

- By adding a counting node you can retrieve totals of fireplaces that have the named features (e.g., total with lights, total with fan, etc.).

<img width="352" height="209" alt="Untitled 6" src="https://github.com/user-attachments/assets/7873b507-fb93-43a5-9c62-7e5a0fae405f" />


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


## Convert to file and Save in Docker
- Create a file with the digested data and save it locally inside the container to persist it (example path inside container): `/data/digested/<YYYY-MM-DD>-features.json`.


## Execute last node and see result

<img width="352" height="209" alt="Untitled 6" src="https://github.com/user-attachments/assets/ab0d34df-e540-454d-9072-af5af87f9a07" />
