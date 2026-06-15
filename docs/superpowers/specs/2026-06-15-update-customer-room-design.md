# updateCustomerRoom — design

Date: 2026-06-15
Source spec: `globalconnect-project-documentation/api_de/sequence_diagrams/7000_0400-update_cust_room_v1.0.puml`

## Goal

Add a new jBPM process `updateCustomerRoom` to the `GC-AUTOMATION` kjar. It re-parents a
service's CPE node to a target room (a `TECHNICAL_AREA` node identified by a Smallworld external
id). It follows the same authoring conventions as `pniDesign` and `logicalDesign`: every external
call is an `SdcApiCall` work item routed by Automaton, the FIND segment runs before the UPDATE
segment, and the process returns a synchronous `response` Map.

## Inputs (from the TMF_STORE request body, extracted by JSONATA)

| field | type | meaning |
|---|---|---|
| `version` | String | API contract version, echoed in the response |
| `crossProjectName` | String | CROSS project the re-parent happens in |
| `serviceCrossId` | String | the service whose CPE is re-parented (ODIN `ServiceID`) |
| `smwRoomId` | String | target room, Smallworld external id, becomes the CPE's new parent |
| `country` | String | resolves the SMW `systemId` used to look up `smwRoomId` |

Start parameter: `_order` (String) = the `automaton.tmf_store` id, as for every process.

## Process flow (`GC-AUTOMATION.updateCustomerRoom`)

```
Start
 → [SdcApiCall] TMF_STORE: get data
        domain=TMF_STORE, operation=GET, transformation=JSONATA, dataMap.id=_order
        jsonata: version, crossProjectName, serviceCrossId, smwRoomId, country
        On Exit: if country=="DE" → smwSystemId="PNI_DE" else throw IllegalArgumentException
 → [SdcApiCall] REST_API_CALL: cross/findService   (in: serviceCrossId → out: serviceComponents)
 → [Script]     find cpeCrossId
        find serviceComponent where componentTypeName == "CPE"; take primary resource (else first);
        cpeCrossId = resource.resourceRef.id; on any miss set errorMessage
 → <exclusive gw>  errorMessage != null → throw Link "Failed"   ELSE → continue
 → [SdcApiCall] REST_API_CALL: cross/findNode/CPE  (routeMap.source = cross/findNode; in: cpeCrossId →
        out: cpeName, cpeStatusDiscriminator, cpeCapacityFull, cpeNodeTypeDiscriminators)
 → [SdcApiCall] REST_API_CALL: cross/findRoom
        On Entry: roomCriteria = "systemId=="+smwSystemId+";externalId=="+smwRoomId+";nodeTypeDiscriminator==TECHNICAL_AREA"
        On Exit: validate resultList (not empty / unique) → roomCrossId; else errorMessage
 → <exclusive gw>  errorMessage != null → throw Link "Failed"   ELSE → throw Link "Upsert"

(catch "Upsert")
 → [SdcApiCall] REST_API_CALL: cross/updateCpeParent
        On Entry: parentCrossIds = [roomCrossId]
        PUT re-parent the CPE → throw Link "Completed"

(catch "Completed") → [Script] Create result: COMPLETED → End
(catch "Failed")    → [Script] Create result: FAILED   → End
```

`Failed` is a single catch link fed by both error-gateway throw events (same multi-source link
pattern as `logicalDesign`). An unsupported `country` is a hard stop (IllegalArgumentException in the
TMF_STORE On Exit), not a `Failed` link.

## Process variables

Per the PUML, plus three added to satisfy the real PUT contract:

`_apiCallUrl`, `_order`, `version`, `crossProjectName`, `serviceCrossId`, `smwRoomId`, `country`,
`smwSystemId`, `serviceComponents` (List), `cpeCrossId`, `cpeNodeTypeDiscriminators` (List),
**`cpeName` (String)**, **`cpeStatusDiscriminator` (String)**, **`cpeCapacityFull` (Long)**,
`roomCrossId`, `roomCriteria`, `resultList` (List), `httpStatusCode` (Integer), `message`,
`errorMessage`, `response` (Map).

## Corrections vs. the original PUML (verified against `cross_5-0/cross_api`)

These were applied to both this design and the PUML (`7000_0400-update_cust_room_v1.0.puml`):

1. **CPE service component** is matched by `componentTypeName == "CPE"` (the PUML's `SC_CPE` was a
   flagged TODO; confirmed by the user).
