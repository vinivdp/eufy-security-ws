# Fix: Station Connection State Not Updated on P2P Timeout/Disconnect

## Summary
This PR fixes a critical bug where station connection state remains stale after P2P connection failures, causing offline cameras to incorrectly appear as online with outdated property values.

## Problem Description

### Current Behavior (Bug)
When a station P2P connection times out or closes:
1. ✅ `station.on("close")` event fires correctly
2. ✅ `StationEvent.disconnected` is forwarded to clients
3. ❌ **BUT** the cached `connected` property is NOT updated
4. ❌ `station.isConnected()` still returns `true` (stale cached value)
5. ❌ Device property queries return outdated data showing camera as online

### Impact
- **Offline cameras appear online** with stale battery percentages
- **No reliable way to detect offline status** via WebSocket API
- **Particularly affects standalone cameras** (T8B0*, T8150*) that act as their own stations
- Clients polling device state receive incorrect connection status

### Real-World Example
```
Camera T8B00511242309F6 goes offline
→ P2P connection timeout after ~20s
→ station.on("close") fires
→ StationEvent.disconnected sent to clients
→ BUT station.isConnected() = true (cached)
→ device.get_properties returns: { connected: true, battery: 100% } (stale!)
```

## Root Cause
In `src/lib/forward.ts`, station event handlers only forward event notifications but do **not** send `propertyChanged` events to update the cached connection state:

```typescript
// Current code - only sends event, doesn't update state
station.on("close", () => {
    this.forwardEvent({
        source: "station",
        event: StationEvent.disconnected,
        serialNumber: station.getSerial()
    }, 0);
    // Missing: propertyChanged event for "connected" property!
});
```

## Solution
Send `StationEvent.propertyChanged` events when station connection state changes to keep cached state synchronized:

### Changes Made
1. **On station "connect"**: Send `propertyChanged(connected=true)`
2. **On station "close"**: Send `propertyChanged(connected=false)`
3. **On station "connection error"**: Send `propertyChanged(connected=false)`

### Code Changes
```typescript
station.on("close", () => {
    // Send disconnected event
    this.forwardEvent({
        source: "station",
        event: StationEvent.disconnected,
        serialNumber: station.getSerial()
    }, 0);

    // ALSO send property changed to update cached connected state
    this.forwardEvent({
        source: "station",
        event: StationEvent.propertyChanged,
        serialNumber: station.getSerial(),
        name: "connected",
        value: false
    }, 0);
});
```

## Testing

### Test Environment
- **Device**: Eufy 4G Starlight Camera (T8B00511242309F6)
- **Scenario**: Camera powered off to simulate offline state
- **Monitoring**: eufy-security-client logs + WebSocket API queries

### Test Results

#### Before Fix ❌
```
1. Camera goes offline
2. Logs show: "Timeout connecting to station T8B00511242309F6"
3. station.on("close") fires
4. Query device.get_properties:
   Response: { connected: true, battery: 100% } ← STALE DATA
5. Health check incorrectly shows camera as ONLINE
```

#### After Fix ✅
```
1. Camera goes offline
2. Logs show: "Timeout connecting to station T8B00511242309F6"
3. station.on("close") fires
4. StationEvent.propertyChanged(connected=false) sent
5. Query device.get_properties:
   Response: { connected: false, battery: null } ← FRESH DATA
6. Health check correctly shows camera as OFFLINE
```

### Verified Scenarios
- [x] P2P connection timeout (standalone camera offline)
- [x] Station disconnect event
- [x] Connection error event
- [x] Station reconnect (propertyChanged updates to `true`)
- [x] Multiple cameras with different connection states

## Backwards Compatibility
✅ **Fully backwards compatible**
- Only adds additional `propertyChanged` events
- Does not modify existing event structure
- No breaking changes to WebSocket API
- Clients that don't listen to `propertyChanged` are unaffected

## Related Issues
This fix addresses the core issue described in:
- Offline camera detection problems
- Stale device state caching
- Standalone camera connection status accuracy

## Checklist
- [x] Bug fix (non-breaking change which fixes an issue)
- [x] Tested in real-world environment
- [x] No breaking changes
- [x] Code follows project style
- [x] Commit message follows conventional commits format

## Additional Notes
This is a minimal, surgical fix that addresses the root cause without changing the overall architecture. The fix ensures that connection state changes propagate correctly through the event system, maintaining consistency between actual P2P connection status and cached property values.
