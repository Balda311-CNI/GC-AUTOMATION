# updateCustomerRoom Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a jBPM process `updateCustomerRoom` to the `GC-AUTOMATION` kjar that re-parents a service's CPE node to a target room (a `TECHNICAL_AREA` node identified by a Smallworld external id), plus the Automaton `runtime-config` sources it calls.

**Architecture:** A hand-authored BPMN2 process mirroring `logicalDesign.bpmn`: an `SdcApiCall` work-item task per external call, all FIND tasks before the UPDATE task, FIND segment ends by throwing the `Upsert` intermediate link, UPDATE segment begins at the `Upsert` catch. Errors set `errorMessage` and throw the `Failed` link. A final script assembles the synchronous `response` Map. Three new generic CROSS sources are added to Automaton's `runtime-config(gc-demo).js`.

**Tech Stack:** BPMN2 (jBPM Process Modeler dialect, `drools:` extensions), inline Java scripts (`language="http://www.java.com/java"`), `kie-maven-plugin` for validation, Automaton runtime-config (JS array of source definitions).

**Reference spec:** `docs/superpowers/specs/2026-06-15-update-customer-room-design.md`
**Reference PUML:** `globalconnect-project-documentation/api_de/sequence_diagrams/7000_0400-update_cust_room_v1.0.puml`
**Template to copy XML shapes from:** `src/main/resources/com/gc/gc_automation/logicalDesign.bpmn`

---

## Conventions used by every SdcApiCall task (read once)

Copy the `<bpmn2:task drools:taskName="SdcApiCall">` shape from `logicalDesign.bpmn` (a REST_API_CALL example lives at lines ~371–450; the TMF_STORE example at ~1884–2017). For each task in this plan:

- The task element id is the **UUID** given in the task. **Every** `dataInput`/`dataOutput`/`itemDefinition`/`inputSet`/`outputSet`/association id for that task is prefixed with that same UUID. Reusing a wrong/duplicate UUID fails the build — keep them consistent within a task and unique across tasks.
- Every task carries `apiUrl` (from process var `_apiCallUrl`) and a constant `TaskName="SdcApiCall"` input, exactly like the template.
- **Constant** inputs (e.g. `routeMap.domain="REST_API_CALL"`) use a `<bpmn2:dataInputAssociation>` with `<bpmn2:assignment><bpmn2:from><![CDATA[VALUE]]></bpmn2:from><bpmn2:to>…InputX</bpmn2:to></bpmn2:assignment>`.
- **Variable** inputs (e.g. `dataMap.serviceCrossId` ← `serviceCrossId`) use `<bpmn2:dataInputAssociation><bpmn2:sourceRef>VAR</bpmn2:sourceRef><bpmn2:targetRef>…InputX</bpmn2:targetRef></bpmn2:dataInputAssociation>`.
- **Outputs** use `<bpmn2:dataOutputAssociation><bpmn2:sourceRef>…OutputX</bpmn2:sourceRef><bpmn2:targetRef>VAR</bpmn2:targetRef></bpmn2:dataOutputAssociation>`.
- Each task gets a `<drools:onEntry-script>` banner (`System.out.println` of a "===" banner naming the step), matching the template. onExit scripts are given explicitly where needed.
- Do **not** redeclare a process variable inside any script or condition (e.g. never `String errorMessage = …`) — they are injected as typed locals; redeclaring fails the build with "Duplicate local variable".

Element UUIDs (use verbatim):

| node | id |
|---|---|
| StartEvent | `_C10F678B-1D88-4E6F-BB83-EF2E45529103` |
| TMF_STORE: get data | `_A5BE9CA5-73C3-402B-B50E-31739BB0CA5A` |
| REST_API_CALL: cross/findService | `_D25B668F-EE31-469E-9F12-F75D54EC088E` |
| Script: find cpeCrossId | `_308AF56D-8B1A-4B9D-81F1-D546F32D9087` |
| Gateway 1 (after find cpeCrossId) | `_C2586953-D211-4AEF-A761-82289F7B6FC6` |
| Throw "Failed" #1 | `_2D7AC52B-18C2-4558-BAA6-4CC538ED387D` |
| REST_API_CALL: cross/findNode/CPE | `_21E60016-0F71-4539-8824-BD9716CF2C7A` |
| REST_API_CALL: cross/findRoom | `_DE70A09C-D121-4102-BB26-7C89723EFE89` |
| Gateway 2 (after find room) | `_B3621A74-EE43-47B3-81BD-697E1F6D9B24` |
| Throw "Failed" #2 | `_EF59CBD4-ED4B-481F-A13A-B63677AA1282` |
| Throw "Upsert" | `_D95EE22A-59AD-49BB-B800-70B428BEFE43` |
| Catch "Upsert" | `_8158F9DD-358E-46F1-BC82-CE4928B23162` |
| REST_API_CALL: cross/updateCpeParent | `_2219AC2D-7D00-49DA-B3D2-86FFADE5B45C` |
| Throw "Completed" | `_AB09F9D9-AC47-4B6B-9DBE-8F12CEB69D65` |
| Catch "Completed" | `_204F9DB0-6969-48B3-83A3-88D788551BD9` |
| Script: Create result: COMPLETED | `_ECD39505-5686-4C54-B7B5-65076DBCDA3D` |
| Catch "Failed" | `_C69B199F-6320-4024-8BD9-FD889B6E8E94` |
| Script: Create result: FAILED | `_78DA6FC7-FD80-4183-9085-CFD5814B8D36` |
| EndEvent (after COMPLETED) | `_E2349E36-92BB-4A64-9495-1F8F71D24903` |
| EndEvent (after FAILED) | `_D0302EDE-27B8-4DE9-8316-DE42314CE329` |

linkEventDefinition ids (the throw's `<target>` must equal the catch's def id; the catch's `<source>` must equal the throw's def id):

| link | throw def id | catch def id |
|---|---|---|
| Upsert | `_C6B92E64-F3EA-4167-AA53-44B711360C91` | `_A85268AA-63B1-4072-9B6C-533D1E6EA8AC` |
| Completed | `_8F11B003-A717-44DA-8E2C-46CBAEB54CCC` | `_FE0EEE1B-17B6-427D-B2EC-C6C88F90A434` |
| Failed (catch only; two throws feed it) | — | `_83D47377-7053-492E-A68C-2CC0806B7E04` |

