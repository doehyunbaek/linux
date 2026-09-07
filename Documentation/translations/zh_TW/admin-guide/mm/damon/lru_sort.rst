.. SPDX-License-Identifier: GPL-2.0
.. include:: ../../../disclaimer-zh_TW.rst

:Original: Documentation/admin-guide/mm/damon/lru_sort.rst

:翻譯:

 臧雷剛 Leigang Zang <zangleigang@hisilicon.com>
 Doehyun Baek <doehyunbaek@gmail.com>

:校譯:

==========================
基於 DAMON 的 LRU 串列排序
==========================

基於 DAMON 的 LRU 串列排序（DAMON_LRU_SORT）是靜態核心模組，以主動、輕量
的方式，根據資料存取模式提高或降低頁面在 LRU 串列上的優先順序，讓 LRU
串列成為更可靠的存取模式資訊來源。

何時需要主動排序 LRU 串列？
===========================

在大型系統中，以頁面為粒度檢查存取情形可能帶來顯著的額外負擔。因此，
LRU 串列通常不會主動排序，而是回應特定使用者請求、系統呼叫、記憶體壓力
等事件，進行局部排序。這使得 LRU 串列在某些情況下無法充分反映存取模式，
例如在突然出現記憶體壓力時，選擇要回收的頁面。

DAMON 能在使用者指定的額外負擔範圍內，盡可能準確地識別存取模式。
因此，主動執行 DAMON_LRU_SORT，可在負擔低且可控的前提下，讓 LRU 串列
成為更可靠的存取模式資訊來源。

運作方式
========

DAMON_LRU_SORT 使用 DAMON 找出熱頁（所在區域的存取頻率高於使用者指定
門檻的頁面）與冷頁（所在區域未被存取的時間超過使用者指定門檻的頁面），
提高熱頁並降低冷頁在 LRU 串列上的優先順序。為避免排序消耗過多 CPU，
可設定 CPU 使用時間上限。在此限制下，優先提高較熱頁面的優先順序，
並降低較冷頁面的優先順序。系統管理員也可設定三個記憶體壓力水位，決定
此方案應在何種情況下自動啟動或停止作用。

冷熱門檻與 CPU 配額的預設值較為保守。採用預設參數的模組可廣泛用於一般
情境而不致造成傷害，僅使用少量、有限的 CPU 時間，就能讓記憶體壓力下
具有明確冷熱存取模式的系統受益。

介面：模組參數
==============

使用此功能前，請先確認系統執行的核心在建置時啟用了
``CONFIG_DAMON_LRU_SORT=y``。

DAMON_LRU_SORT 提供模組參數，讓系統管理員啟用或停用模組，並針對系統調校。
可在核心開機命令列加入 ``damon_lru_sort.<parameter>=<value>``，或將適當的值
寫入 ``/sys/module/damon_lru_sort/parameters/<parameter>`` 檔案。

以下說明各個參數。

enabled
-------

啟用或停用 DAMON_LRU_SORT。

將此參數設為 ``Y`` 可啟用 DAMON_LRU_SORT，設為 ``N`` 則停用。
請注意，即使已啟用，DAMON_LRU_SORT 仍可能因為未滿足水位啟動條件而不進行
實際的監測與 LRU 串列排序。詳情請參閱下方的水位參數說明。

commit_inputs
-------------

讓 DAMON_LRU_SORT 重新讀取 ``enabled`` 以外的輸入參數。

DAMON_LRU_SORT 執行期間更新的輸入參數預設不會套用。將此參數設為 ``Y`` 後，
DAMON_LRU_SORT 會重新讀取 ``enabled`` 以外的參數值，完成後將此參數設為
``N``。若重新讀取時發現無效參數，DAMON_LRU_SORT 會被停用。

寫入 ``Y`` 後，使用者不得再寫入任何參數，直到再次讀取 ``commit_inputs``
傳回 ``N`` 為止。違反此規則可能導致核心出現未定義的行為。

active_mem_bp
-------------

期望的活躍記憶體占活躍與非活躍 LRU 記憶體總量的比率，單位為 bp
（萬分之一）。

DAMON_LRU_SORT 在遵守其他配額上限的同時，會自動增減有效配額，透過提高
熱頁與降低冷頁的 LRU 優先順序，使活躍記憶體比率達到此目標。
設為零可停用此自動調校功能。

