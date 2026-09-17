# Demo provisioning runbook (catalog-driven, caller-aware)

Stands up the live Field-Level Entitlement Filter demo with `anypoint-cli-v4` + the
A2D MCP tools. Per-field sensitivity is derived from CDGC, so you need a real IDMC
tenant with a **scanned schema asset** whose columns are linked to Business Terms
(one of which is marked with the `sensitiveMarker`, e.g. "Confidential").

## Live IDs used for this build (org `030e0aac-30d9-460f-9234-428c16a123c4`, env Sandbox `5933e3e7-…`)

| Thing | Value |
|---|---|
| A2D mock MCP server | `f2a18790-2be4-4f8b-bc2d-0db85a052d88` |
| Mock URL | `https://www.a2d-ai.com/api/platform/f2a18790-2be4-4f8b-bc2d-0db85a052d88/mcp` |
| Exchange asset | `product-catalog-entitlement/1.0.0` (type `mcp`) |
| API Manager instance | `21180385` (label `entitlement-filter-demo`) |
| Flex gateway target | `6a42db0e-fe38-4db5-8e73-6ec2da8e69de` (`omni-gw-small`, v1.13.5) |
| Applied policy id | `9263610` (`field-level-entitlement-filter` 1.0.0) |
| Governed schema asset | `cb2345f7-d54f-4d8b-ba76-9b78b14576c4` (`dim_product.csv`) |
| Governed endpoint | `https://omni-gw-small-fqj8wl.5sc6y6-1.usa-e2.cloudhub.io/entitlement-filter-demo/mcp` |

## 0. Find the schema-asset id (CDGC search API)

```bash
curl -s -X POST "https://cdgc-api.<pod>.informaticacloud.com/ccgf-searchv2/api/v1/search" \
  -H "Authorization: Bearer <jwt>" -H "X-INFA-ORG-ID: <orgId>" \
  -H "X-INFA-SEARCH-LANGUAGE: elasticsearch" -H "Content-Type: application/json" \
  -d '{"from":0,"size":25,"query":{"bool":{"must":[{"terms":{"core.classType":["com.infa.odin.models.file.flat.FlatFile"]}}]}}}'
```
`core.identity` → `schemaId`. (Get `<jwt>` via `/identity-service/api/v1/Login`
then `/jwt/Token` — the same chain the policy uses.)

## 1. A2D mock (returns the SAME full record for everyone)

`design_mcp_server` (type `mock`) + `add_mcp_tool get_products` with one scenario
(`condition: catalog === "products"`) returning all governed columns, **including
`unit_cost`** (the field whose Business Term is marked Confidential). The mock does
NOT vary by caller — the policy does.

## 2. Publish + deploy the MCP Flex instance

```bash
anypoint-cli-v4 exchange:asset:upload --name "Product Catalog Entitlement Demo" \
  --type mcp --status published --properties='{"platform":"a2d"}' \
  --files='{"mcp-metadata.json":"./mcp-metadata.json"}' product-catalog-entitlement/1.0.0

anypoint-cli-v4 api-mgr:api:manage product-catalog-entitlement 1.0.0 \
  --environment Sandbox --isFlex --type mcp --withProxy \
  --scheme http --port 8081 --path "/entitlement-filter-demo/" \
  --uri "https://www.a2d-ai.com/api/platform/<mockServerId>/" \
  --apiInstanceLabel entitlement-filter-demo

anypoint-cli-v4 api-mgr:api:deploy <apiInstanceId> --environment Sandbox \
  --target <gatewayId> --gatewayVersion 1.13.5 --overwrite
```

## 3. Apply the filter (config = one id + creds + entitlement rule)

```bash
cp config.json.example config.json   # fill cdgc creds/urls + schemaId + clearedLevels/allowedPurposes
anypoint-cli-v4 api-mgr:policy:apply <apiInstanceId> field-level-entitlement-filter \
  --environment Sandbox --groupId <orgId> --policyVersion 1.0.0 --configFile ./config.json
anypoint-cli-v4 api-mgr:api:redeploy <apiInstanceId> --environment Sandbox
```

## 4. Run

```bash
cp env.local.sh.example env.local.sh   # set CMP_GW_URL
./demo.sh
```

Expected: analyst (`internal`/`analytics`) → `unit_cost` masked, `x-entitlement-filtered: 1`;
fraud investigator (`restricted`/`fraud-detection`) → `unit_cost` visible, `x-entitlement-filtered: 0`.

Quick manual check of both personas + the header:

```bash
GW="https://<gatewayPublicHost>/entitlement-filter-demo/mcp"
BODY='{"jsonrpc":"2.0","id":9,"method":"tools/call","params":{"name":"get_products","arguments":{"catalog":"products"}}}'
curl -sS -i -X POST "$GW" -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" -H "mcp-session-id: e-demo" \
  -H "x-dp-clearance: internal" -H "x-dp-purpose: analytics" -d "$BODY" | grep -iE "x-entitlement-filtered|unit_cost"
```

## Notes
- Egress: the policy calls `cdgcLoginUrl` + `cdgcSearchUrl` (both `format:service`);
  confirm the gateway can reach `*.informaticacloud.com`.
- `recordsPath=products` because records live under `products` in the payload.
- Sensitivity comes from a field's Business Term description containing the
  `sensitiveMarker` (default "confidential"); the demo relies on `unit_cost`'s term.
- In production the clearance/purpose headers would be set by an upstream identity
  policy (or a JWT-to-header mapping), not by the caller directly.