For the two `Failed` throw events give each its own `linkEventDefinition` id (generate two fresh UUIDs, e.g. `FAIL_THROW_1`, `FAIL_THROW_2`); the `Failed` catch lists **both** as `<bpmn2:source>` entries (multi-source catch, same as `logicalDesign.bpmn`).

---

## File Structure

- **Create:** `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn` — the process definition.
- **Modify:** `/home/jb/IdeaProjects/GlobalConnect/gc/gc_docker_compose/files/210_automaton/init/runtime-config(gc-demo).js` — add 3 `backend.sources` and 1 `backend.processes` entry.
- **Regenerate (manual, via Business Central):** `src/main/resources/com/gc/gc_automation/GC-AUTOMATION.updateCustomerRoom-svg.svg` — the diagram companion. Not authored by hand.

---

## Task 1: Add Automaton runtime-config sources + process registration

**Files:**
- Modify: `/home/jb/IdeaProjects/GlobalConnect/gc/gc_docker_compose/files/210_automaton/init/runtime-config(gc-demo).js`

- [ ] **Step 1: Add the three sources**

Insert these three objects into the `backend.sources` array (place them next to the other `cross/*` entries, e.g. right after the existing `cross/findProject` block). `cross/findService` already exists — do not duplicate it.

```js
{
  "name": "cross/findNode",
  "system": "crossApi",
  "baseUrl": "{{{gvarCrossApi.baseUrl}}}",
  "method": "GET",
  "uri": "/v1/node/#{dataMap['crossId']}?attributesToShow=crossId,name,nodeTypes,statusRef,capacityFull",
  "basicAuth": {
    "user": "{{{gvarCrossApi.basicAuth.user}}}",
    "password": "{{{gvarCrossApi.basicAuth.password}}}",
    "description": "{{{gvarCrossApi.basicAuth.description}}}"
  },
  "mapResult": {
    "crossId": "$.crossId",
    "name": "$.name",
    "statusDiscriminator": "$.statusRef.id",
    "capacityFull": "$.capacityFull",
    "nodeTypeDiscriminators": "$.nodeTypes[*].id",
    "httpStatusCode": "$.httpStatusCode",
    "message": "$.message"
  }
},
{
  "name": "cross/findRoom",
  "system": "crossApi",
  "baseUrl": "{{{gvarCrossApi.baseUrl}}}",
  "method": "GET",
  "uri": "/v1/node/?attributesToShow=crossId&criteria=#{dataMap['criteria']}",
  "basicAuth": {
    "user": "{{{gvarCrossApi.basicAuth.user}}}",
    "password": "{{{gvarCrossApi.basicAuth.password}}}",
    "description": "{{{gvarCrossApi.basicAuth.description}}}"
  },
  "mapResult": {
    "resultList": "$.items",
    "httpStatusCode": "$.httpStatusCode",
    "message": "$.message"
  }
},
{
  "name": "cross/updateCpeParent",
  "system": "crossApi",
  "baseUrl": "{{{gvarCrossApi.baseUrl}}}",
  "method": "PUT",
  "uri": "/v1/node/#{dataMap['crossId']}?projectName=#{dataMap['projectName']}&attributesToShow=crossId",
  "basicAuth": {
    "user": "{{{gvarCrossApi.basicAuth.user}}}",
    "password": "{{{gvarCrossApi.basicAuth.password}}}",
    "description": "{{{gvarCrossApi.basicAuth.description}}}"
  },
  "mapResult": {
    "crossId": "$.crossId",
    "httpStatusCode": "$.httpStatusCode",
    "message": "$.message"
  },
  "payload": {
    "name": "#{dataMap['name']}",
    "statusDiscriminator": "#{dataMap['statusDiscriminator']}",
    "capacityFull": "#{dataMap['capacityFull']}",
    "nodeTypeDiscriminators": "#{dataMap['nodeTypeDiscriminators']}",
    "parentCrossIds": "#{dataMap['parentCrossIds']}",
    "_meta_parentsImplicitRemove": true
  }
}
```

- [ ] **Step 2: Register the process**

Add this object to the `backend.processes` array (after the `GC-AUTOMATION.logicalDesign` entry):

```js
{
  "id": "GC-AUTOMATION.updateCustomerRoom",
  "container": "GC-AUTOMATION_1.0.0-SNAPSHOT",
  "authorities": [
    "AUTOMATON_WF_READ"
  ]
}
```

- [ ] **Step 3: Verify the file still parses as JavaScript**

Run: `node --check "/home/jb/IdeaProjects/GlobalConnect/gc/gc_docker_compose/files/210_automaton/init/runtime-config(gc-demo).js"`
Expected: no output, exit code 0 (syntax OK). If it errors, you most likely left a trailing/missing comma where the new objects were spliced in.

- [ ] **Step 4: Verify the new names are present exactly once each**

Run: `grep -c -E '"cross/findNode"|"cross/findRoom"|"cross/updateCpeParent"|"GC-AUTOMATION.updateCustomerRoom"' "/home/jb/IdeaProjects/GlobalConnect/gc/gc_docker_compose/files/210_automaton/init/runtime-config(gc-demo).js"`
Expected: `4`

---

## Task 2: BPMN skeleton — root, itemDefinitions, process header, process variables

**Files:**
- Create: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

- [ ] **Step 1: Write the document root + global itemDefinitions + process header + properties + start event**

Copy the root `<bpmn2:definitions …>` element (namespaces + `xsi:schemaLocation` + `exporter`) verbatim from `logicalDesign.bpmn:1-2`, but give the root a fresh `id` (any `_<UUID>`). Then author the process. Use this exact process header:

```xml
<bpmn2:process id="GC-AUTOMATION.updateCustomerRoom" drools:packageName="com.gc.gc_automation" drools:version="1.0" drools:adHoc="false" name="updateCustomerRoom" isExecutable="true" processType="Public">
  <bpmn2:extensionElements>
    <drools:global identifier="_apiCallUrl" type="String"/>
  </bpmn2:extensionElements>
```

Declare one global `<bpmn2:itemDefinition>` per variable type and one `<bpmn2:property>` per process variable. Variables and their `structureRef` types:

