# Background Task Submission

Background tasks are how Syteline runs utilities, reports, and posting routines asynchronously. The `BGTaskSubmit` method on the `BGTaskDefinitions` IDO is the universal entry point.

## Submitting a Task

```
POST /invoke/BGTaskDefinitions?method=BGTaskSubmit
Authorization: {token}
Content-Type: application/json
Body: [ ...20 positional parameters... ]
```

### The 20 Parameters

| Index | Name | Type | Direction | Typical Value |
|---|---|---|---|---|
| 0 | TaskName | VARCHAR(50) | INPUT | `"ChangeCOStatusUtility"` |
| 1 | TaskParms1 | VARCHAR(MAX) | INPUT | Task-specific parameter string |
| 2 | TaskParms2 | VARCHAR(MAX) | INPUT | `null` |
| 3 | Infobar | VARCHAR(2800) | OUTPUT | `null` (returns error text on failure) |
| 4 | TaskID | NUMERIC(11) | OUTPUT | `null` (returns assigned task number) |
| 5 | TaskStatusCode | VARCHAR(8) | INPUT | `null` |
| 6 | StringTable | VARCHAR(255) | INPUT | `null` |
| 7 | RequestingUser | VARCHAR(128) | INPUT | `"$SYTELINE_USERNAME"` |
| 8 | PrintPreview | BYTE | INPUT | `0` (**not** `false`) |
| 9 | TaskHistoryRowPointer | GUID | OUTPUT | `null` |
| 10 | PreviewInterval | LONG(10) | OUTPUT | `null` |
| 11-19 | Scheduling fields | Various | INPUT | `null` for immediate execution |

### Immediate Execution Template

For running a task right now (no scheduling), set indices 11-19 to `null`:

```json
[
  "TASK_NAME_HERE",
  "TASK_PARMS1_HERE",
  null, null, null, null, null,
  "$SYTELINE_USERNAME",
  0,
  null, null,
  null, null, null, null, null, null, null, null, null
]
```

All 20 positions must be present. Use `null` for unused slots.

### Curl Example

```bash
curl -s -X POST \
  "$SYTELINE_BASE_URL/invoke/BGTaskDefinitions?method=BGTaskSubmit" \
  -H "Authorization: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '["ChangeCOStatusUtility","TASK_PARMS1_VALUE",null,null,null,null,null,"$SYTELINE_USERNAME",0,null,null,null,null,null,null,null,null,null,null,null]'
```

### Successful Response

```json
{
  "Success": true,
  "ReturnValue": "0",
  "Parameters": [
    "ChangeCOStatusUtility",
    "...",
    "",
    "",
    "2939087",
    "", "", "{USERNAME}", "0", "", "",
    "", "", "", "", "", "", "", "", ""
  ]
}
```

**TaskID is at `Parameters[4]`** — save this for monitoring.

---

## TaskParms1 Format

Every background task has its own parameter format. There are thousands of submittable tasks. There is no universal schema — you must discover the format per task.

### Three observed styles:

**Style 1: Comma-delimited positional values** — Used by status-change utilities.
```
,,,,,,,,,,,,,,T,T,C,MESSAGE,,,,,,
```

**Style 2: SETVARVALUES()** — Used by reports and posting procedures. Named key=value pairs.
```
SETVARVALUES(NewId=08b34d68-...,PostThroughVar=2026-02-28 00:00:00.000,Parm_Site={SITE_FROM_USER_SCOPE})
```

**Style 3: ~LIT~() with BG~TASKID~ tokens** — Used by on-demand reports. Syteline resolves placeholders at runtime.
```
~LIT~(V000222721),E,,0,1,1,1,0,1,0,BG~TASKID~,1
```

### How to Discover TaskParms1 for Any Task

**Query BGTaskHistories** — someone has almost certainly run this task before:

```bash
curl -s -X GET \
  "$SYTELINE_BASE_URL/load/BGTaskHistories?properties=TaskName,TaskNumber,TaskParms1,TaskParms2,CompletionStatus,CompletionDate,RequestingUser&filter=TaskName%20LIKE%20N'YOUR_TASK%25'&orderby=CompletionDate%20DESC&recordcap=5" \
  -H "Authorization: $TOKEN"
```

If no history exists, run the task once manually in the Syteline UI, then query again.

---

## Monitoring Task Status

### Step 1: Check if the task is still active

```bash
curl -s -X GET \
  "$SYTELINE_BASE_URL/load/ActiveBGTasks?properties=TaskNumber,TaskName,TaskStatusCode&filter=TaskNumber%20%3D%20{TASK_ID}" \
  -H "Authorization: $TOKEN"
```

If `Items` is empty, the task has completed (it moved to history).

### Step 2: Check completion in history

```bash
curl -s -X GET \
  "$SYTELINE_BASE_URL/load/BGTaskHistories?properties=TaskNumber,TaskName,CompletionStatus,TaskErrorMsg,StartDate,CompletionDate&filter=TaskNumber%20%3D%20{TASK_ID}" \
  -H "Authorization: $TOKEN"
```

- `CompletionStatus: "0"` — success
- `CompletionStatus: "-1"` — failed (check `TaskErrorMsg`)

### Step 3: Get detailed error logs (on failure)

```bash
curl -s -X GET \
  "$SYTELINE_BASE_URL/load/ProcessErrorLogs?properties=*&filter=TaskNumber%20%3D%20{TASK_ID}" \
  -H "Authorization: $TOKEN"
```

### Polling Pattern (bash)

```bash
TASK_ID=2939087
while true; do
  RESULT=$(curl -s -X GET \
    "$SYTELINE_BASE_URL/load/BGTaskHistories?properties=TaskNumber,CompletionStatus,TaskErrorMsg&filter=TaskNumber%20%3D%20$TASK_ID" \
    -H "Authorization: $TOKEN")
  STATUS=$(echo "$RESULT" | jq -r '.Items[0].CompletionStatus // empty')
  if [ -n "$STATUS" ]; then
    echo "Task $TASK_ID completed with status: $STATUS"
    echo "$RESULT" | jq '.Items[0]'
    break
  fi
  echo "Waiting..."
  sleep 5
done
```

---

## Key IDOs for Background Tasks

| IDO | Purpose |
|---|---|
| `BGTaskDefinitions` | Task definitions + `BGTaskSubmit` method |
| `ActiveBGTasks` | Currently queued/running tasks |
| `BGTaskHistories` | Completed task history (results, errors, parameters used) |
| `ProcessErrorLogs` | Detailed error messages from failed tasks |
