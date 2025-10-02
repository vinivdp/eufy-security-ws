# Station Connection State Bug Fix

## Overview
This fork contains a critical bug fix for eufy-security-ws that resolves stale connection state caching when station P2P connections fail.

## The Bug
**Symptom**: Offline cameras appear online with outdated battery readings

**Root Cause**: When a station P2P connection times out:
- The library correctly emits `disconnected` and `connectionError` events
- **BUT** it does NOT update the cached `connected` property
- Subsequent queries return stale `connected: true` state

## The Fix
Modified `src/lib/forward.ts` to send `propertyChanged` events when connection state changes:

```typescript
// On station connect/disconnect/error
this.forwardEvent({
    source: "station",
    event: StationEvent.propertyChanged,
    serialNumber: station.getSerial(),
    name: "connected",
    value: false  // or true for connect
}, 0);
```

This ensures `station.isConnected()` returns fresh, accurate values.

## Impact
- ✅ Fixes offline detection for standalone cameras (T8B0*, T8150*)
- ✅ Device state queries return real-time connection status
- ✅ Resolves "zombie camera" issue (offline showing as online)
- ✅ Fully backwards compatible

## Using This Fork

### npm
```bash
npm install git+https://github.com/vinivdp/eufy-security-ws.git
```

### Docker
```dockerfile
FROM node:20-alpine

WORKDIR /app

# Clone this fixed fork
RUN apk add --no-cache git && \
    git clone https://github.com/vinivdp/eufy-security-ws.git . && \
    npm ci && \
    npm run build

CMD ["node", "dist/bin/server.js"]
```

### Render.com (Direct from GitHub)
1. Create new Web Service
2. Connect repository: `vinivdp/eufy-security-ws`
3. Build settings:
   - **Runtime**: Docker
   - **Dockerfile Path**: `docker/Dockerfile`
4. Deploy

## Testing the Fix

### Before Fix
```bash
# Camera is offline but reports as online
curl -X POST http://localhost:3000 -H "Content-Type: application/json" -d '{
  "messageId": "test",
  "command": "device.get_properties",
  "serialNumber": "T8B00511242309F6",
  "properties": ["connected", "battery"]
}'

# Returns STALE data:
# { "connected": true, "battery": 100 }
```

### After Fix
```bash
# Same query with fix applied
curl -X POST http://localhost:3000 -H "Content-Type: application/json" -d '{
  "messageId": "test",
  "command": "device.get_properties",
  "serialNumber": "T8B00511242309F6",
  "properties": ["connected", "battery"]
}'

# Returns FRESH data:
# { "connected": false, "battery": null }
```

## Contributing Back to Upstream
A pull request has been prepared with:
- Detailed problem description
- Root cause analysis
- Solution explanation
- Real-world test results
- Backwards compatibility verification

See `PULL_REQUEST.md` for the full PR description ready to submit to [bropat/eufy-security-ws](https://github.com/bropat/eufy-security-ws).

## Files Changed
- `src/lib/forward.ts` - Added propertyChanged events for connection state updates

## Acknowledgments
- Bug discovered and fixed by [@vinivdp](https://github.com/vinivdp)
- Built on [bropat/eufy-security-ws](https://github.com/bropat/eufy-security-ws)
- Assisted by Claude Code (Anthropic)

## License
MIT (same as upstream eufy-security-ws)