| variable | structureRef |
|---|---|
| `_apiCallUrl` | `String` |
| `_order` | `String` |
| `version` | `String` |
| `crossProjectName` | `String` |
| `serviceCrossId` | `String` |
| `smwRoomId` | `String` |
| `country` | `String` |
| `smwSystemId` | `String` |
| `serviceComponents` | `java.util.List` |
| `cpeCrossId` | `String` |
| `cpeNodeTypeDiscriminators` | `java.util.List` |
| `cpeName` | `String` |
| `cpeStatusDiscriminator` | `String` |
| `cpeCapacityFull` | `Long` |
| `roomCrossId` | `String` |
| `roomCriteria` | `String` |
| `resultList` | `java.util.List` |
| `httpStatusCode` | `Integer` |
| `message` | `String` |
| `errorMessage` | `String` |
| `response` | `java.util.Map` |

Follow the template's id pattern: itemDefinition `id="_<var>Item"` (and `__apiCallUrlItem`/`__orderItem` for the underscore-prefixed ones, exactly as `logicalDesign.bpmn:166-210` does), property `id="<var>"` `itemSubjectRef="_<var>Item"` `name="<var>"`.

Add the start event and leave the rest of the process body to later tasks:

```xml
<bpmn2:startEvent id="_C10F678B-1D88-4E6F-BB83-EF2E45529103">
  <bpmn2:outgoing>flow_start_tmf</bpmn2:outgoing>
</bpmn2:startEvent>
```

(Use readable sequenceFlow ids like `flow_start_tmf`; any unique string is fine. They are defined as `<bpmn2:sequenceFlow id=… sourceRef=… targetRef=…/>` and referenced by `<bpmn2:incoming>`/`<bpmn2:outgoing>` on each node.)

Close `</bpmn2:process>` and `</bpmn2:definitions>` for now so the file is well-formed; later tasks insert nodes before `</bpmn2:process>`.

- [ ] **Step 2: Verify well-formed XML**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

- [ ] **Step 3: Verify all 21 properties declared**

Run: `grep -c '<bpmn2:property ' src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`
Expected: `21`

---

## Task 3: TMF_STORE: get data task

**Files:**
- Modify: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

- [ ] **Step 1: Insert the TMF_STORE task** (id `_A5BE9CA5-73C3-402B-B50E-31739BB0CA5A`, name `TMF_STORE:\nget data`)

Copy the TMF_STORE task shape from `logicalDesign.bpmn:1884-2017`. Wire incoming = `flow_start_tmf`, outgoing = `flow_tmf_findservice`.

Inputs (constants unless marked ←var):
| input name | value |
|---|---|
| `apiUrl` | ← `_apiCallUrl` |
| `routeMap.domain` | `TMF_STORE` |
| `routeMap.operation` | `GET` |
| `routeMap.transformation` | `JSONATA` |
| `dataMap.id` | ← `_order` |
| `dataMap.jsonata.version` | `version` |
| `dataMap.jsonata.crossProjectName` | `crossProjectName` |
| `dataMap.jsonata.serviceCrossId` | `serviceCrossId` |
| `dataMap.jsonata.smwRoomId` | `smwRoomId` |
| `dataMap.jsonata.country` | `country` |
| `TaskName` | `SdcApiCall` |

Outputs (→ process var):
| output name | → var |
|---|---|
| `version` | `version` |
| `crossProjectName` | `crossProjectName` |
| `serviceCrossId` | `serviceCrossId` |
| `smwRoomId` | `smwRoomId` |
| `country` | `country` |

onEntry-script:
```java
System.out.println("================================================");
System.out.println("updateCustomerRoom - TMF_STORE: get data");
System.out.println("================================================");
```

onExit-script (resolves SMW systemId from country; hard stop on unsupported country):
```java
System.out.println("Data Outputs: version=" + version + ", crossProjectName=" + crossProjectName + ", serviceCrossId=" + serviceCrossId + ", smwRoomId=" + smwRoomId + ", country=" + country);
if ( "DE".equalsIgnoreCase(country) ) {
    kcontext.setVariable("smwSystemId", "PNI_DE");
} else {
    throw new IllegalArgumentException("Unsupported country '" + country + "' - only 'DE' (Germany) is supported for now");
}
```

- [ ] **Step 2: Verify well-formed**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

---

## Task 4: cross/findService task (canonical SdcApiCall pattern)

**Files:**
- Modify: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

This is the simplest REST task — use the full XML below as the canonical pattern for Tasks 6, 7 and 8 (swap the UUID, name, inputs, and outputs).

- [ ] **Step 1: Insert the findService task** (id `_D25B668F-EE31-469E-9F12-F75D54EC088E`, name `REST_API_CALL:\ncross/findService`). Incoming = `flow_tmf_findservice`, outgoing = `flow_findservice_findcpe`.

