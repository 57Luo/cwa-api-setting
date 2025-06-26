## 透過中央氣象局 API 獲取天氣資料 for Home Assistan

要在 Home Assistant 中透過 **中央氣象署開放資料平臺** 的 REST API 取得即時氣象觀測資料，可以參考以下步驟與範例：

### 1. 前往開放資料平臺

* 平臺連結： [中央氣象署開放資料平臺 API 文件](https://opendata.cwa.gov.tw/dist/opendata-swagger.html#/)
* 選擇資料集：`O-A0001-001`（自動測站氣象資料）
* 點擊欲查詢的 API，藉由參數 `StationName` 篩選指定測站。

### 2. 在 `configuration.yaml` 中加入 REST Sensor

```yaml
sensor:
  - platform: rest
    name: "即時天氣資料"
    resource: |
      https://opendata.cwa.gov.tw/api/v1/rest/datastore/O-A0001-001
      ?Authorization= your cwa_api_key 
      &limit=100
      &StationName=%E7%99%BD%E6%B2%B3
    method: GET
    scan_interval: 1800  # 每 30 分鐘更新一次
    verify_ssl: true     # 建議開啟 SSL 驗證
    json_attributes_path: "$.records.Station[0]"
    json_attributes:
      - WeatherElement
      - ObsTime
    value_template: "{{ value_json.records.Station[0].StationName }}"
```

> **備註：也可以使用 *`params`* 分離參數的寫法**
>
> ```yaml
> sensor:
>   - platform: rest
>     name: "即時天氣資料"
>     resource: "https://opendata.cwa.gov.tw/api/v1/rest/datastore/O-A0001-001"
>     params:
>       Authorization: !secret cwa_api_key
>       limit: 100
>       StationName: "白河"
>     method: GET
>     scan_interval: 1800  # 每 30 分鐘更新一次
>     verify_ssl: true     # 建議開啟 SSL 驗證
>     json_attributes_path: "$.records.Station[0]"
>     json_attributes:
>       - WeatherElement
>       - ObsTime
>     value_template: "{{ value_json.records.Station[0].StationName }}"
> ```

---

### 3. 欄位說明

| 欄位                     | 說明                                                                                                                                          |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `platform: rest`       | 指定為 RESTful API 感測器。                                                                                                                        |
| `name`                 | 感測器在 Home Assistant 中的顯示名稱，如 `sensor.即時天氣資料`。                                                                                             |
| `resource`             | API 請求 URL，包含以下參數：- `Authorization`：您的 API Key（建議以 `{{ cwa_api_key }}` 從 `secrets.yaml` 引用）- `limit`：最大回傳筆數- `StationName`：欲擷取的測站名稱（URL 編碼） |
| `method: GET`          | HTTP 請求方法，通常為 `GET`。                                                                                                                        |
| `scan_interval: 1800`  | 資料更新頻率（秒），此範例為 1800 秒（30 分鐘）。                                                                                                               |
| `verify_ssl`           | 是否驗證 SSL 憑證。建議設為 `true`，確保連線安全。                                                                                                             |
| `json_attributes_path` | JSONPath 路徑，指定要抽取的屬性節點，此處取陣列中第一筆測站資料。                                                                                                       |
| `json_attributes`      | 要抽出的屬性名稱清單，如 `WeatherElement`（氣象要素）與 `ObsTime`（觀測時間）。                                                                                       |
| `value_template`       | 定義感測器主要狀態的 Jinja 模板，此例顯示測站名稱。                                                                                                               |

### 4. 進階建議

* **Secrets 管理**：在 `secrets.yaml` 中定義 `cwa_api_key: YOUR_API_KEY`，避免直接將金鑰寫入 `configuration.yaml`。
* **多測站串接**：如要同時觀測多個測站，可複製多個 sensor 區塊，僅修改 `name` 與 `StationName` 參數。
* **自訂屬性** (Template Sensor):若只需要特定氣象要素（例如：溫度、濕度），可以透過 **template sensor** 進一步萃取。
    1. #### **建立 `template` 資料夾**  
       >##### 在 Home Assistant Config 目錄下，新增 `template/` 資料夾<br>並在 `configuration.yaml` 中包含它：
       ```yaml
       template: !include_dir_merge_named template
       ```
    2. #### 新增 sensor 定義
          >##### 在 `template` 資料夾下新增 `sensor.yaml（檔名可自訂）`<br>並加入以下內容：  
          ```yaml
            sensor:
              - name: "相對濕度"
                unique_id: humi
                unit_of_measurement: "%"
                state_class: measurement
                state: >
                  {% set w = state_attr('sensor.bai_he_ji_shi_tian_qi_zi_liao', 'WeatherElement') %}
                  {% set h = w.RelativeHumidity if w and 'RelativeHumidity' in w else None %}
                  {{ h | float(0) if h is not none else unavailable }}
          ```
> *以上設定完成後，重新啟動 Home Assistant，即可在`開發工具 > 狀態`裡看到「即時天氣資料」感測器及其屬性。*