2. **PUT `/v1/node` required fields.** `UpdateNodeValidator` requires `name`, `statusDiscriminator`
   and `capacityFull` — **not** `nodeTypeDiscriminators` as the PUML assumed. So `cross/findNode`
   reads `crossId,name,nodeTypes,statusRef,capacityFull` and `cross/updateCpeParent` resends
   `name` / `statusDiscriminator` / `capacityFull` / `nodeTypeDiscriminators` alongside the new
   `parentCrossIds`.
   - GET output field mapping: `name → $.name`, `statusDiscriminator → $.statusRef.id`,
     `capacityFull → $.capacityFull`, `nodeTypeDiscriminators → $.nodeTypes[*].id`
     (`statusRef`/`nodeTypes` serialize as `RefOutputDto` objects with an `id` field).
3. **Re-parent is a true move.** The PUT sets `parentCrossIds=[roomCrossId]` **and**
   `_meta_parentsImplicitRemove=true` (default is additive), so the CPE ends up with exactly the
   room as parent. The flag is hardcoded in the runtime-config source payload.
4. **Room search has no `systemId` query param.** The `GET /v1/node/` search endpoint reads
   `systemId` only as a criteria term, and the type term is `nodeTypeDiscriminator` (not
   `nodeType`). So `systemId` is folded into `roomCriteria` and the result list is `$.items`.

Out of scope (kept as a PUML `//TODO`, not a step): updating the service component that records the
customer placement/room with the newly assigned room.

## Automaton `runtime-config(gc-demo).js` changes

File: `GlobalConnect/gc/gc_docker_compose/files/210_automaton/init/runtime-config(gc-demo).js`

Add three `backend.sources` entries (all `crossApi`, basic auth like the others); `cross/findService`
already exists and is reused as-is:

- **`cross/findNode`** (generic GET node by crossId; the BPMN step is labelled `cross/findNode/CPE`,
  mirroring `logicalDesign`'s `cross/findService/SC_ACCESS` source/qualifier convention) —
  `GET /v1/node/#{dataMap['crossId']}?attributesToShow=crossId,name,nodeTypes,statusRef,capacityFull`;
  `mapResult`: `crossId:$.crossId`, `name:$.name`, `statusDiscriminator:$.statusRef.id`,
  `capacityFull:$.capacityFull`, `nodeTypeDiscriminators:$.nodeTypes[*].id`,
  `httpStatusCode:$.httpStatusCode`, `message:$.message`.
- **`cross/findRoom`** — `GET /v1/node/?attributesToShow=crossId&criteria=#{dataMap['criteria']}`;
  `mapResult`: `resultList:$.items`, `httpStatusCode:$.httpStatusCode`, `message:$.message`
  (the last two for error visibility, matching `cross/findNodeNames`/`cross/findLinkNames`).
- **`cross/updateCpeParent`** — `PUT /v1/node/#{dataMap['crossId']}?projectName=#{dataMap['projectName']}&attributesToShow=crossId`;
  payload: `name`, `statusDiscriminator`, `capacityFull`, `nodeTypeDiscriminators`,
  `parentCrossIds` (all templated from `dataMap`), and literal `_meta_parentsImplicitRemove: true`;
  `mapResult`: `httpStatusCode:$.httpStatusCode`, `message:$.message`.

Register the process in `backend.processes`:
`{ "id": "GC-AUTOMATION.updateCustomerRoom", "container": "GC-AUTOMATION_1.0.0-SNAPSHOT", "authorities": ["AUTOMATON_WF_READ"] }`.

## Responses

- COMPLETED: `{ version, status:"COMPLETED", serviceCrossId, cpeCrossId, roomCrossId, smwRoomId }`
- FAILED: `{ id:_order, version, status:"FAILED", error:errorMessage }`

## Authoring & validation notes

- Hand-author `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn` mirroring
  `logicalDesign.bpmn` structure: BPMN2 namespaces, `__<var>Item` itemDefinitions, `<bpmn2:property>`
  process vars, `drools:taskName="SdcApiCall"` tasks with per-task UUID-prefixed dataInput/dataOutput
  ids, `routeMap.*`/`dataMap.*` constant assignments, `_apiCallUrl`→`apiUrl`, `drools:onEntry-script`/
  `onExit-script` banners, exclusive gateways with inline-Java `conditionExpression`, and
  throw/catch `linkEventDefinition`s for `Upsert`/`Completed`/`Failed`.
- Do not redeclare process variables inside scripts/conditions (the "Duplicate local variable" trap).
- Include a valid `BPMNDiagram` layout so the kjar builds and Business Central can open it. The
  rendered SVG companion (`GC-AUTOMATION.updateCustomerRoom-svg.svg`) should be regenerated by
  re-saving through Business Central; a hand-authored SVG would only be a placeholder.
- Validate with `mvn -o clean install` (kie-maven-plugin compiles the inline Java and validates
  BPMN). Requires `cz.sykora:sdc-jbpm:1.0.0-SNAPSHOT` already in `~/.m2`.