```xml
<bpmn2:task id="_D25B668F-EE31-469E-9F12-F75D54EC088E" drools:taskName="SdcApiCall" name="REST_API_CALL:&#10;cross/findService">
  <bpmn2:documentation><![CDATA[Retrieve service serviceCrossId via CROSS API. (FIND)]]></bpmn2:documentation>
  <bpmn2:extensionElements>
    <drools:metaData name="elementname"><drools:metaValue><![CDATA[REST_API_CALL:
cross/findService]]></drools:metaValue></drools:metaData>
    <drools:onEntry-script scriptFormat="http://www.java.com/java"><drools:script><![CDATA[System.out.println("================================================");
System.out.println("updateCustomerRoom - REST_API_CALL: cross/findService");
System.out.println("================================================");]]></drools:script></drools:onEntry-script>
  </bpmn2:extensionElements>
  <bpmn2:incoming>flow_tmf_findservice</bpmn2:incoming>
  <bpmn2:outgoing>flow_findservice_findcpe</bpmn2:outgoing>
  <bpmn2:ioSpecification>
    <bpmn2:dataInput id="_D25B668F-EE31-469E-9F12-F75D54EC088E_apiUrlInputX" drools:dtype="String" itemSubjectRef="__D25B668F-EE31-469E-9F12-F75D54EC088E_apiUrlInputXItem" name="apiUrl"/>
    <bpmn2:dataInput id="_D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.domainInputX" drools:dtype="String" itemSubjectRef="__D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.domainInputXItem" name="routeMap.domain"/>
    <bpmn2:dataInput id="_D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.sourceInputX" drools:dtype="String" itemSubjectRef="__D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.sourceInputXItem" name="routeMap.source"/>
    <bpmn2:dataInput id="_D25B668F-EE31-469E-9F12-F75D54EC088E_dataMap.serviceCrossIdInputX" drools:dtype="String" itemSubjectRef="__D25B668F-EE31-469E-9F12-F75D54EC088E_dataMap.serviceCrossIdInputXItem" name="dataMap.serviceCrossId"/>
    <bpmn2:dataInput id="_D25B668F-EE31-469E-9F12-F75D54EC088E_TaskNameInputX" drools:dtype="Object" name="TaskName"/>
    <bpmn2:dataOutput id="_D25B668F-EE31-469E-9F12-F75D54EC088E_httpStatusCodeOutputX" drools:dtype="Integer" itemSubjectRef="__D25B668F-EE31-469E-9F12-F75D54EC088E_httpStatusCodeOutputXItem" name="httpStatusCode"/>
    <bpmn2:dataOutput id="_D25B668F-EE31-469E-9F12-F75D54EC088E_messageOutputX" drools:dtype="String" itemSubjectRef="__D25B668F-EE31-469E-9F12-F75D54EC088E_messageOutputXItem" name="message"/>
    <bpmn2:dataOutput id="_D25B668F-EE31-469E-9F12-F75D54EC088E_serviceComponentsOutputX" drools:dtype="java.util.List" itemSubjectRef="__D25B668F-EE31-469E-9F12-F75D54EC088E_serviceComponentsOutputXItem" name="serviceComponents"/>
    <bpmn2:inputSet>
      <bpmn2:dataInputRefs>_D25B668F-EE31-469E-9F12-F75D54EC088E_apiUrlInputX</bpmn2:dataInputRefs>
      <bpmn2:dataInputRefs>_D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.domainInputX</bpmn2:dataInputRefs>
      <bpmn2:dataInputRefs>_D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.sourceInputX</bpmn2:dataInputRefs>
      <bpmn2:dataInputRefs>_D25B668F-EE31-469E-9F12-F75D54EC088E_dataMap.serviceCrossIdInputX</bpmn2:dataInputRefs>
      <bpmn2:dataInputRefs>_D25B668F-EE31-469E-9F12-F75D54EC088E_TaskNameInputX</bpmn2:dataInputRefs>
    </bpmn2:inputSet>
    <bpmn2:outputSet>
      <bpmn2:dataOutputRefs>_D25B668F-EE31-469E-9F12-F75D54EC088E_httpStatusCodeOutputX</bpmn2:dataOutputRefs>
      <bpmn2:dataOutputRefs>_D25B668F-EE31-469E-9F12-F75D54EC088E_messageOutputX</bpmn2:dataOutputRefs>
      <bpmn2:dataOutputRefs>_D25B668F-EE31-469E-9F12-F75D54EC088E_serviceComponentsOutputX</bpmn2:dataOutputRefs>
    </bpmn2:outputSet>
  </bpmn2:ioSpecification>
  <bpmn2:dataInputAssociation>
    <bpmn2:sourceRef>_apiCallUrl</bpmn2:sourceRef>
    <bpmn2:targetRef>_D25B668F-EE31-469E-9F12-F75D54EC088E_apiUrlInputX</bpmn2:targetRef>
  </bpmn2:dataInputAssociation>
  <bpmn2:dataInputAssociation>
    <bpmn2:targetRef>_D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.domainInputX</bpmn2:targetRef>
    <bpmn2:assignment><bpmn2:from xsi:type="bpmn2:tFormalExpression"><![CDATA[REST_API_CALL]]></bpmn2:from><bpmn2:to xsi:type="bpmn2:tFormalExpression">_D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.domainInputX</bpmn2:to></bpmn2:assignment>
  </bpmn2:dataInputAssociation>
  <bpmn2:dataInputAssociation>
    <bpmn2:targetRef>_D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.sourceInputX</bpmn2:targetRef>
    <bpmn2:assignment><bpmn2:from xsi:type="bpmn2:tFormalExpression"><![CDATA[cross/findService]]></bpmn2:from><bpmn2:to xsi:type="bpmn2:tFormalExpression">_D25B668F-EE31-469E-9F12-F75D54EC088E_routeMap.sourceInputX</bpmn2:to></bpmn2:assignment>
  </bpmn2:dataInputAssociation>
  <bpmn2:dataInputAssociation>
    <bpmn2:sourceRef>serviceCrossId</bpmn2:sourceRef>
    <bpmn2:targetRef>_D25B668F-EE31-469E-9F12-F75D54EC088E_dataMap.serviceCrossIdInputX</bpmn2:targetRef>
  </bpmn2:dataInputAssociation>
  <bpmn2:dataInputAssociation>
    <bpmn2:targetRef>_D25B668F-EE31-469E-9F12-F75D54EC088E_TaskNameInputX</bpmn2:targetRef>
    <bpmn2:assignment><bpmn2:from xsi:type="bpmn2:tFormalExpression"><![CDATA[SdcApiCall]]></bpmn2:from><bpmn2:to xsi:type="bpmn2:tFormalExpression">_D25B668F-EE31-469E-9F12-F75D54EC088E_TaskNameInputX</bpmn2:to></bpmn2:assignment>
  </bpmn2:dataInputAssociation>
  <bpmn2:dataOutputAssociation>
    <bpmn2:sourceRef>_D25B668F-EE31-469E-9F12-F75D54EC088E_httpStatusCodeOutputX</bpmn2:sourceRef>
    <bpmn2:targetRef>httpStatusCode</bpmn2:targetRef>
  </bpmn2:dataOutputAssociation>
  <bpmn2:dataOutputAssociation>
    <bpmn2:sourceRef>_D25B668F-EE31-469E-9F12-F75D54EC088E_messageOutputX</bpmn2:sourceRef>
    <bpmn2:targetRef>message</bpmn2:targetRef>
  </bpmn2:dataOutputAssociation>
  <bpmn2:dataOutputAssociation>
    <bpmn2:sourceRef>_D25B668F-EE31-469E-9F12-F75D54EC088E_serviceComponentsOutputX</bpmn2:sourceRef>
    <bpmn2:targetRef>serviceComponents</bpmn2:targetRef>
  </bpmn2:dataOutputAssociation>
</bpmn2:task>
```

Also declare the per-task itemDefinitions (one per Input/Output, in the global itemDefinitions block at the top) following the `__<uuid>_<name>InputXItem` / `OutputXItem` pattern from `logicalDesign.bpmn`. The `TaskName` input has `drools:dtype="Object"` and **no** itemSubjectRef/itemDefinition (matches the template).

- [ ] **Step 2: Verify well-formed**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

---

## Task 5: find cpeCrossId script + Gateway 1 + Failed throw #1

**Files:**
- Modify: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

