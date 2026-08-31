# node-dashboard-url delta — netapp-ontap-cluster-dashboard-param

## MODIFIED Requirements

### Requirement: Dashboard 請求參數組裝

`/dashboard` 查詢的 query 參數 MUST 由被開啟節點的 `data` 屬性(及 dashboard `timeRange`)以純函式組裝(可單測),參數值型別為 `string | string[]`(`string[]` 即重複參數,如 `ipaddress`),規則如下:

- **排除集合**:`labels` 與所有 panel 內部 rendering-only / 結構欄位 MUST NOT 送出——accent 顏色(`clusterColor` / `namespaceColor`)、`parent`、`worstStatus`、`is*` compound 旗標(`isCluster` / `isController` / `isNamespace`;`isStorageClass` 已隨 D6 storageclass leaf 化而從旗標模型移除,不再列於排除集合)、`storageclass` leaf 的**結構欄位** `provisioner` 與 `parameters`(供右上角釘選 tooltip 呈現之用,屬節點資訊而非 query 參數),以及結構性的 `id`(後端 D6 起 controller 的 `id` 為後端 path 值如 `<c>/namespace/<ns>/application/<app>/controller/<Kind>/<name>`,屬結構識別而非可查詢屬性;節點身分以 kind + name 表示)。
- **僅送 scalar(`ipaddress` 例外)**:非 scalar 值(陣列 / 物件,如 `alerts` / `containers` / `owner` / `parameters`)MUST NOT 作為 query 參數送出。**例外**:`ipAddress`(`string[]`,pod 節點上)SHALL 以重複 `ipaddress=` 參數送出(陣列原樣交予 `getBackendSrv().get` 的 `params`,序列化為重複 query 參數);陣列為空時 MUST 省略。
- **欄名對應**:節點顯示名存於 `data.label`(normalize 自上游 `name` 對應而來、未保留 `name`),組裝時 MUST 以 `name` 為參數名送出該值;`kind` 原樣送出。
- **Leaf 節點**:送出其(經上述排除後的)scalar 屬性。
- **Compound 節點(僅 k8s-node / controller)**:送出該容器**自身**的 scalar 屬性,**外加**在其**所有直接子節點**(`data.parent === 容器 id`)上**值皆相同**的屬性;值在子節點間**相異**的屬性 MUST **略過**;與自身屬性**衝突時自身值優先**(own-wins);子節點屬性同樣套用上述排除集合與 scalar-only 規則;容器無直接子節點時僅送自身屬性。
- **`cluster` 參數**:雖然 `cluster` 對 eligible 節點**非** first-class data 欄(`labels` 又在排除集合中),Panel SHALL 仍解析並送出 `cluster`,且**同一個參數名 `cluster` 同時承載 K8s cluster 與 ONTAP cluster**——backend 由節點的 `kind` 分辨兩者,Panel MUST NOT 另立 `ontap_cluster` 參數。**權威來源**為該節點**最近的 cluster 容器祖先**(沿 `data.parent` 上溯,穿過 namespace box、`netapp-node` 等中間 compound):該祖先為 `isCluster` 時取其 `data.cluster`,為 `isStorageCluster`(ONTAP cluster 容器)時取其 `data.storageCluster`;上溯途中遇到的**第一個** cluster 容器即為答案,MUST NOT 越過它續尋另一種。祖先解析是唯一能涵蓋 **controller** 的來源(controller 既無 `data.cluster` 亦無 `labels`)。找不到任一種 cluster 容器祖先時 MUST **退回**該節點自身的 `labels.cluster`,再退回 `labels.ontap_cluster`(NetApp 節點不屬於任何 K8s cluster,故永不帶 `labels.cluster`;其 ONTAP cluster 名僅存於 `labels.ontap_cluster`);兩種 label 皆無時 MUST **省略** `cluster`(如無所屬 cluster 的頂層 external 節點)。祖先解析 MUST **優先於** labels 退回(祖先為權威),labels 退回中 `labels.cluster` MUST **優先於** `labels.ontap_cluster`,且 MUST **不覆寫**節點自身已帶的 `cluster`(own-wins)。
- **`controller` 參數**(與 `cluster` 對稱):Panel SHALL 解析並送出 `controller`:**權威來源**為該節點**最近的 `isController` 祖先**(沿 `data.parent` 上溯)之名稱(`data.label`)——controller mode 下一個 pod 的直接 parent 即其 controller compound。找不到 `isController` 祖先時(如 k8s-node mode,pod 巢狀於 node compound 下、無 controller 節點)MUST **退回**該 pod 自身的 `data.owner.name`(`useNodeDetailUrls` 解析 pod controller 的同一來源);兩者皆無時 MUST **省略** `controller`(controller compound 自身無 parent controller;裸 service / pvc / storageclass / external 無 owner)。祖先解析 MUST **優先於** owner 退回,且 MUST **不覆寫**節點自身已帶的 `controller`(own-wins)。`controller` 與既有的 `application`(ArgoCD application name)正交,兩者可並存。
- **`from_time` / `to_time` 參數**:Panel SHALL 由 dashboard 當前 `timeRange`(panel `PanelProps` 的 `timeRange`)送出 `from_time` = `timeRange.from` 之 **Unix 秒**、`to_time` = `timeRange.to` 之 **Unix 秒**(與 backend graph query 的 `start`/`end` 同採 Unix 秒;backend 亦接受 RFC 3339,此處採秒)。時間界由組裝純函式自 `timeRange` 參數注入,僅於 eligible(非 `undefined`)分支加入,與其餘參數共用同一 param map / 請求 key。`timeRange` 缺漏時 MUST 省略 `from_time` / `to_time`。

