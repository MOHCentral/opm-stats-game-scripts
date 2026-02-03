# JSON Migration Guide for Game Scripts

## ✅ COMPLETED

### API Changes
- **handlers.go**: Now accepts JSON arrays `[{...},{...}]` as primary format
- **Backward compatible**: Still accepts newline-delimited URL-encoded for legacy scripts
- **Bruno files**: All 105 .bru files now send JSON arrays

### Script Changes (tracker_common.scr)
- ✅ Added `build_json_field` - builds JSON key-value pairs
- ✅ Added `json_escape` - basic string escaping
- ✅ Added `build_player_json` - converts player data to JSON fields
- ✅ Added `build_base_json` - builds base event object with type/match_id/timestamp
- ✅ Added `queue_event_json` - queues JSON objects into array
- ✅ Updated `flush_queue` - sends JSON array with proper headers

### Updated Event Handlers
- ✅ `send_match_start` - converted to JSON
- ✅ `heartbeat_loop` - converted to JSON  
- ✅ `on_player_jump` - converted to JSON

## 🔄 REMAINING WORK

### Pattern for Conversion

**OLD (URL-encoded):**
```morpheus
on_some_event local.player:
    local.payload = waitthread global/tracker_common.scr::build_base_payload "event_type"
    local.payload = local.payload + waitthread global/tracker_common.scr::build_player_payload local.player "player"
    local.payload = local.payload + "&custom_field=" + local.value
    
    thread global/tracker_common.scr::queue_event local.payload
end
```

**NEW (JSON):**
```morpheus
on_some_event local.player:
    local.json = waitthread global/tracker_common.scr::build_base_json "event_type"
    local.json = local.json + "," + (waitthread global/tracker_common.scr::build_player_json local.player "player")
    local.json = local.json + "," + (waitthread global/tracker_common.scr::build_json_field "custom_field" local.value 1)
    
    thread global/tracker_common.scr::queue_event_json local.json
end
```

### Files to Update

1. **global/tracker.scr** - Remaining handlers:
   - `send_match_end`
   - `send_round_start`
   - `send_round_end`
   - `on_player_land`
   - `on_player_crouch`
   - `on_player_prone`
   - `on_player_distance`
   - `on_player_movement`
   - `on_ladder_mount`
   - `on_ladder_dismount`
   - `on_player_use`
   - `on_client_connect`
   - `on_client_disconnect`
   - `on_client_begin`
   - `player_accuracy_loop`
   - `on_team_join`
   - `send_player_auth_event`

2. **global/tracker_combat_ext.scr** - All combat events
3. **global/tracker_items_ext.scr** - All item/pickup events
4. **global/tracker_gameflow_ext.scr** - Game lifecycle events
5. **global/tracker_interaction_ext.scr** - Chat/interaction events
6. **global/tracker_movement_ext.scr** - Movement events
7. **global/tracker_client_ext.scr** - Client events

### curl_post Signature Change

**EXPECTED NEW ENGINE SIGNATURE:**
```
curl_post url headers body callback
```

Where `headers` is a string with newline-separated headers:
```
"Content-Type: application/json\nX-Server-Token: abc123"
```

### Important Notes

1. **String vs Number Fields**:
   - Use `1` as the third arg to `build_json_field` for strings
   - Use `0` for numbers/booleans

2. **JSON Object Completion**:
   - Event handlers build incomplete JSON (no closing `}`)
   - `queue_event_json` adds the closing brace
   - This allows flexible field addition

3. **Server Auth**:
   - Server token now goes in `X-Server-Token` header
   - No longer in URL query params or body
   - server_id injected by API middleware (don't send in body)

4. **Backward Compatibility**:
   - API supports both formats during transition
   - Game scripts can be updated incrementally
   - Test each file after conversion

## Testing Checklist

After updating each script file:

1. ✅ Check syntax (no missing quotes/braces)
2. ✅ Test event fires without errors
3. ✅ Check API logs show `Parsed as JSON array`
4. ✅ Verify `processed > 0` in response
5. ✅ Confirm data appears in ClickHouse

## Quick Reference

### Add Simple Field
```morpheus
local.json = local.json + "," + (waitthread global/tracker_common.scr::build_json_field "field_name" local.value 1)
```

### Add Numeric Field
```morpheus
local.json = local.json + "," + (waitthread global/tracker_common.scr::build_json_field "score" local.score 0)
```

### Add Player Data
```morpheus
local.json = local.json + "," + (waitthread global/tracker_common.scr::build_player_json local.player "player")
```

### Queue the Event
```morpheus
thread global/tracker_common.scr::queue_event_json local.json
```
