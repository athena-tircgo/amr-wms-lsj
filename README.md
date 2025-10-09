# PTS派車系統 和 WMS倉儲管理系統<br>HTTPS 通訊規格書
## 0. 版本管理


|版本 | 更新| 編制者 |
|:------|:------|:------|
| v1.0.0　新建| 2025-10-07 |Athena |


## 1. 總覽
本規格書定義了派車系統（PTS）與倉儲管理系統(WMS)之間的 HTTP 通訊協定。倉儲管理系統(WMS)作為伺服器端，派車系統（PTS）作為客戶端，透過WebAPI進行通訊，使用HTTP協議傳輸JSON格式資料。

- **通訊協定**：HTTPS
- **傳輸格式**：JSON
- **系統架構**：
  - **WMS Server**：倉儲管理系統伺服器，提供WebAPI服務。
  - **PTS Client**：派車系統端，主動呼叫WMS系統API。


**系統架構圖：**

```mermaid
flowchart TD
    %% 節點設定
    style WMS fill:#f5f5dc,stroke:#333,stroke-width:2px,rx:7,ry:7
    style PTS fill:#7fbf7f,stroke:#333,stroke-width:2px,rx:7,ry:7
    style AMR1 fill:#a3d4f5,stroke:#333,stroke-width:2px,rx:7,ry:7
    style AMR2 fill:#a3d4f5,stroke:#333,stroke-width:2px,rx:7,ry:7
    style AMR3 fill:#a3d4f5,stroke:#333,stroke-width:2px,rx:7,ry:7
    style AMR4 fill:#a3d4f5,stroke:#333,stroke-width:2px,rx:7,ry:7

    %% 節點文字
    WMS["WMS 系統"]
    PTS["PTS 系統"]
    AMR1["AMR_1"]
    AMR2["AMR_2"]
    AMR3["AMR_3"]
    AMR4["AMR_4"]

    %% 連線
    WMS <-->PTS
    PTS -->AMR1
    PTS -->AMR2
    PTS -->AMR3
    PTS -->AMR4
```


## 2. API 定義規格

基礎URL
```
http://[WMS系統IP]:[端口]/api/
```

|項目 | 說明| 類別 | 方法 |
|:------|:------|:------|:-----|
| 1| 取得任務清單 | getTranslationList | GET |
| 2| 回報位置、電量、狀態及異常 | postVehicleStatus | POST |
| 3| 回報派遣任務狀態 |postTranslationState | POST |


### 2.1 取得任務清單

每隔10秒，PTS會主動詢問WMS取得任務清單，若任務清單資訊無異常，將會執行任務。若取得的任務清單解析後有異常，會透過postTranslationState將任務清單的異常資訊回傳給WMS，並且不會執行該項任務。

**API 端點：**
```
getTranslationList.php?STATE=0
```
**請求參數：**
```json
{
  "STATE":"任務狀態",
}
```
- **任務狀態定義：**  
  - STATE=0（尚未執行）
  - STATE=1（執行中）
  - STATE=2（已完成）
  - STATE=3（取消任務）
  - STATE=4（任務清單有異常）
  - STATE=空白（全部）

**回應範例：** 依據getTranslationList所請求的參數回應,如無帶參數(空白),請回覆全部的任務。

無任務回應範例:

```json
{
  "ret": "true",
  "message":"no_task"
}
```
有任務回應範例(若有多個任務，請依序回應):

```json
{
  "ret": "true",
  "data":
    [
        {
            "date":"12:31:05(登記時間)",
            "translation":"1(任務流水號)",
            "Start":"1001(起始點)",
            "Stop1":"1005(停靠點1)",
            "Stop2":"1007（停靠點2)",
            "End":"1001(到達點)",
            "vehicle":"1(指定搬運車編號)",
            "priority":"1(優先順序)"
            "state":"0(任務狀態)",
        },
        {
            "date":"12:31:10",
            "translation":"2",
            "Start":"2001",
            "Stop1":"2005",
            "Stop2":"2006",
            "Stop2":"2009",
            "End":"2001",
            "vehicle":"2",
            "priority":"1"
            "state":"0",
        }
    ]
}
```

- **優先順序定義：**  
  - 緊急：0 (若會優先會插入任務)
  - 普通：1（依照任務的時間順序處理)  