- [ ] **Step 1: Insert the scriptTask** (id `_308AF56D-8B1A-4B9D-81F1-D546F32D9087`, name `find cpeCrossId`, `scriptFormat="http://www.java.com/java"`). Incoming = `flow_findservice_findcpe`, outgoing = `flow_findcpe_gw1`. Copy the `<bpmn2:scriptTask>` shape from `logicalDesign.bpmn` (e.g. ~739-753). Script body:

```java
System.out.println("================================================");
System.out.println("updateCustomerRoom - find cpeCrossId");
System.out.println("================================================");

boolean found = false;
if ( serviceComponents != null ) {
    for ( java.util.Map serviceComponent : (java.util.List<java.util.Map>) serviceComponents ) {
        if ( "CPE".equals( serviceComponent.get("componentTypeName") ) ) {
            java.util.List resources = (java.util.List) serviceComponent.get("resources");
            if ( resources == null || resources.isEmpty() ) {
                kcontext.setVariable("errorMessage", "Service component CPE has no resources");
                return;
            }
            java.util.Map resource = (java.util.Map) resources.get(0);
            for ( java.util.Object o : resources ) {
                java.util.Map r = (java.util.Map) o;
                if ( Boolean.TRUE.equals( r.get("primary") ) ) { resource = r; }
            }
            java.util.Map resourceRef = (java.util.Map) resource.get("resourceRef");
            kcontext.setVariable("cpeCrossId", (String) resourceRef.get("id"));
            found = true;
        }
    }
}
if ( !found ) {
    kcontext.setVariable("errorMessage", "Unable to find the CPE service component at the given service");
}
System.out.println("Data Outputs: cpeCrossId=" + kcontext.getVariable("cpeCrossId") + ", errorMessage=" + kcontext.getVariable("errorMessage"));
```

- [ ] **Step 2: Insert Gateway 1** (id `_C2586953-D211-4AEF-A761-82289F7B6FC6`, `gatewayDirection="Diverging"`). Incoming = `flow_findcpe_gw1`. Two outgoing flows; the **default** is the happy path:

```xml
<bpmn2:exclusiveGateway id="_C2586953-D211-4AEF-A761-82289F7B6FC6" gatewayDirection="Diverging" default="flow_gw1_findnode">
  <bpmn2:incoming>flow_findcpe_gw1</bpmn2:incoming>
  <bpmn2:outgoing>flow_gw1_fail</bpmn2:outgoing>
  <bpmn2:outgoing>flow_gw1_findnode</bpmn2:outgoing>
</bpmn2:exclusiveGateway>
```

Conditional flow to the Failed throw (`flow_gw1_fail`, target `_2D7AC52B-18C2-4558-BAA6-4CC538ED387D`):
```xml
<bpmn2:sequenceFlow id="flow_gw1_fail" name="Error" sourceRef="_C2586953-D211-4AEF-A761-82289F7B6FC6" targetRef="_2D7AC52B-18C2-4558-BAA6-4CC538ED387D">
  <bpmn2:conditionExpression xsi:type="bpmn2:tFormalExpression" language="http://www.java.com/java"><![CDATA[boolean result = errorMessage != null;
System.out.println("VALIDATION: find cpeCrossId failed=" + result + ", errorMessage=" + errorMessage);
return result;]]></bpmn2:conditionExpression>
</bpmn2:sequenceFlow>
```
Default flow `flow_gw1_findnode` (no condition), target = `_21E60016-0F71-4539-8824-BD9716CF2C7A` (findNode).

- [ ] **Step 3: Insert Failed throw #1** (id `_2D7AC52B-18C2-4558-BAA6-4CC538ED387D`, name `Failed`). Incoming = `flow_gw1_fail`. Use a fresh linkEventDefinition id (call it `FAIL_THROW_1`), `name="Failed"`, with `<bpmn2:target>_83D47377-7053-492E-A68C-2CC0806B7E04</bpmn2:target>` (the Failed catch def id):

```xml
<bpmn2:intermediateThrowEvent id="_2D7AC52B-18C2-4558-BAA6-4CC538ED387D" name="Failed">
  <bpmn2:extensionElements><drools:metaData name="elementname"><drools:metaValue><![CDATA[Failed]]></drools:metaValue></drools:metaData></bpmn2:extensionElements>
  <bpmn2:incoming>flow_gw1_fail</bpmn2:incoming>
  <bpmn2:linkEventDefinition id="FAIL_THROW_1" name="Failed"><bpmn2:target>_83D47377-7053-492E-A68C-2CC0806B7E04</bpmn2:target></bpmn2:linkEventDefinition>
</bpmn2:intermediateThrowEvent>
```
(Replace `FAIL_THROW_1` with a real fresh `_<UUID>`.)

- [ ] **Step 4: Verify well-formed**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

---

## Task 6: cross/findNode/CPE task

**Files:**
- Modify: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

- [ ] **Step 1: Insert the task** (id `_21E60016-0F71-4539-8824-BD9716CF2C7A`, name `REST_API_CALL:\ncross/findNode/CPE`). Incoming = `flow_gw1_findnode`, outgoing = `flow_findnode_findroom`. Use the Task 4 canonical XML, changing UUID/name and these inputs/outputs:

Inputs:
| input name | value |
|---|---|
| `apiUrl` | ← `_apiCallUrl` |
| `routeMap.domain` | `REST_API_CALL` |
| `routeMap.source` | `cross/findNode` |
| `dataMap.crossId` | ← `cpeCrossId` |
| `TaskName` | `SdcApiCall` |

Outputs (→ var). The output **names must exactly match the `mapResult` keys** from Task 1's `cross/findNode` source:
| output name | drools:dtype | → var |
|---|---|---|
| `httpStatusCode` | `Integer` | `httpStatusCode` |
| `message` | `String` | `message` |
| `name` | `String` | `cpeName` |
| `statusDiscriminator` | `String` | `cpeStatusDiscriminator` |
| `capacityFull` | `Long` | `cpeCapacityFull` |
| `nodeTypeDiscriminators` | `java.util.List` | `cpeNodeTypeDiscriminators` |

onEntry banner: `"updateCustomerRoom - REST_API_CALL: cross/findNode/CPE"`.
The `elementname` metaValue is `REST_API_CALL:\ncross/findNode/CPE`.

- [ ] **Step 2: Verify well-formed**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

---

## Task 7: cross/findRoom task + Gateway 2 + Upsert throw + Failed throw #2

**Files:**
- Modify: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

