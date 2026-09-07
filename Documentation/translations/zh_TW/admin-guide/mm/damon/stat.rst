.. SPDX-License-Identifier: GPL-2.0
.. include:: ../../../disclaimer-zh_TW.rst

:Original: Documentation/admin-guide/mm/damon/stat.rst

:翻譯:

 Doehyun Baek <doehyunbaek@gmail.com>

====================
資料存取監測結果統計
====================

資料存取監測結果統計（DAMON_STAT）是用於簡易存取模式監測的靜態核心模組。
它使用 DAMON 監測系統全部實體記憶體的存取情形，並提供簡化的統計資料，
包括閒置時間的百分位數和估計的記憶體頻寬。

.. _damon_stat_monitoring_accuracy_overhead_zh_TW:

監測準確度與額外負擔
====================

DAMON_STAT 使用監測間隔的
:ref:`自動調校 <damon_design_monitoring_intervals_autotuning_zh_TW>` 機制，
以提高準確度並降低額外負擔。它會自動調校間隔，目標是在每個快照中擷取
4 % 的可觀測存取事件，同時將取樣間隔限制在 5 毫秒到 10 秒之間。
在少數正式運作的伺服器系統上，測試結果顯示它僅使用單一 CPU 時間的
0.x %，便能擷取品質合理的存取模式。調校後的間隔可透過
``aggr_interval_us`` :ref:`參數 <damon_stat_aggr_interval_us_zh_TW>` 取得。

介面：模組參數
==============

使用此功能前，請先確認系統執行的核心在建置時啟用了
``CONFIG_DAMON_STAT=y``。將 ``CONFIG_DAMON_STAT_ENABLED_DEFAULT`` 設為
true，即可在建置時指定預設啟用此功能。

DAMON_STAT 提供模組參數，讓系統管理員在開機時或執行期間啟用或停用模組，
以及讀取監測結果。以下各節說明這些參數。

enabled
-------

啟用或停用 DAMON_STAT。

將此參數設為 ``Y`` 可啟用 DAMON_STAT，設為 ``N`` 則停用。
預設值由建置組態選項 ``CONFIG_DAMON_STAT_ENABLED_DEFAULT`` 決定。

請注意，此模組（damon_stat）無法與其他基於 DAMON 的專用模組同時執行。
詳情請參閱 :ref:`DAMON 設計文件的專用模組互斥性
<damon_design_special_purpose_modules_exclusivity_zh_TW>`。

.. _damon_stat_aggr_interval_us_zh_TW:

aggr_interval_us
----------------

自動調校後的彙整間隔，單位為微秒。

使用者可讀取 DAMON_STAT 所使用的 DAMON 實例的彙整間隔。
此間隔會 :ref:`自動調校 <damon_stat_monitoring_accuracy_overhead_zh_TW>`，
因此其值會動態變化。

estimated_memory_bandwidth
--------------------------

系統記憶體頻寬使用量的估計值，單位為位元組/秒。

DAMON_STAT 讀取目前 DAMON 結果快照中觀測到的存取事件，將其轉換為以
位元組/秒表示的記憶體頻寬使用量估計值，再透過此唯讀參數提供給使用者。
由於 DAMON 採用取樣方式，這只是存取強度的估計，而非精確的記憶體頻寬。

memory_idle_ms_percentiles
--------------------------

系統記憶體中各位元組的閒置時間百分位數，單位為毫秒。

DAMON_STAT 根據目前的 DAMON 結果快照，計算記憶體中每個位元組截至目前
未被存取的時間（閒置時間）。若區域的存取頻率（``nr_accesses``）大於零，
則以目前存取頻率持續的時間乘以 ``-1``，作為該區域各位元組的閒置時間。
若區域的存取頻率為零，則以零存取頻率持續的時間（``age``）作為閒置時間。
DAMON_STAT 透過此唯讀參數提供這些閒置時間的百分位數。讀取此參數會傳回
101 個以逗號分隔、單位為毫秒的閒置時間值，依序表示第 0、1、2、3、……、
99 和 100 百分位的閒置時間。

kdamond_pid
-----------

DAMON 執行緒的 PID。

若 DAMON_STAT 已啟用，此參數會顯示工作執行緒的 PID；否則為 -1。
