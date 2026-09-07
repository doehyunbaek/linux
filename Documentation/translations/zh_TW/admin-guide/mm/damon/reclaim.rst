.. SPDX-License-Identifier: GPL-2.0
.. include:: ../../../disclaimer-zh_TW.rst

:Original: Documentation/admin-guide/mm/damon/reclaim.rst

:翻譯:

 司延騰 Yanteng Si <siyanteng@loongson.cn>
 Doehyun Baek <doehyunbaek@gmail.com>

:校譯:

=================
基於 DAMON 的回收
=================

基於 DAMON 的回收（DAMON_RECLAIM）是靜態核心模組，用於在輕度記憶體壓力
下進行主動、輕量的回收。它並非要取代基於 LRU 串列、以頁面為粒度的回收
機制，而是依不同的記憶體壓力與需求，選擇性地使用。

何時需要主動回收？
==================

在一般記憶體超額配置的系統上，主動回收冷頁可節省記憶體，並減少行程直接
回收或 kswapd 消耗 CPU 所造成的延遲突增，同時僅造成輕微的效能下降
[1]_ [2]_。

以空閒頁回報 [3]_ 為基礎、採用記憶體超額配置的虛擬化系統就是一個例子。
在這類系統中，客體虛擬機器向主機回報空閒記憶體，主機再將其重新分配給
其他客體，使系統的記憶體得到充分利用。然而，客體不一定會節省記憶體，
主要是因為部分核心子系統和使用者空間應用程式會盡可能使用可用的記憶體。
因此，客體可能只向主機回報少量空閒記憶體，造成系統記憶體利用率下降。
在客體中執行主動回收可緩解此問題。

運作方式
========

DAMON_RECLAIM 找出一段指定時間內未被存取的記憶體區域，並將其換出。
為避免換出操作消耗過多 CPU，可設定速率限制。在此限制下，優先換出較長
時間未被存取的區域。系統管理員也可設定三個記憶體壓力水位，決定此方案
應在何種情況下自動啟動或停止作用。

介面：模組參數
==============

使用此功能前，請先確認系統執行的核心在建置時啟用了
``CONFIG_DAMON_RECLAIM=y``。

DAMON_RECLAIM 提供模組參數，讓系統管理員啟用或停用模組，並針對系統調校。
可在核心開機命令列加入 ``damon_reclaim.<parameter>=<value>``，或將適當的值
寫入 ``/sys/module/damon_reclaim/parameters/<parameter>`` 檔案。

以下說明各個參數。

enabled
-------

啟用或停用 DAMON_RECLAIM。

將此參數設為 ``Y`` 可啟用 DAMON_RECLAIM，設為 ``N`` 則停用。
請注意，即使已啟用，DAMON_RECLAIM 仍可能因為未滿足水位啟動條件而不進行
實際的監測與回收。詳情請參閱下方的水位參數說明。

commit_inputs
-------------

讓 DAMON_RECLAIM 重新讀取 ``enabled`` 以外的輸入參數。

DAMON_RECLAIM 執行期間更新的輸入參數預設不會套用。將此參數設為 ``Y`` 後，
DAMON_RECLAIM 會重新讀取 ``enabled`` 以外的參數值，完成後將此參數設為
``N``。若重新讀取時發現無效參數，DAMON_RECLAIM 會被停用。

寫入 ``Y`` 後，使用者不得再寫入任何參數，直到再次讀取 ``commit_inputs``
傳回 ``N`` 為止。違反此規則可能導致核心出現未定義的行為。

min_age
-------

用於識別冷記憶體區域的時間門檻，單位為微秒。

若記憶體區域在此時間或更長時間內未被存取，DAMON_RECLAIM 會將其視為冷區域
並回收。

預設為 120 秒。

autotune_monitoring_intervals
-----------------------------

將此參數設為 ``Y`` 時，DAMON_RECLAIM 會自動調校 DAMON 的取樣與彙整間隔。
目標是在每個 DAMON 快照中擷取有意義數量的存取事件，同時將取樣間隔限制
在 5 毫秒到 10 秒之間。設為 ``N`` 則停用自動調校。