#### Scenario: leaf 參數排除 labels 與 rendering 欄、label 以 name 送出

- **WHEN** 對一個帶 `kind` / `label` / `namespace` / `labels` / `parent` 的 pod leaf 組裝參數
- **THEN** 送出 `kind` 與 `name`(值為 `data.label`)、以及 `namespace`;MUST NOT 送出 `labels`、`parent`、或任何 `is*` / `*Color` / `worstStatus` / `id` 欄

#### Scenario: storageclass leaf 結構欄位 provisioner / parameters 不送出

- **WHEN** 對一個帶 `kind: 'storageclass'` / `label` / `provisioner` / `parameters` 的 storageclass leaf 組裝參數
- **THEN** 送出 `kind` 與 `name`(值為 `data.label`);MUST NOT 送出 `provisioner`、`parameters`(結構欄位),亦 MUST NOT 送出任何 `is*` / `*Color` / `parent` / `id` 欄

#### Scenario: compound 合併子節點一致屬性、略過相異屬性

- **WHEN** 對一個 controller compound 組裝參數,其所有子 pod 的某屬性(如 `namespace`)值皆相同、另一屬性(如 `name`)值各異
- **THEN** 該一致屬性併入參數(若自身未帶該欄),相異屬性略過;自身已帶的欄以自身值為準(own-wins)

#### Scenario: 非 scalar 與合成 id 不送出(`ipAddress` 除外)

- **WHEN** 被組裝的節點帶 `alerts` / `containers` / `owner` / `parameters` 等非 scalar 欄,且(若為 controller)帶合成 `id`
- **THEN** 這些欄 MUST NOT 出現在 `/dashboard` 的 query 參數中(`ipAddress` 為例外,見下「ipaddress」scenario)

#### Scenario: ipAddress 以重複 ipaddress 參數送出

- **WHEN** 對一個帶 `ipAddress: ['10.0.0.1', '10.0.0.2']` 的 pod leaf 組裝參數
- **THEN** `/dashboard` 帶重複 `ipaddress=10.0.0.1&ipaddress=10.0.0.2`(陣列原樣送出)
- **WHEN** pod 的 `ipAddress` 缺漏或為空陣列
- **THEN** `ipaddress` 參數 MUST 省略

#### Scenario: controller 自最近 isController 祖先解析

- **WHEN** 某 pod leaf 於 controller mode 巢狀於某 `isController` compound 之下
- **THEN** `/dashboard` 參數含 `controller`,值為該最近 `isController` 祖先的名稱(`data.label`)

#### Scenario: controller 退回 owner.name、皆無則省略

- **WHEN** 某 pod 無任何 `isController` 祖先(如 k8s-node mode)但自身帶 `data.owner.name`
- **THEN** `controller` 取自 `data.owner.name`
- **WHEN** 節點既無 `isController` 祖先亦無 `data.owner`(如 controller compound 自身、或裸 service / pvc / storageclass / external)
- **THEN** `controller` 參數 MUST 省略

#### Scenario: from_time / to_time 帶 timeRange 之 Unix 秒

- **WHEN** 對一個 eligible 節點組裝參數,且 dashboard `timeRange` 之 `from` / `to` 對應 Unix 秒 `1700000000` / `1700003600`
- **THEN** `/dashboard` 參數含 `from_time=1700000000` 與 `to_time=1700003600`
- **WHEN** `timeRange` 缺漏
- **THEN** `from_time` / `to_time` 參數 MUST 省略

#### Scenario: cluster 自最近 isCluster 祖先解析(含 controller 經 namespace box 上溯)

- **WHEN** 某 eligible 節點(leaf / controller / k8s-node)巢狀於某 `isCluster` compound 之下(可能經 namespace box 等中間 compound)
- **THEN** `/dashboard` 參數含 `cluster`,值為該最近 `isCluster` 祖先的 `data.cluster`

#### Scenario: NetApp 節點的 cluster 自最近 isStorageCluster 祖先解析

- **WHEN** 某 `netapp-node` compound(其 parent 為某 `isStorageCluster` 容器)或某 `netapp-aggr` leaf(經其 `netapp-node` parent 上溯至該容器)被開啟
- **THEN** `/dashboard` 參數含 `cluster`,值為該最近 `isStorageCluster` 祖先的 `data.storageCluster`(即該 ONTAP cluster 名),與 K8s 節點使用同一個參數名

#### Scenario: NetApp 節點無 storage-cluster 祖先時退回 labels.ontap_cluster

- **WHEN** 某 `netapp-node` / `netapp-aggr` 節點無任何 cluster 容器祖先(上游未送出 `storage-cluster` 群組),但自身帶 `labels.ontap_cluster`
- **THEN** `cluster` 取自 `labels.ontap_cluster`

#### Scenario: cluster 退回 labels.cluster、皆無則省略

- **WHEN** 節點無任何 cluster 容器祖先但自身帶 `labels.cluster`
- **THEN** `cluster` 取自 `labels.cluster`(即使同時帶 `labels.ontap_cluster`,`labels.cluster` 仍優先)
- **WHEN** 節點既無 cluster 容器祖先,亦無 `labels.cluster` 與 `labels.ontap_cluster`(如頂層 external)
- **THEN** `cluster` 參數 MUST 省略