預設停用。

autotune_monitoring_intervals
-----------------------------

將此參數設為 ``Y`` 時，DAMON_LRU_SORT 會自動調校 DAMON 的取樣與彙整間隔。
目標是在每個 DAMON 快照中擷取有意義數量的存取事件，同時將取樣間隔限制
在 5 毫秒到 10 秒之間。設為 ``N`` 則停用自動調校。

預設停用。

filter_young_pages
------------------

依頁面是否最近被存取，篩選適合提高或降低 LRU 優先順序的頁面。

啟用此參數後，每次調整 LRU 優先順序前，都會再次以頁面為粒度檢查存取
情形（young 狀態）。若頁面自上次檢查後未被存取（非 young），則略過
提高優先順序的操作；若頁面曾被存取（young），則略過降低優先順序的操作。
設為 ``Y`` 可啟用此功能，設為 ``N`` 則停用。

預設停用。

hot_thres_access_freq
---------------------

用於識別熱記憶體區域的存取頻率門檻，單位為千分之一。

若記憶體區域的存取頻率大於或等於此值，DAMON_LRU_SORT 會將其視為熱區域，
並在 LRU 串列上標記為已存取，避免在記憶體壓力下被回收。預設為 50%。

cold_min_age
------------

用於識別冷記憶體區域的時間門檻，單位為微秒。

若記憶體區域在此時間或更長時間內未被存取，DAMON_LRU_SORT 會將其視為
冷區域，並在 LRU 串列上標記為未存取，使其在記憶體壓力下優先被回收。
預設為 120 秒。

quota_ms
--------

嘗試排序 LRU 串列所能使用的時間上限，單位為毫秒。

DAMON_LRU_SORT 在每個時間區間（``quota_reset_interval_ms``）內，最多使用
此時間嘗試排序 LRU 串列，以限制 CPU 使用量。設為零可停用此限制。

預設為 10 毫秒。

quota_reset_interval_ms
-----------------------

時間配額的用量重設間隔，單位為毫秒。

此參數指定時間配額（``quota_ms``）的用量重設間隔。也就是說，在每個
``quota_reset_interval_ms`` 毫秒內，DAMON_LRU_SORT 嘗試排序 LRU 串列所用
的時間不超過 ``quota_ms`` 毫秒。

預設為 1 秒。

wmarks_interval
---------------

水位檢查間隔，單位為微秒。

DAMON_LRU_SORT 已啟用但因水位規則而未作用時，檢查水位前的最短等待時間。
預設為 5 秒。

wmarks_high
-----------

高水位的空閒記憶體比率，單位為千分之一。

若系統空閒記憶體的千分比高於此值，DAMON_LRU_SORT 會停止作用，僅定期檢查
水位。預設為 200（20%）。

wmarks_mid
----------

中水位的空閒記憶體比率，單位為千分之一。

若系統空閒記憶體的千分比介於此值與低水位之間，DAMON_LRU_SORT 會開始作用，
進行監測與 LRU 串列排序。預設為 150（15%）。

wmarks_low
----------

低水位的空閒記憶體比率，單位為千分之一。

若系統空閒記憶體的千分比低於此值，DAMON_LRU_SORT 會停止作用，僅定期檢查
水位。預設為 50（5%）。

sample_interval
---------------

監測的取樣間隔，單位為微秒。

DAMON 監測冷記憶體時使用的取樣間隔。詳情請參閱 DAMON 文件（:doc:`usage`）。
預設為 5 毫秒。

aggr_interval
-------------

監測的彙整間隔，單位為微秒。

DAMON 監測冷記憶體時使用的彙整間隔。詳情請參閱 DAMON 文件（:doc:`usage`）。
預設為 100 毫秒。

min_nr_regions
--------------

監測區域數量的下限。

DAMON 監測冷記憶體時的最少區域數，可用來設定監測品質的下限。
但設得太高會增加監測的額外負擔。詳情請參閱 DAMON 文件（:doc:`usage`）。
預設為 10。

此值必須至少為 3，原因請參閱設計文件的
:ref:`監測 <damon_design_monitoring_zh_TW>` 章節。

max_nr_regions
--------------

監測區域數量的上限。

DAMON 監測冷記憶體時的最多區域數，可用來限制監測的額外負擔。
但設得太低會降低監測品質。詳情請參閱 DAMON 文件（:doc:`usage`）。
預設為 1000。