預設停用。

quota_ms
--------

嘗試回收所能使用的時間上限，單位為毫秒。

DAMON_RECLAIM 在每個時間區間（``quota_reset_interval_ms``）內，最多使用
此時間嘗試回收冷頁，以限制 CPU 使用量。設為零可停用此限制。

預設為 10 毫秒。

quota_sz
--------

嘗試回收的記憶體大小上限，單位為位元組。

DAMON_RECLAIM 會將每個時間區間（``quota_reset_interval_ms``）內嘗試回收的
記憶體量計入配額，確保不超過此上限，以限制 CPU 與 I/O 使用量。
設為零可停用此限制。

預設為 128 MiB。

quota_reset_interval_ms
-----------------------

時間與大小配額的用量重設間隔，單位為毫秒。

此參數指定時間配額（``quota_ms``）與大小配額（``quota_sz``）的用量重設
間隔。也就是說，在每個 ``quota_reset_interval_ms`` 毫秒內，DAMON_RECLAIM
嘗試回收所用的時間不超過 ``quota_ms`` 毫秒，嘗試回收的記憶體量不超過
``quota_sz`` 位元組。

預設為 1 秒。

quota_mem_pressure_us
---------------------

期望的記憶體壓力停滯時間，單位為微秒。

DAMON_RECLAIM 在遵守其他配額上限的同時，會自動增減有效配額，目標是讓
記憶體壓力停滯時間達到此值。它會在每個配額重設間隔
（``quota_reset_interval_ms``）收集全系統的 ``some`` 記憶體壓力停滯資訊
（PSI），以微秒為單位，並與此值比較，判斷是否達成目標。
設為零可停用此自動調校功能。

預設停用。

quota_autotune_feedback
-----------------------

由使用者提供、用於有效配額自動調校的回饋值。

DAMON_RECLAIM 在遵守其他配額上限的同時，會自動增減有效配額，目標是收到
使用者提供的 ``10,000`` 回饋值。它假設回饋值與配額成正比。
設為零可停用此自動調校功能。

預設停用。

wmarks_interval
---------------

DAMON_RECLAIM 已啟用但因水位規則而未作用時，檢查水位前的最短等待時間。

wmarks_high
-----------

高水位的空閒記憶體比率，單位為千分之一。

若系統空閒記憶體的千分比高於此值，DAMON_RECLAIM 會停止作用，僅定期檢查
水位。

wmarks_mid
----------

中水位的空閒記憶體比率，單位為千分之一。

若系統空閒記憶體的千分比介於此值與低水位之間，DAMON_RECLAIM 會開始作用，
進行監測與回收。

wmarks_low
----------

低水位的空閒記憶體比率，單位為千分之一。

若系統空閒記憶體的千分比低於此值，DAMON_RECLAIM 會停止作用，僅定期檢查
水位。此時，系統會退回基於 LRU 串列、以頁面為粒度的回收機制。

sample_interval
---------------

監測的取樣間隔，單位為微秒。

DAMON 監測冷記憶體時使用的取樣間隔。詳情請參閱 DAMON 文件（:doc:`usage`）。

aggr_interval
-------------

監測的彙整間隔，單位為微秒。

DAMON 監測冷記憶體時使用的彙整間隔。詳情請參閱 DAMON 文件（:doc:`usage`）。

min_nr_regions
--------------

監測區域數量的下限。

DAMON 監測冷記憶體時的最少區域數，可用來設定監測品質的下限。
但設得太高會增加監測的額外負擔。詳情請參閱 DAMON 文件（:doc:`usage`）。

此值必須至少為 3，原因請參閱設計文件的
:ref:`監測 <damon_design_monitoring_zh_TW>` 章節。

max_nr_regions
--------------

監測區域數量的上限。

DAMON 監測冷記憶體時的最多區域數，可用來限制監測的額外負擔。
但設得太低會降低監測品質。詳情請參閱 DAMON 文件（:doc:`usage`）。

