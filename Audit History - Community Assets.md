# Collibra Activity History — All Assets in a Community

Query the [Collibra REST API](https://developer.collibra.com/rest/) to pull Activity (audit) History for every asset in a given community. This includes changes to the assets themselves and to their attributes, relations and responsibilities.

This is a companion to the `Audit History` sample in this repo, which targets `WorkflowInstance` activity.

## Step 1 — Get the Community ID

```
GET https://<instance-name>.collibra.com/rest/2.0/communities?name=<Community Name>&nameMatchMode=EXACT
```

Copy the `id` from the result. You can also find it in the community page URL in the Collibra UI.

## Step 2 — Pull Activities scoped to the Community

```
GET https://<instance-name>.collibra.com/rest/2.0/activities?contextId=<communityId>&resourceDiscriminators=Asset&resourceDiscriminators=Attribute&resourceDiscriminators=Relation&resourceDiscriminators=Responsibility&offset=0&limit=1000
```

**Example** (replace `<instance-name>` and `<communityId>`):

```
https://customer.collibra.com/rest/2.0/activities?contextId=0190a1b2-c3d4-7e5f-8a9b-0c1d2e3f4a5b&resourceDiscriminators=Asset&resourceDiscriminators=Attribute&resourceDiscriminators=Relation&resourceDiscriminators=Responsibility&offset=0&limit=1000
```

Optional: restrict to a time window with `startDate` / `endDate`. Both are Unix timestamps in **milliseconds**.

```
&startDate=1735689600000&endDate=1759190400000
```

## Worked Example — Community `c3f25ca7-db84-4ba2-a9b1-d0e423db3e84`

Full history for all assets in the community (page with `offset` = 0, 1000, 2000 … until a page returns fewer than 1000 results):

```
GET https://<instance-name>.collibra.com/rest/2.0/activities?contextId=c3f25ca7-db84-4ba2-a9b1-d0e423db3e84&resourceDiscriminators=Asset&resourceDiscriminators=Attribute&resourceDiscriminators=Relation&resourceDiscriminators=Responsibility&offset=0&limit=1000
```

Recent changes only (since 2026-09-01 00:00 UTC):

```
GET https://<instance-name>.collibra.com/rest/2.0/activities?contextId=c3f25ca7-db84-4ba2-a9b1-d0e423db3e84&resourceDiscriminators=Asset&resourceDiscriminators=Attribute&resourceDiscriminators=Relation&resourceDiscriminators=Responsibility&startDate=1788220800000&offset=0&limit=1000
```

Scope as of 2026-09-29: 1,217 assets across 23 domains and 16 sub-communities (Forest Service, National Interagency Fire Center, FPAC Business Center, Steampunk, etc.). Because the community is a parent with nested sub-communities, the fallback loop must cover every sub-community.

## Query Parameters

| Parameter | Description |
|---|---|
| `contextId` | ID of the community to scope the activity search to |
| `resourceDiscriminators` | Repeatable. Resource kinds to include (`Asset`, `Attribute`, `Relation`, `Responsibility`). Omit to return every activity in the community, including domain and community changes |
| `startDate` / `endDate` | Optional time window, Unix epoch in milliseconds |
| `performedByUserId` | Optional. Only return activities performed by this user |
| `offset` | Index of the first result, for paging |
| `limit` | Maximum number of results per page (up to 1000) |

## Fields Returned

| Field | Description |
|---|---|
| `id` | Unique identifier of the activity record |
| `timestamp` | Date/time the activity occurred |
| `user` | User who performed the activity |
| `activityType` | Type of activity (e.g., create, update, delete) |
| `cause` | What triggered the activity (e.g., manual edit, import, workflow) |
| `affectedName` | Name of the affected asset/object |
| `affectedType` | Type of the affected asset/object |
| `businessItemName` | Name of the related business item |
| `field` | Field that was changed |
| `oldValue` | Value before the change |
| `newValue` | Value after the change |
| `descriptionRaw` | Raw JSON payload / description of the activity |

## Fallback — Per-Asset Loop

If your instance returns only community-level events for a community `contextId` rather than events for child assets, iterate the assets and query each one:

```
GET https://<instance-name>.collibra.com/rest/2.0/assets?communityId=<communityId>&offset=0&limit=1000
GET https://<instance-name>.collibra.com/rest/2.0/activities?contextId=<assetId>&offset=0&limit=1000
```

If the community has sub-communities, check that the `/assets` call includes their assets. If it does not, list them with `GET /rest/2.0/communities?parentId=<communityId>` and repeat the loop for each one.

## Sample Script — Page Through and Export to CSV

```python
import csv, requests

BASE = "https://<instance-name>.collibra.com/rest/2.0"
AUTH = ("<username>", "<password>")   # or use an OAuth bearer token
COMMUNITY_ID = "c3f25ca7-db84-4ba2-a9b1-d0e423db3e84"
DISCRIMINATORS = ["Asset", "Attribute", "Relation", "Responsibility"]
PAGE = 1000

def get_all(path, params):
    offset = 0
    while True:
        r = requests.get(f"{BASE}{path}", params={**params, "offset": offset, "limit": PAGE}, auth=AUTH)
        r.raise_for_status()
        results = r.json().get("results", [])
        yield from results
        if len(results) < PAGE:
            break
        offset += PAGE

def community_activities():
    # requests expands a list value into repeated query params
    return get_all("/activities", {"contextId": COMMUNITY_ID, "resourceDiscriminators": DISCRIMINATORS})

def per_asset_activities():
    for asset in get_all("/assets", {"communityId": COMMUNITY_ID}):
        for act in get_all("/activities", {"contextId": asset["id"]}):
            act["_assetName"] = asset["name"]
            yield act

rows = list(community_activities())
# rows = list(per_asset_activities())   # use the fallback if needed

fields = sorted({k for row in rows for k in row})
with open("community_audit_history.csv", "w", newline="", encoding="utf-8") as f:
    w = csv.DictWriter(f, fieldnames=fields)
    w.writeheader()
    for row in rows:
        w.writerow({k: row.get(k) for k in fields})

print(f"Exported {len(rows)} activities")
```

## Notes

- Activity field names can vary by Collibra version. Confirm the exact response shape in your instance's REST Core API docs before building reports on it.
- Large communities can produce a lot of history. Use `startDate` / `endDate` to pull incremental windows instead of the full history each time.
- The user running the query needs view permissions on the community's resources; activities on resources they can't see are not returned.