- [ ] **Step 1: Insert the findRoom task** (id `_DE70A09C-D121-4102-BB26-7C89723EFE89`, name `REST_API_CALL:\ncross/findRoom`). Incoming = `flow_findnode_findroom`, outgoing = `flow_findroom_gw2`. Use the Task 4 canonical XML with these inputs/outputs and the two scripts.

Inputs:
| input name | value |
|---|---|
| `apiUrl` | ← `_apiCallUrl` |
| `routeMap.domain` | `REST_API_CALL` |
| `routeMap.source` | `cross/findRoom` |
| `dataMap.criteria` | ← `roomCriteria` |
| `TaskName` | `SdcApiCall` |

Outputs:
| output name | drools:dtype | → var |
|---|---|---|
| `resultList` | `java.util.List` | `resultList` |

onEntry-script (banner + build criteria; note `systemId` is a criteria term, not a query param):
```java
System.out.println("================================================");
System.out.println("updateCustomerRoom - REST_API_CALL: cross/findRoom");
System.out.println("================================================");
kcontext.setVariable("resultList", null);
kcontext.setVariable("roomCriteria", "systemId==" + smwSystemId + ";externalId==" + smwRoomId + ";nodeTypeDiscriminator==TECHNICAL_AREA");
System.out.println("roomCriteria=" + kcontext.getVariable("roomCriteria"));
```

onExit-script (validate + extract roomCrossId):
```java
System.out.println("Data Outputs: resultList=" + resultList);
if ( resultList == null || ((java.util.List) resultList).isEmpty() ) {
    kcontext.setVariable("errorMessage", "Room not found for smwRoomId '" + smwRoomId + "' (systemId " + smwSystemId + ")");
} else if ( ((java.util.List) resultList).size() > 1 ) {
    kcontext.setVariable("errorMessage", "Room not unique for smwRoomId '" + smwRoomId + "' (systemId " + smwSystemId + ")");
} else {
    java.util.Map firstRow = (java.util.Map) ((java.util.List) resultList).get(0);
    kcontext.setVariable("roomCrossId", (String) firstRow.get("crossId"));
    System.out.println("roomCrossId=" + kcontext.getVariable("roomCrossId"));
}
```

- [ ] **Step 2: Insert Gateway 2** (id `_B3621A74-EE43-47B3-81BD-697E1F6D9B24`). Incoming = `flow_findroom_gw2`. Default flow throws Upsert; conditional flow throws Failed:

```xml
<bpmn2:exclusiveGateway id="_B3621A74-EE43-47B3-81BD-697E1F6D9B24" gatewayDirection="Diverging" default="flow_gw2_upsert">
  <bpmn2:incoming>flow_findroom_gw2</bpmn2:incoming>
  <bpmn2:outgoing>flow_gw2_fail</bpmn2:outgoing>
  <bpmn2:outgoing>flow_gw2_upsert</bpmn2:outgoing>
</bpmn2:exclusiveGateway>
```
Conditional `flow_gw2_fail` (target `_EF59CBD4-ED4B-481F-A13A-B63677AA1282`) condition:
```xml
<bpmn2:conditionExpression xsi:type="bpmn2:tFormalExpression" language="http://www.java.com/java"><![CDATA[boolean result = errorMessage != null;
System.out.println("VALIDATION: find room failed=" + result + ", errorMessage=" + errorMessage);
return result;]]></bpmn2:conditionExpression>
```
Default `flow_gw2_upsert` target = `_D95EE22A-59AD-49BB-B800-70B428BEFE43` (Upsert throw).

- [ ] **Step 3: Insert Upsert throw** (id `_D95EE22A-59AD-49BB-B800-70B428BEFE43`, name `Upsert`). Incoming = `flow_gw2_upsert`. linkEventDefinition id `_C6B92E64-F3EA-4167-AA53-44B711360C91`, `<target>_A85268AA-63B1-4072-9B6C-533D1E6EA8AC</target>`.

- [ ] **Step 4: Insert Failed throw #2** (id `_EF59CBD4-ED4B-481F-A13A-B63677AA1282`, name `Failed`). Incoming = `flow_gw2_fail`. Fresh linkEventDefinition id (`FAIL_THROW_2` → real UUID), `name="Failed"`, `<target>_83D47377-7053-492E-A68C-2CC0806B7E04</target>`.

- [ ] **Step 5: Verify well-formed**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

---

## Task 8: Upsert catch → cross/updateCpeParent → Completed throw

**Files:**
- Modify: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

- [ ] **Step 1: Insert the Upsert catch** (id `_8158F9DD-358E-46F1-BC82-CE4928B23162`, name `Upsert`). Outgoing = `flow_upsert_update`. linkEventDefinition id `_A85268AA-63B1-4072-9B6C-533D1E6EA8AC`, `name="Upsert"`, `<bpmn2:source>_C6B92E64-F3EA-4167-AA53-44B711360C91</bpmn2:source>`.

```xml
<bpmn2:intermediateCatchEvent id="_8158F9DD-358E-46F1-BC82-CE4928B23162" name="Upsert">
  <bpmn2:extensionElements><drools:metaData name="elementname"><drools:metaValue><![CDATA[Upsert]]></drools:metaValue></drools:metaData></bpmn2:extensionElements>
  <bpmn2:outgoing>flow_upsert_update</bpmn2:outgoing>
  <bpmn2:linkEventDefinition id="_A85268AA-63B1-4072-9B6C-533D1E6EA8AC" name="Upsert"><bpmn2:source>_C6B92E64-F3EA-4167-AA53-44B711360C91</bpmn2:source></bpmn2:linkEventDefinition>
</bpmn2:intermediateCatchEvent>
```

- [ ] **Step 2: Insert the updateCpeParent task** (id `_2219AC2D-7D00-49DA-B3D2-86FFADE5B45C`, name `REST_API_CALL:\ncross/updateCpeParent`). Incoming = `flow_upsert_update`, outgoing = `flow_update_completed`. Use the Task 4 canonical XML with these inputs/outputs and an onEntry script that builds `parentCrossIds`.

onEntry-script:
```java
System.out.println("================================================");
System.out.println("updateCustomerRoom - REST_API_CALL: cross/updateCpeParent");
System.out.println("================================================");
java.util.List parentCrossIds = new java.util.ArrayList();
parentCrossIds.add( roomCrossId );
kcontext.setVariable("parentCrossIds", parentCrossIds);
```
Add a process variable + itemDefinition `parentCrossIds` (`java.util.List`) — append it to the property/itemDefinition block from Task 2 (so the total property count becomes 22).