monitor_region_start
--------------------

目標記憶體區域的起始實體位址。

DAMON_LRU_SORT 處理的記憶體區域的起始實體位址。
預設使用系統全部的實體記憶體。

monitor_region_end
------------------

目標記憶體區域的結束實體位址。

DAMON_LRU_SORT 處理的記憶體區域的結束實體位址。
預設使用系統全部的實體記憶體。

addr_unit
---------

記憶體位址與位元組數的縮放因子。

用於設定及取得 DAMON_LRU_SORT 所使用的 DAMON 實例的
:ref:`位址單位 <damon_design_addr_unit_zh_TW>` 參數。

``monitor_region_start`` 和 ``monitor_region_end`` 應以此單位表示。
例如，若 ``addr_unit``、``monitor_region_start`` 和 ``monitor_region_end``
分別設為 ``1024``、``0`` 和 ``10``，DAMON_LRU_SORT 就會處理從位址零開始、
長度為 10 KiB 的實體位址範圍（以位元組表示為 ``[0 * 1024, 10 * 1024)``）。

以 ``bytes_`` 為前綴的統計參數也使用此單位。
例如，若 ``addr_unit``、``bytes_lru_sort_tried_hot_regions`` 和
``bytes_lru_sorted_hot_regions`` 分別為 ``1024``、``42`` 和 ``32``，表示
DAMON_LRU_SORT 共嘗試對 42 KiB 熱記憶體進行 LRU 排序，並成功排序其中的
32 KiB。

若不確定，使用預設值（``1``）即可，不必另行調整此參數。

kdamond_pid
-----------

DAMON 執行緒的 PID。

若 DAMON_LRU_SORT 已啟用，此參數會顯示工作執行緒的 PID；否則為 -1。

nr_lru_sort_tried_hot_regions
-----------------------------

嘗試進行 LRU 排序的熱記憶體區域總數。

bytes_lru_sort_tried_hot_regions
--------------------------------

嘗試進行 LRU 排序的熱記憶體區域總位元組數。

nr_lru_sorted_hot_regions
-------------------------

成功進行 LRU 排序的熱記憶體區域總數。

bytes_lru_sorted_hot_regions
----------------------------

成功進行 LRU 排序的熱記憶體區域總位元組數。

nr_hot_quota_exceeds
--------------------

超過熱區域時間配額上限的次數。

nr_lru_sort_tried_cold_regions
------------------------------

嘗試進行 LRU 排序的冷記憶體區域總數。

bytes_lru_sort_tried_cold_regions
---------------------------------

嘗試進行 LRU 排序的冷記憶體區域總位元組數。

nr_lru_sorted_cold_regions
--------------------------

成功進行 LRU 排序的冷記憶體區域總數。

bytes_lru_sorted_cold_regions
-----------------------------

成功進行 LRU 排序的冷記憶體區域總位元組數。

nr_cold_quota_exceeds
---------------------

超過冷區域時間配額上限的次數。

範例
====

下列執行期間的命令範例讓 DAMON_LRU_SORT 找出存取頻率至少為 50% 的區域，
提高其 LRU 優先順序，並降低至少 120 秒未被存取的區域的優先順序。
為避免排序消耗過多 CPU 時間，優先順序調整限制為最多使用 1% 的 CPU 時間。
系統空閒記憶體比率高於 50% 時，DAMON_LRU_SORT 不進行實際工作；低於 40%
時開始工作。若排序沒有進展，空閒記憶體比率因而低於 20%，則再次停止工作，
退回基於 LRU 串列、以頁面為粒度的回收機制。::

    # cd /sys/module/damon_lru_sort/parameters
    # echo 500 > hot_thres_access_freq
    # echo 120000000 > cold_min_age
    # echo 10 > quota_ms
    # echo 1000 > quota_reset_interval_ms
    # echo 500 > wmarks_high
    # echo 400 > wmarks_mid
    # echo 200 > wmarks_low
    # echo Y > enabled

請注意，此模組（damon_lru_sort）無法與其他基於 DAMON 的專用模組同時執行。
詳情請參閱 :ref:`DAMON 設計文件的專用模組互斥性
<damon_design_special_purpose_modules_exclusivity_zh_TW>`。