monitor_region_start
--------------------

目標記憶體區域的起始實體位址。

DAMON_RECLAIM 處理的記憶體區域的起始實體位址。它會在此區域內找出冷記憶體
並回收。預設使用系統全部的實體記憶體。

monitor_region_end
------------------

目標記憶體區域的結束實體位址。

DAMON_RECLAIM 處理的記憶體區域的結束實體位址。它會在此區域內找出冷記憶體
並回收。預設使用系統全部的實體記憶體。

addr_unit
---------

記憶體位址與位元組數的縮放因子。

用於設定及取得 DAMON_RECLAIM 所使用的 DAMON 實例的
:ref:`位址單位 <damon_design_addr_unit_zh_TW>` 參數。

``monitor_region_start`` 和 ``monitor_region_end`` 應以此單位表示。
例如，若 ``addr_unit``、``monitor_region_start`` 和 ``monitor_region_end``
分別設為 ``1024``、``0`` 和 ``10``，DAMON_RECLAIM 就會處理從位址零開始、
長度為 10 KiB 的實體位址範圍（以位元組表示為 ``[0 * 1024, 10 * 1024)``）。

``bytes_reclaim_tried_regions`` 和 ``bytes_reclaimed_regions`` 也使用此單位。
例如，若 ``addr_unit``、``bytes_reclaim_tried_regions`` 和
``bytes_reclaimed_regions`` 分別為 ``1024``、``42`` 和 ``32``，表示
DAMON_RECLAIM 共嘗試回收 42 KiB 記憶體，並成功回收其中的 32 KiB。

若不確定，使用預設值（``1``）即可，不必另行調整此參數。

skip_anon
---------

略過匿名頁的回收。

將此參數設為 ``Y`` 時，DAMON_RECLAIM 不會回收匿名頁。預設為 ``N``。

kdamond_pid
-----------

DAMON 執行緒的 PID。

若 DAMON_RECLAIM 已啟用，此參數會顯示工作執行緒的 PID；否則為 -1。

nr_reclaim_tried_regions
------------------------

DAMON_RECLAIM 嘗試回收的記憶體區域總數。

bytes_reclaim_tried_regions
---------------------------

DAMON_RECLAIM 嘗試回收的記憶體區域總位元組數。

nr_reclaimed_regions
--------------------

DAMON_RECLAIM 成功回收的記憶體區域總數。

bytes_reclaimed_regions
-----------------------

DAMON_RECLAIM 成功回收的記憶體區域總位元組數。

nr_quota_exceeds
----------------

超過時間或大小配額上限的次數。

範例
====

下列執行期間的命令範例讓 DAMON_RECLAIM 找出至少 30 秒未被存取的記憶體
區域並將其換出。為避免換出操作消耗過多 CPU 時間，回收量限制為每秒最多
1 GiB。系統空閒記憶體比率高於 50% 時，DAMON_RECLAIM 不進行實際工作；
低於 40% 時開始工作。若回收沒有進展，空閒記憶體比率因而低於 20%，則再次
停止工作，退回基於 LRU 串列、以頁面為粒度的回收機制。::

    # cd /sys/module/damon_reclaim/parameters
    # echo 30000000 > min_age
    # echo $((1 * 1024 * 1024 * 1024)) > quota_sz
    # echo 1000 > quota_reset_interval_ms
    # echo 500 > wmarks_high
    # echo 400 > wmarks_mid
    # echo 200 > wmarks_low
    # echo Y > enabled

請注意，此模組（damon_reclaim）無法與其他基於 DAMON 的專用模組同時執行。
詳情請參閱 :ref:`DAMON 設計文件的專用模組互斥性
<damon_design_special_purpose_modules_exclusivity_zh_TW>`。

.. [1] https://research.google/pubs/pub48551/
.. [2] https://lwn.net/Articles/787611/
.. [3] https://www.kernel.org/doc/html/latest/mm/free_page_reporting.html