**取消任務：** 只能取消任務狀態**未執行**的任務，**執行中**的任務無法取消。取消任務的方式不需要重新新增任務，直接把State 狀態為0 的任務，改成 State = 3 即可取消任務。

```json
{
  "ret": "true",
  "data":
    [
        {
            "date":"12:31:05",
            "translation":"1",
            "Start":"1001",
            "Stop1":"1005",
            "Stop2":"1007",
            "End":"1001",
            "vehicle":"1",
            "priority":"1"
            "state":"3",
        }
    ]
}

```

**getTranslationList時序圖：**

```mermaid
sequenceDiagram
    participant PTS as PTS (派車系統)
    participant WMS as WMS (倉儲管理系統)
    loop 每 10 問秒一次
        PTS->>WMS: getTranslationList
        alt No task
            WMS-->>PTS: Response no_task
            PTS-->>PTS: Wait 10 seconds for next polling
        else Task available
            WMS-->>PTS: Response Task data (date,translation....)
            PTS->>PTS: Parse task and start executing
        end
    end
```
---

### 2.2 回報位置、電量、狀態及異常

每隔10秒，PTS會回報WMS每一台搬運車的狀態資訊以及是否有異常。

**API 端點：**  
```
postVehicleStatus.php?VEHICLE=1&POSITION=2001&POWER=70&STATUS=2& ERROR=1
```

**請求參數：**
```json
[
  {
    "VEHCILE":"1(搬運車編號)",
    "POSITION":"2001(現在位置)",
    "POWER":"70(電量 1 - 100)",
    "STATUS":"2(搬運車狀態)",
    "ERROR":"1(異常代碼)",
  },
  {
    "VEHCILE":"2",
    "POSITION":"1005",
    "POWER":"95",
    "STATUS":"1",
    "ERROR":"0",
  }
]
```
- **搬運車狀態定義：**
  - STATUS=0（待命中）
  - STATUS=1（工作中）
  - STATUS=2（充電中）
  - STATUS=3（有異常狀況）
  - STATUS=4（無開機或連線異常）

- **異常代碼定義：**  
  - ERROR=0（無異常）
  - ERROR=1（電池電量過低）
  - ERROR=2（圖資須更新）
  - ERROR=3（有障礙物）

**回應範例：**
```json
{
  "ret": "true",
  "message": "完成登錄作業", 
}
```
**postVehicleStatus時序圖：**

```mermaid
sequenceDiagram
    participant PTS
    participant WMS

    loop 每10秒回報一次
        PTS->>WMS: postVehicleStatus (AMR 狀態)
        WMS-->>PTS: Response 完成登錄作業
    end
```

---

### 2.3 回報派遣任務狀態

每隔10秒，PTS會回報WMS執行中和已完成的任務清單，若有收到的派遣任務清單格式有異常無法處理，也會透過此方式回報讓WMS掌握。

**API 端點：**
```
postTranslationState.php?VEHCILE=1&TRANSLATION=2&STATE=2&ERROR=0
```

**請求參數：**

```json
[
  {
      "VEHCILE": "1(搬運車編號)",
      "TRANSLATION": "2(任務流水號)",
      "STATE": "2(任務狀態)",
      "ERROR":"0(無異常)"
  },
  {
      "VEHCILE": "2",
      "TRANSLATION": "1",
      "STATE": "4",
      "ERROR":"2(站點不存在)"
  }
]

```
- **任務狀態定義：**  
  - STATE=1（執行中)
  - STATE=2（已完成）
  - STATE=4（任務清單有異常）

- **任務異常定義：**
  - ERROR=0（任務資訊無異常)
  - ERROR=1（站點重複)
  - ERROR=2（不存在的站點）
  - ERROR=3（指定的車號不存在）
  - ERROR=4（任務流水號重複）
  - ERROR=5（指定的車號未開機）


**回應範例：**
```json
{
  "ret": "true",
  "message": "完成登錄作業", 
}
```

**postVehicleStatus時序圖：**

```mermaid
sequenceDiagram
    participant PTS
    participant WMS

    loop 每10秒回報一次
        PTS->>WMS: postTranslationState (派遣任務狀態)
        WMS-->>PTS: Response 完成登錄作業
    end
```

## 3. 操作情境

### 3.1 情境1
WMS 要下任務前須確認每台AMR 電源是否開啟，並確認PTS系統已經啟用