Inputs:
| input name | value |
|---|---|
| `apiUrl` | ← `_apiCallUrl` |
| `routeMap.domain` | `REST_API_CALL` |
| `routeMap.source` | `cross/updateCpeParent` |
| `dataMap.projectName` | ← `crossProjectName` |
| `dataMap.crossId` | ← `cpeCrossId` |
| `dataMap.parentCrossIds` | ← `parentCrossIds` |
| `dataMap.name` | ← `cpeName` |
| `dataMap.statusDiscriminator` | ← `cpeStatusDiscriminator` |
| `dataMap.capacityFull` | ← `cpeCapacityFull` |
| `dataMap.nodeTypeDiscriminators` | ← `cpeNodeTypeDiscriminators` |
| `TaskName` | `SdcApiCall` |

Outputs:
| output name | drools:dtype | → var |
|---|---|---|
| `httpStatusCode` | `Integer` | `httpStatusCode` |
| `message` | `String` | `message` |

(`dataMap.parentCrossIds`, `dataMap.nodeTypeDiscriminators` and `dataMap.capacityFull` are variable refs, NOT constants — use the `<sourceRef>VAR</sourceRef>` association form. Their per-task itemDefinitions use `structureRef="java.util.List"` / `"Long"` respectively.)

- [ ] **Step 3: Insert the Completed throw** (id `_AB09F9D9-AC47-4B6B-9DBE-8F12CEB69D65`, name `Completed`). Incoming = `flow_update_completed`. linkEventDefinition id `_8F11B003-A717-44DA-8E2C-46CBAEB54CCC`, `<target>_FE0EEE1B-17B6-427D-B2EC-C6C88F90A434</target>`.

- [ ] **Step 4: Verify well-formed**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

---

## Task 9: Result scripts (COMPLETED + FAILED) and end events

**Files:**
- Modify: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

- [ ] **Step 1: Insert the Completed catch** (id `_204F9DB0-6969-48B3-83A3-88D788551BD9`, name `Completed`). Outgoing = `flow_completed_resultok`. linkEventDefinition id `_FE0EEE1B-17B6-427D-B2EC-C6C88F90A434`, `<source>_8F11B003-A717-44DA-8E2C-46CBAEB54CCC</source>`.

- [ ] **Step 2: Insert the Create result: COMPLETED scriptTask** (id `_ECD39505-5686-4C54-B7B5-65076DBCDA3D`, name `Create result:\nCOMPLETED`). Incoming = `flow_completed_resultok`, outgoing = `flow_resultok_end`. Script:

```java
System.out.println("================================================");
System.out.println("updateCustomerRoom - Create result: COMPLETED");
System.out.println("================================================");

java.util.Map<String, Object> map = new java.util.HashMap<String, Object>();
map.put("version", kcontext.getVariable("version"));
map.put("status", "COMPLETED");
map.put("serviceCrossId", kcontext.getVariable("serviceCrossId"));
map.put("cpeCrossId", kcontext.getVariable("cpeCrossId"));
map.put("roomCrossId", kcontext.getVariable("roomCrossId"));
map.put("smwRoomId", kcontext.getVariable("smwRoomId"));
kcontext.setVariable("response", map);
```

- [ ] **Step 3: Insert the Failed catch** (id `_C69B199F-6320-4024-8BD9-FD889B6E8E94`, name `Failed`). Outgoing = `flow_failed_resultfail`. linkEventDefinition id `_83D47377-7053-492E-A68C-2CC0806B7E04`, `name="Failed"`, with **both** throw def ids as sources:
```xml
<bpmn2:linkEventDefinition id="_83D47377-7053-492E-A68C-2CC0806B7E04" name="Failed">
  <bpmn2:source>FAIL_THROW_1</bpmn2:source>
  <bpmn2:source>FAIL_THROW_2</bpmn2:source>
</bpmn2:linkEventDefinition>
```
(Replace `FAIL_THROW_1`/`FAIL_THROW_2` with the real UUIDs you generated in Tasks 5 and 7.)

- [ ] **Step 4: Insert the Create result: FAILED scriptTask** (id `_78DA6FC7-FD80-4183-9085-CFD5814B8D36`, name `Create result:\nFAILED`). Incoming = `flow_failed_resultfail`, outgoing = `flow_resultfail_end`. Script:

```java
System.out.println("================================================");
System.out.println("updateCustomerRoom - Create result: FAILED");
System.out.println("================================================");

if ( errorMessage == null ) {
    throw new IllegalArgumentException("errorMessage cannot be null. Set it by: kcontext.setVariable(\"errorMessage\", \"MESSAGE\");");
}
java.util.Map<String, Object> map = new java.util.HashMap<String, Object>();
map.put("id", kcontext.getVariable("_order"));
map.put("version", kcontext.getVariable("version"));
map.put("status", "FAILED");
map.put("error", errorMessage);
kcontext.setVariable("response", map);
```

- [ ] **Step 5: Insert two end events**
```xml
<bpmn2:endEvent id="_E2349E36-92BB-4A64-9495-1F8F71D24903"><bpmn2:incoming>flow_resultok_end</bpmn2:incoming></bpmn2:endEvent>
<bpmn2:endEvent id="_D0302EDE-27B8-4DE9-8316-DE42314CE329"><bpmn2:incoming>flow_resultfail_end</bpmn2:incoming></bpmn2:endEvent>
```

- [ ] **Step 6: Add all sequenceFlow elements** referenced by `incoming`/`outgoing` above but not yet defined (the plain ones with no condition): `flow_start_tmf`, `flow_tmf_findservice`, `flow_findservice_findcpe`, `flow_findcpe_gw1`, `flow_gw1_findnode`, `flow_findnode_findroom`, `flow_findroom_gw2`, `flow_gw2_upsert`, `flow_upsert_update`, `flow_update_completed`, `flow_completed_resultok`, `flow_failed_resultfail`, `flow_resultok_end`, `flow_resultfail_end`. Each: `<bpmn2:sequenceFlow id="…" sourceRef="…" targetRef="…"/>`. (The conditional flows `flow_gw1_fail`, `flow_gw2_fail` were already added in Tasks 5/7.)

- [ ] **Step 7: Verify well-formed + flow integrity**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

