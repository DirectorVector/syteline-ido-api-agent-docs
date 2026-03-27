---
name: ido-inspect
description: Inspect a Syteline IDO — fetch its properties, methods, and optionally method parameters from the live API.
---

# IDO Introspection

Inspect the Syteline IDO named **$ARGUMENTS** using the live API.

Run the following steps using the Bash tool. For each step, fetch a fresh token immediately before the API call.

**Token fetch pattern (use every time):**
```bash
RESPONSE=$(curl -s "$SYTELINE_BASE_URL/token/$DEFAULT_SITE" \
  -H "username: $SYTELINE_AGENT_USERNAME" \
  -H "password: $SYTELINE_AGENT_PASSWORD")
TOKEN=$(pwsh -NoProfile -Command "('$RESPONSE' | ConvertFrom-Json).Token")
```

---

## Step 1 — Verify the IDO exists

```bash
curl -s "$SYTELINE_BASE_URL/load/IdoCollections?properties=CollectionName,AccessAs&filter=CollectionName%3D'$ARGUMENTS'&recordcap=1" \
  -H "Authorization: $TOKEN"
```

- If `Items` is empty, the IDO name is wrong. Try a LIKE search: `filter=CollectionName%20LIKE%20N'%25$ARGUMENTS%25'`
- Note the `AccessAs` value — `BaseSyteLine` = core Infor IDO, empty = custom IDO

## Step 2 — Get properties

```bash
curl -s "$SYTELINE_BASE_URL/load/IdoProperties?properties=PropertyName,DataType,IsReadOnly,ColumnName&filter=CollectionName%3D'$ARGUMENTS'&recordcap=0" \
  -H "Authorization: $TOKEN"
```

## Step 3 — Get methods

```bash
curl -s "$SYTELINE_BASE_URL/load/IdoMethods?properties=MethodName,MethodType&filter=CollectionName%3D'$ARGUMENTS'&recordcap=0" \
  -H "Authorization: $TOKEN"
```

## Step 4 — Present results

Summarize what you found in a structured format:

- **IDO name** and `AccessAs`
- **Writable properties** (IsReadOnly is null or "0") — list PropertyName + ColumnName
- **Read-only properties** (IsReadOnly = "1") — list briefly
- **Derived properties** (ColumnName is null) — list separately
- **Methods** — list MethodName, note MethodType (0, 2, or 3)
- Any subcollections (PropertyName with no ColumnName that looks like a child IDO link)

If the user asks to inspect a specific method's parameters, run:

```bash
curl -s "$SYTELINE_BASE_URL/load/IdoMethodParameters?properties=Sequence,ParameterName,DataType,SpDataType,SpDataLength,InputFlag,OutputFlag&filter=CollectionName%3D'$ARGUMENTS'%20AND%20MethodName%3D'METHOD_NAME'&orderby=Sequence&recordcap=0" \
  -H "Authorization: $TOKEN"
```

Replace `METHOD_NAME` with the method name the user specifies.