```mermaid
sequenceDiagram
    participant PTS
    participant WMS

    loop 每10秒回報一次
        PTS->>WMS:postVehicleStatus (AMR_1 開啟電源, Status=0 待命中)
        WMS-->>PTS: Response 完成登錄作業
    end

    loop 每10秒回報一次
        PTS->>WMS:postVehicleStatus (AMR_1 開啟電源, Status=0 待命中 )
        WMS-->>PTS: Response 完成登錄作業
        PTS->>WMS:postVehicleStatus (AMR_2 開啟電源, Status=0 待命中)
        WMS-->>PTS: Response 完成登錄作業
    end
```

### 3.1 情境2






## 4. JSON 傳輸格式說明

JSON (JavaScript Object Notation) 是一種輕量級的資料交換格式，常用於伺服器與客戶端之間的資料傳輸。
其格式以純文字構成，易於人類閱讀與撰寫，也方便機器解析與生成。

**一、基本結構**

JSON 的資料由兩種基本結構組成：

**1.物件（Object)** : 使用 { } 括號包起來的鍵值對集合。

例如：

```json
{
  "VEHCILE": "2",
  "TRANSLATION": "1",
  "STATE": "1"
}
```

**2.陣列（Array）** : 使用 [ ] 括號包起來的有序資料集合。

例如：

```json
[
  {
      "VEHCILE": "1",
      "TRANSLATION": "2",
      "STATE": "2)"
  },
  {
      "VEHCILE": "2",
      "TRANSLATION": "1",
      "STATE": "1"
  }
]

```


**二、資料型態**

JSON 支援以下基本型態：

| 型態名稱 | 範例 | 說明 |
|-----------|--------|------|
| 字串 (String) | `"VEHCILE"` | 以雙引號包圍的文字 |
| 整數 (Number) | `100` | 可為整數或小數 |
| 布林 (Boolean) | `true / false` | 表示邏輯值 |
| 陣列 (Array) | `[1, 2, 3]` | 有序的值集合 |
| 物件 (Object) | `{"TRANSLATION":2, "STATE":2}` | 鍵值對集合 |
| Null | `null` | 表示空值或未知資料 |


**三、語法規則**

1.鍵名（Key）必須使用雙引號 "key"。

2.值（Value）可為上述任一型態。

3.鍵值對以冒號 : 分隔。

4.各鍵值對以逗號 , 分隔。

5.最外層可以是物件 {} 或陣列 []。



**四、傳輸特性**

- 編碼格式：UTF-8

- MIME Type：application/json

- 傳輸方向：AMR ↔ WMS

- 優點：結構清晰、解析效率高、跨平台相容性佳




## 5. HTTPS 傳輸規範說明

**一、傳輸協定**

系統採用 HTTPS（Hypertext Transfer Protocol Secure） 作為通訊協定，確保資料在 AMR 與 WMS 之間傳遞時具備加密與完整性保護。

| 項目 | 說明 |
|:-----------|:--------|
| 通訊協定	| HTTPS（基於 HTTP over TLS）| 
| 埠號（Port）	| 預設為 443| 
| 加密層	| TLS 1.2 或以上版本| 
| 資料格式	| JSON| 
| 傳輸方向	| 雙向（AMR ↔ WMS）| 

**二、安全機制**

1.TLS 加密傳輸
  - 所有通訊內容經 TLS 加密，防止攔截與竄改。
  - 不允許使用明碼 HTTP 傳輸。

2.伺服器憑證驗證
  - AMR 在連線時須驗證 WMS 伺服器的 SSL 憑證是否有效（由受信任 CA 簽發）。
  - 憑證若過期、無效或域名不符，應拒絕連線。

3.身份驗證（Authentication）
  - 若系統需求，HTTP Header 中可加入 API Token 或 Bearer Token 作為身分驗證機制。

4.資料完整性（Integrity）
  - 所有請求及回應應透過 HTTPS 保證資料未被竄改。


**三、HTTP通用狀態碼**

| 狀態碼 | 說明 | 處理方式 |
|:-----------|:--------|:--------|
| 200	| 成功	| 無需重試| 
| 400	| 請求格式錯誤	| 檢查 JSON 結構| 
| 401	| 驗證失敗	| 檢查 Token 是否有效| 
| 404	| 資源不存在	| 檢查 API 路徑| 
| 500	| 伺服器內部錯誤	| 等待後重試| 