Run (every incoming/outgoing flow id must be defined as a sequenceFlow):
```bash
cd /home/jb/jBPM/GC/GC-AUTOMATION && python3 - <<'PY'
import re
x=open('src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn').read()
defined=set(re.findall(r'<bpmn2:sequenceFlow id="([^"]+)"',x))
refd=set(re.findall(r'<bpmn2:(?:in|out)going>([^<]+)</bpmn2:(?:in|out)going>',x))
print("undefined refs:", refd-defined)
print("unused defs:", defined-refd)
PY
```
Expected: both sets empty (`set()`).

---

## Task 10: BPMNDiagram layout

**Files:**
- Modify: `src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn`

- [ ] **Step 1: Append a `<bpmndi:BPMNDiagram>` section** before `</bpmn2:definitions>`. For each node add a `<bpmndi:BPMNShape bpmnElement="<nodeId>"><dc:Bounds .../></bpmndi:BPMNShape>`; for each sequenceFlow add a `<bpmndi:BPMNEdge bpmnElement="<flowId>">` with two `<di:waypoint>`s. Lay the happy path top-to-bottom at x≈400, y stepping by 150 (start 100, tmf 250, findservice 400, findcpe 550, gw1 700, findnode 850, findroom 1000, gw2 1150, upsert-throw 1300; then update 1500, completed-throw 1650; catches/results to the side at x≈650). Exact coordinates are not important — Business Central will re-layout on save. Tasks use `height="102" width="154"`, events `height="56" width="56"`, gateways `height="56" width="56"`.

- [ ] **Step 2: Verify well-formed**

Run: `xmllint --noout src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn && echo OK`
Expected: `OK`

---

## Task 11: Build & validate the kjar

**Files:** none (build only)

- [ ] **Step 1: UUID-consistency sanity check** (each task's input/output ids must reference an itemDefinition that exists). Quick check that every `itemSubjectRef` used resolves to a declared itemDefinition id:
```bash
cd /home/jb/jBPM/GC/GC-AUTOMATION && python3 - <<'PY'
import re
x=open('src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn').read()
defs=set(re.findall(r'<bpmn2:itemDefinition id="([^"]+)"',x))
refs=set(re.findall(r'itemSubjectRef="([^"]+)"',x))
print("missing itemDefinitions:", refs-defs)
PY
```
Expected: `missing itemDefinitions: set()`

- [ ] **Step 2: Run the build**

Run: `cd /home/jb/jBPM/GC/GC-AUTOMATION && mvn -o clean install`
Expected: `BUILD SUCCESS`, producing `target/GC-AUTOMATION-1.0.0-SNAPSHOT.jar`.

Notes on reading the output:
- Ignore pre-existing non-fatal parser noise from the other files (`shape_null` shapes in pniDesign/lniDesign/resourceAvailabilityCheck, a duplicate `signal` id in pniDesign). It does not fail the build.
- If a script fails to compile, the error names the process/task; fix the inline Java. The most likely failure is "Duplicate local variable" — means a script redeclared a process variable; remove the type declaration.
- If the build can't resolve `cz.sykora:sdc-jbpm:1.0.0-SNAPSHOT`, install that artifact into `~/.m2` from its source repo first, then re-run.

- [ ] **Step 3: Commit (GC-AUTOMATION repo)**

The repo is on `master`; create a branch first.
```bash
cd /home/jb/jBPM/GC/GC-AUTOMATION
git checkout -b feature/update-customer-room
git add src/main/resources/com/gc/gc_automation/updateCustomerRoom.bpmn docs/superpowers/
git commit -m "feat: add updateCustomerRoom BPMN process

Re-parents a service's CPE node to a target room (TECHNICAL_AREA node by
Smallworld external id). FIND segment (findService, find cpeCrossId,
findNode/CPE, findRoom) then UPDATE (updateCpeParent via PUT /v1/node with
_meta_parentsImplicitRemove=true). Mirrors logicalDesign structure.

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 4: Commit the runtime-config + PUML changes (separate repos)**

```bash
cd /home/jb/IdeaProjects/GlobalConnect && git checkout -b feature/update-customer-room && \
  git add "gc/gc_docker_compose/files/210_automaton/init/runtime-config(gc-demo).js" && \
  git commit -m "feat(automaton): add cross/findNode, cross/findRoom, cross/updateCpeParent sources + register GC-AUTOMATION.updateCustomerRoom

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
cd /home/jb/IdeaProjects/globalconnect-project-documentation && git checkout -b feature/update-customer-room && \
  git add api_de/sequence_diagrams/7000_0400-update_cust_room_v1.0.puml && \
  git commit -m "docs: align update_cust_room PUML with CROSS API (CPE component, PUT required fields, parent replace, systemId-in-criteria)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 12: Regenerate the SVG companion (manual, Business Central)

**Files:**
- `src/main/resources/com/gc/gc_automation/GC-AUTOMATION.updateCustomerRoom-svg.svg`

- [ ] **Step 1:** Import the kjar / pull the process into jBPM Business Central, open `updateCustomerRoom` in the Process Modeler, and save it. Business Central regenerates the layout and writes the `GC-AUTOMATION.updateCustomerRoom-svg.svg` companion. Commit the BPMN (re-saved) + SVG together, matching the repo's existing "BPMN + matching SVG per commit" convention. This step is manual because a hand-authored SVG would only be a non-representative placeholder.

---

## Self-Review

**Spec coverage:** TMF_STORE+country→smwSystemId (Task 3), findService (Task 4), find cpeCrossId on `componentTypeName=="CPE"` (Task 5), findNode/CPE reading name/status/capacity/nodeTypes (Task 6), findRoom with systemId-in-criteria + uniqueness (Task 7), updateCpeParent PUT with `_meta_parentsImplicitRemove=true` + required fields (Tasks 1 & 8), Upsert/Completed/Failed links (Tasks 5/7/8/9), COMPLETED/FAILED responses (Task 9), runtime-config sources + process registration (Task 1), build (Task 11), SVG (Task 12). All spec sections map to a task.

**Placeholder scan:** the only intentional "placeholder" tokens are `FAIL_THROW_1`/`FAIL_THROW_2` (two fresh UUIDs the executor generates) — flagged explicitly each time. No TBD/TODO logic.

**Type/name consistency:** SdcApiCall output names equal the runtime-config `mapResult` keys (`name`, `statusDiscriminator`, `capacityFull`, `nodeTypeDiscriminators`, `resultList`); variable names and link def-id pairings match across tasks; the JSONPaths (`$.statusRef.id`, `$.nodeTypes[*].id`, `$.items`) match the verified CROSS output DTOs.
