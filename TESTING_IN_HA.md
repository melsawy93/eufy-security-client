# Testing C30 Lock Support in Home Assistant - Quick Reference

**⚠️ IMPORTANT:** Container filesystem is ephemeral. Changes are lost when:
- Add-on is stopped
- Add-on is updated
- Add-on is reinstalled
- HA restarts (container may be recreated)

## Quick Iteration Cycle

### 1. Build TypeScript to JavaScript (Local Machine)

```bash
cd eufy-security-client
npm run build
```

**Output:** Compiled files in `build/http/`:
- `types.js`, `types.js.map`
- `device.js`, `device.js.map`
- `station.js`, `station.js.map`

### 2. Upload Files to HA

Using **File Editor** add-on:
1. Navigate to `/config/`
2. Create/use folder: `c30_update`
3. Upload 6 files from `build/http/`:
   - `types.js`, `types.js.map`
   - `device.js`, `device.js.map`
   - `station.js`, `station.js.map`

### 3. Start Add-on & Find Container

```bash
# Start add-on (creates container)
ha addons start 402f1039_eufy_security_ws
sleep 20

# Find container ID
CONTAINER=$(docker ps | grep "402f1039_eufy_security_ws" | awk '{print $1}')
echo "Container: $CONTAINER"
```

### 4. Copy Files to Container

```bash
CONTAINER="<container_id>"  # From step 3
BUILD_PATH="/usr/src/app/node_modules/eufy-security-client/build/http"

# Copy JavaScript files
docker cp /config/c30_update/types.js $CONTAINER:$BUILD_PATH/types.js
docker cp /config/c30_update/device.js $CONTAINER:$BUILD_PATH/device.js
docker cp /config/c30_update/station.js $CONTAINER:$BUILD_PATH/station.js

# Copy source maps
docker cp /config/c30_update/types.js.map $CONTAINER:$BUILD_PATH/types.js.map
docker cp /config/c30_update/device.js.map $CONTAINER:$BUILD_PATH/device.js.map
docker cp /config/c30_update/station.js.map $CONTAINER:$BUILD_PATH/station.js.map

# Verify
docker exec $CONTAINER ls -lh $BUILD_PATH/ | grep -E "types|device|station"
```

### 5. Restart Add-on & HA

```bash
# Restart add-on to load new files
ha addons restart 402f1039_eufy_security_ws
sleep 20

# Restart HA integration (in HA Dashboard → Developer Tools → Services)
# service: homeassistant.restart
```

### 6. Test C30 Lock

- Check **Settings → Devices & Services → Eufy Security**
- Test lock/unlock: `service: lock.lock` / `lock.unlock`
- Check battery sensor
- Test push notifications

## Complete One-Liner Script

```bash
# Build locally first, then run this in HA terminal:
ha addons start 402f1039_eufy_security_ws && sleep 20 && \
CONTAINER=$(docker ps | grep "402f1039_eufy_security_ws" | awk '{print $1}') && \
BUILD_PATH="/usr/src/app/node_modules/eufy-security-client/build/http" && \
docker cp /config/c30_update/types.js $CONTAINER:$BUILD_PATH/types.js && \
docker cp /config/c30_update/device.js $CONTAINER:$BUILD_PATH/device.js && \
docker cp /config/c30_update/station.js $CONTAINER:$BUILD_PATH/station.js && \
docker cp /config/c30_update/types.js.map $CONTAINER:$BUILD_PATH/types.js.map && \
docker cp /config/c30_update/device.js.map $CONTAINER:$BUILD_PATH/device.js.map && \
docker cp /config/c30_update/station.js.map $CONTAINER:$BUILD_PATH/station.js.map && \
ha addons restart 402f1039_eufy_security_ws
```

## Troubleshooting

### Container Not Found
```bash
# Container is removed when add-on stops - start it first
ha addons start 402f1039_eufy_security_ws
sleep 20
docker ps | grep eufy
```

### Files Not Updating
```bash
# Verify files were copied
CONTAINER=$(docker ps | grep "402f1039_eufy_security_ws" | awk '{print $1}')
docker exec $CONTAINER cat /usr/src/app/node_modules/eufy-security-client/build/http/types.js | grep -i "LOCK_85D0"
```

### Add-on Won't Start
```bash
# Check logs
ha addons logs 402f1039_eufy_security_ws
```

## Notes

- **Files are lost on add-on stop/update** - This is normal Docker behavior
- **Must rebuild TypeScript** - Add-on uses compiled JavaScript, not TypeScript
- **Must restart add-on** - New files only load after restart
- **For permanent changes** - Submit PR and wait for official add-on update

## File Locations

**Local (after build):**
- `eufy-security-client/build/http/*.js`
- `eufy-security-client/build/http/*.js.map`

**HA File Editor:**
- `/config/c30_update/*.js`
- `/config/c30_update/*.js.map`

**Container (runtime):**
- `/usr/src/app/node_modules/eufy-security-client/build/http/*.js`
