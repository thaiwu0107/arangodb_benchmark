- [ArangoDB 集群測試報告](#arangodb-%E9%9B%86%E7%BE%A4%E6%B8%AC%E8%A9%A6%E5%A0%B1%E5%91%8A)
  - [Introdunction](#introdunction)
  - [目的](#%E7%9B%AE%E7%9A%84)
  - [測試項目](#%E6%B8%AC%E8%A9%A6%E9%A0%85%E7%9B%AE)
  - [未測試的項目](#%E6%9C%AA%E6%B8%AC%E8%A9%A6%E7%9A%84%E9%A0%85%E7%9B%AE)
  - [測試環境](#%E6%B8%AC%E8%A9%A6%E7%92%B0%E5%A2%83)
  - [已知問題](#%E5%B7%B2%E7%9F%A5%E5%95%8F%E9%A1%8C)
  - [結論](#%E7%B5%90%E8%AB%96)
  - [附錄](#%E9%99%84%E9%8C%84)

# ArangoDB 集群測試報告
## Introdunction
本測試文件提供驗證 ArangoDB Cluster 效能及穩定度相關資料。

## 目的
- 測試方法及結果須符合系統規格最低需求
    - 五萬玩家同時上線。
    - 執行 Slot Game (每 2 秒產生一筆 Bet records，遊戲平台承載量為 25,000 Bet records/sec)。
    - 每個玩家平均每個回合執行遊戲 10 分鐘，產生 300 筆 Bet records。
    - 必須最少可以儲存 7 天或更多的 Bet records (七天共產生 150 億筆 Bet records)。
- 評估出一組符合最低系統效能及穩定度需求的 ArangoDB cluster 參數及硬體規格。

## 測試項目
- ArangoDB option "WaitForSync"
  - Set true
  - Set false
- 不同的寫入方式, 影響的持續寫入的效能
  - AQL
  - import
  - save
- Storage
  - SSD
  - HDD
- ArangoDB database engin
  - RockDB
  - Mmfile
- Bet record diff data structure (不存每筆詳細 Bet record 只存 Round record 以及每筆 Bet record 差異) 的影響
  - Full bet record
  - Diff bet record
- ArangoDB unload collection 對寫入速度及系統記憶體消耗的影響
  - Set collection unload
  - Don't set collection unload
- Database split bet records to different collection
  - Bet records are stored in different collection
  - Bet records are stored in one collection

## 未測試的項目
- 長期穩定度。
- 持續寫入當下, 刪除 collection 的影響。

## 測試環境
- 硬件
  - CPU：Intel（R）Core（TM）i7-8700
  - 內存：16 GB
  - SSD：230 GB
- 軟件
  - ArangoDB 3.4 release
  - 3 台 Linux Mint 服務器
  - 28 個 Node.js instance 對 ArangoDB 寫入資料

## 已知問題
- ArangoDB 的 Cluster 在每台機器上消耗硬碟空間量不同。
- 持續寫入 database 速度不穩定，最低寫入速率為 1/3。

## 結論
- Best configuration
  - SSD storage
  - RockDB database engin
  - split collection
  - Set Arango database option wait-for-sync = true
  - unload collection
  - Linux File Max: 1611339

- 最佳兩個結果比較(都是可以使用的方式)
    - diff data structure: 每一千筆diff data structure當作一個document
    - 沒有diff data structure: 每一筆diff data structure當作一個document

    | 差異                    | 平均寫入時間(sec) | 平均每一筆大小 | 總共寫入(筆) | SSD空間消耗大小 |
    | ----------------------- | ----------------- | -------------- | ------------ | --------------- |
    | diff data structure     | 6.5 萬/sec        | 1.54 bytes     | 522,755,000  | 6G              |
    | 沒有diff data structure | 9.2 萬/sec        | 6.6 bytes      | 669,400,000  | 33G             |

## 附錄
```
第一次完整測試
```
- 測試配置
  - WaitForSync: False
  - Linux File Max: 預設值
  - Journal size：256
  - 硬碟配置：HDD
  - Engine：Mmfile
  - insert方式：arangojs import()

- 測試數據
  - 寫入模式：每次1筆 * 每5毫秒20次
  - 持續時間：35分10秒
  - 總寫入筆數：從0到870萬筆, 總寫入870萬筆
  - 平均每秒寫入4千筆

- 測試失敗原因
  - 達到Linux限制(Linux File Max 590432)

```
第二次完整測試
```
- 測試配置
  - WaitForSync: False
  - Linux File Max: 1611339
  - Journal size：256
  - 硬碟配置：HDD
  - Engine：Mmfile
  - insert方式：arangojs import()

- 測試數據
  - 寫入模式：每次10筆 * 每5毫秒10次, 總共每秒2萬筆
  - 持續時間：29分55秒
  - 總寫入筆數：從870萬到4100萬筆, 總寫入3230萬筆
  - 平均每秒1.8萬筆寫入

- 測試失敗原因
   - 筆數超過4000萬以後，寫入速度變慢，以相同速度持續送資料已經達到HDD的IO上限
   - 13分10秒後恢復正常，測試每秒1筆可正常寫入
   - 測試每秒1千筆，資料立即堵塞無法寫入，10分50秒後恢復正常

```
第三次完整測試
```
- 測試配置
  - WaitForSync: False
  - Linux File Max: 1611339
  - Journal size：256
  - 硬碟配置：HDD
  - Engine：Mmfile
  - insert方式：arangojs import()

- 測試數據
  - 寫入模式：每次1筆 * 每秒3百次
  - 持續時間：2天13小時43分50秒
  - 總寫入筆數：從0到6095萬筆, 總寫入6095萬筆
  - 平均每秒3百筆

- 測試失敗原因
    - 無, 寫入太慢, 改測試別種方式

```
第四次完整測試
```
- 測試配置
  - WaitForSync: False
  - Linux File Max: 1611339
  - Journal size：256
  - 硬碟配置：HDD
  - Engine：Mmfile
  - insert方式：arangojs import()
  - 資料寫入5分鐘停10分鐘

- 測試數據
  - 寫入模式：每次2千筆 * 每秒1次
  - 持續時間：17小時35分30秒
  - 總寫入筆數：從0到4274萬筆, 總寫入4274萬筆
  - 平均每秒674筆

- 測試失敗原因
  - 筆數超過4200萬以後，寫入5分鐘停10分鐘，已經達到HDD的IO上限

```
第五次完整測試
```
- 測試配置
  - WaitForSync: False
  - Linux File Max: 1611339
  - Journal size：256
  - 硬碟配置：HDD
  - Engine：Mmfile
  - insert方式：arangojs import()
  - 資料寫入5分鐘停10分鐘

- 測試數據
  - 寫入模式：每次1萬筆 * 每秒1次
  - 持續時間：16小時19分30秒
  - 總寫入筆數：從0到5817萬筆, 總寫入5817萬筆
  - 平均每秒987筆

- 測試失敗原因
  - 筆數超過4千萬以後，寫入5分鐘停10分鐘，達到HDD的IO上限
  - 預計寫入1.95億筆，實際上因IO阻塞，只寫入5817萬筆

```
第六次完整測試
```
- 測試配置
  - WaitForSync: False
  - Linux File Max: 1611339
  - Journal size：256
  - 硬碟配置：HDD
  - Engine：Mmfile
  - insert方式：arangojs import()
  - 資料寫入5分鐘停10分鐘

- 測試數據
  - 寫入模式：每次10萬筆 * 每秒1次
  - 持續時間：06分40秒
  - 總寫入筆數：0到4千萬筆, 總寫入4千萬筆
  - 平均每秒十萬筆

- 測試失敗原因
  - 筆數超過4千萬以後，寫入速度變慢，以相同速度持續送資料會導致達到HDD的IO上限
  - 恢復正常後，繼續以每秒10萬筆測試，寫入20萬筆後失敗

```
第七次完整測試
```
- 測試配置
  - WaitForSync: True
  - Linux File Max: 1611339
  - Journal size：256
  - 硬碟配置：HDD
  - Engine：Mmfile
  - insert方式：arangojs import()
  - 寫入模式：每次寫入完成後才, 進行下一次寫入

- 測試數據
  - 持續時間：2天6小時52分10秒
  - 總寫入筆數：8007萬筆
  - 抽樣時間：04小時53分00秒，當前筆數，5494萬筆，web開啟順暢，平均每秒寫入約3125筆
  - 抽樣時間：23小時03分30秒，當前筆數，6978萬筆，web開啟順暢，平均每秒寫入約840筆
  - 抽樣時間：1天03小時29分30秒，當前筆數，7012萬筆，web開啟稍微卡頓，平均每秒寫入約708筆，中途有暫停，但不影響測試數據
  - 抽樣時間：1天22小時36分46秒，當前筆數，7766萬筆，web開啟稍微卡頓，平均每秒寫入約462筆
  - 抽樣時間：2天03小時10分30秒，當前筆數，7898萬筆，web開啟卡頓，平均每秒寫入約428筆
  - 抽樣時間：2天06小時52分10秒，當前筆數，8007萬筆，web開啟卡頓，平均每秒寫入約405筆，最後一筆，之後清空DB重啟

- 註記
  - 超過4000萬筆速度變慢但不明顯，5000～6000萬之間變化更加明顯，超過7000萬後約每秒寫入100筆以下。
  - web開啟速度在6000萬筆以前看不出來，7000萬之後才有明顯等待
  - 除了速度變慢之外，寫入皆正常無error

```
第八次完整測試
```
- 測試配置
  - WaitForSync: True
  - Linux File Max: 1611339
  - Journal size：256
  - 硬碟配置：HDD
  - Engine：Mmfile
  - insert方式：arangojs import()
  - 測試方式：使用三台電腦只有針對塞入的筆數不同測試
  - 寫入模式：每次寫入完成後才, 進行下一次寫入

- 測試數據
  - 寫入速度：
  - 166機器：每次1百筆
  - 210機器：每次5百筆
  - 211機器：每次1千筆
  - 持續時間：
  - 166機器：5天17小時24分00秒
  - 210機器：5天17小時24分00秒
  - 211機器：5天17小時24分00秒
  - 總寫入筆數：
  - 166機器：1.07億筆
  - 210機器：1.15億筆
  - 211機器：1.17億筆
  - 第一次抽樣時間：
  - 166機器：15小時13分00秒，當前筆數，5587萬筆，web開啟順暢，平均每秒寫入約1019筆
  - 210機器：15小時14分00秒，當前筆數，6337萬筆，web開啟順暢，平均每秒寫入約1155筆
  - 211機器：15小時15分00秒，當前筆數，6483萬筆，web開啟順暢，平均每秒寫入約1180筆
  - 第二次抽樣時間：
  - 166機器：19小時01分00秒，當前筆數，5915萬筆，web開啟順暢，平均每秒寫入約864筆
  - 210機器：19小時04分00秒，當前筆數，6573萬筆，web開啟順暢，平均每秒寫入約957筆
  - 211機器：19小時03分00秒，當前筆數，6736萬筆，web開啟順暢，平均每秒寫入約982筆
  - 第三次抽樣時間：
  - 166機器：4天17小時40分01秒，當前筆數，9598萬筆，web開啟順暢，平均每秒寫入約234筆
  - 210機器：4天17小時40分01秒，當前筆數，1.05億筆，web開啟須1秒，平均每秒寫入約256筆
  - 211機器：4天17小時40分01秒，當前筆數，1.06億筆，web開啟須7～8秒，平均每秒寫入約259筆
  - 第四次抽樣時間：
  - 166機器：4天22小時46分00秒，當前筆數，9961萬筆，web開啟順暢，平均每秒寫入約233筆
  - 210機器：4天22小時46分00秒，當前筆數，1.08億筆，web開啟須2～3秒，平均每秒寫入約252筆
  - 211機器：4天22小時46分00秒，當前筆數，1.10億筆，web開啟須9～10秒，平均每秒寫入約257筆
  - 第五次抽樣時間：
  - 166機器：5天17小時24分00秒，當前筆數，1.07億筆，web開啟順暢，平均每秒寫入約216筆
  - 210機器：5天17小時24分00秒，當前筆數，1.15億筆，web開啟須2～3秒，平均每秒寫入約232筆
  - 211機器：5天17小時24分00秒，當前筆數，1.17億筆，web開啟須5~7秒，平均每秒寫入約237筆

```
第九次完整測試
```
- 測試配置
  - WaitForSync: True
  - Linux File Max: 1611339
  - Journal size：256
  - insert方式：arangojs import()
  - 硬碟配置：SSD
  - 資料格式：split collection and unload collection
  - 寫入模式：每次寫入完成後才, 進行下一次寫入

- 測試數據
  - 持續測試時間：30分鐘

  | 差異   | 總資料筆數 | SSD空間 | 每秒寫入速度 |
  | ------ | ---------- | ------- | ------------ |
  | RockDB | 2.2億      | 13G     | 12萬/sec     |
  | Mmfile | 2.3億      | 52G     | 12萬/sec     |


```
第十次完整測試
```
- 測試配置
  - WaitForSync: True
  - Linux File Max: 1611339
  - Journal size：256
  - insert方式：arangojs import()
  - 硬碟配置：SSD
  - Engine：RockDB
  - 資料格式：split collection and unload collection
  - 寫入模式：每次寫入完成後才, 進行下一次寫入

- 測試數據
  - 持續測試時間：16個小時

  | 使用的DB | 總資料筆數 | SSD空間 | 每秒寫入速度 | 平均每筆資料大小 |
  | -------- | ---------- | ------- | ------------ | ---------------- |
  | RockDB   | 44.17億    | 195G    | 7.6萬/sec    | 5.9byte          |

```
第十一次完整測試
```
- 測試配置
  - WaitForSync: True
  - Linux File Max: 1611339
  - Journal size：256
  - insert方式：arangojs save()
  - 硬碟配置：SSD
  - Engine：RockDB
  - 資料格式：split collection and unload collection
  - 寫入模式：每次寫入完成後才, 進行下一次寫入

- 測試數據
  - 持續測試時間：2小時14分

  | 差異                   | 總資料筆數 | SSD空間 | 每秒寫入速度 | 平均每筆資料大小 |
  | ---------------------- | ---------- | ------- | ------------ | ---------------- |
  | diff data structure    | 5.22755億  | 6G      | 6.5萬/sec    | 1.54byte         |
  | no diff data structure | 6.694億    | 33G     | 9.2萬/sec    | 6.6byte          |

```
第十二次完整測試
```
- 測試配置
  - WaitForSync: True
  - Linux File Max: 1611339
  - Journal size：256
  - insert方式：arangojs save()
  - 硬碟配置：SSD
  - Engine：RockDB
  - 資料格式：split collection and unload collection
  - diff data structure
  - 測試的資料裡面的值都是亂數產生
  - 寫入模式：每次寫入完成後才, 進行下一次寫入

- 測試數據

  | 差異                     | 總資料筆數 | SSD空間 | 每秒寫入速度 | 平均每筆資料大小 | 測試時間   |
  | ------------------------ | ---------- | ------- | ------------ | ---------------- | ---------- |
  | 每200筆當作一筆document  | 40.8億     | 322G    | 6.96萬/sec   | 10.5byte         | 16小時17分 |
  | 每1000筆當作一筆document | 6.2394億   | 53G     | 8.67萬/sec   | 11.4byte         | 2小時      |
