.. SPDX-License-Identifier: GPL-2.0
.. include:: ../../disclaimer-zh_TW.rst

:Original: Documentation/mm/damon/design.rst

:翻譯:

 司延騰 Yanteng Si <siyanteng@loongson.cn>
 Doehyun Baek <doehyunbaek@gmail.com>

:校譯:

====
設計
====

.. _damon_design_execution_model_and_data_structures_zh_TW:

執行模型與資料結構
==================

與監測有關的資訊，包括監測請求規格與基於 DAMON 的操作方案，儲存在稱為
DAMON 監測情境（``context``）的資料結構中。DAMON 使用名為 ``kdamond``
的核心執行緒執行各個情境。多個 kdamond 可平行執行，進行不同類型的監測。

使用者空間如何設定並啟動或停止 DAMON，請參閱
:ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 文件。

使用者也可請求暫停或繼續執行個別情境。暫停時，kdamond 除了套用執行期間
的參數更新，不會進行其他工作。

使用者空間如何暫停或繼續執行情境，請參閱
:ref:`DAMON sysfs 情境 <sysfs_context_zh_TW>` 使用文件。

整體架構
========

DAMON 子系統由三個層次組成：

- :ref:`操作集 <damon_operations_set_zh_TW>`：實作 DAMON 的基本操作，
  這些操作依賴指定的監測目標位址空間，以及可用的軟硬體基礎操作。
- :ref:`核心邏輯 <damon_core_logic_zh_TW>`：在操作集層之上，實作監測的
  額外負擔與準確度控制，以及存取感知系統操作等核心邏輯。
- :ref:`模組 <damon_modules_zh_TW>`：在核心邏輯層之上，實作各種用途的
  核心模組，並提供使用者空間介面。

.. _damon_operations_set_zh_TW:

操作集層
========

.. _damon_design_configurable_operations_set_zh_TW:

為了監測資料存取及執行其他低階工作，DAMON 需要一組特定操作的實作，
這些實作依賴指定的目標位址空間，並針對該空間最佳化。例如，下列兩種
存取監測操作都與位址空間有關：

1. 識別該位址空間中要監測的目標位址範圍。
2. 檢查目標空間中特定位址範圍的存取情形。

DAMON 將這些實作整合至稱為 DAMON 操作集的層次，並定義它與上層之間的介面。
上層專責 DAMON 的核心邏輯，包括控制監測準確度與額外負擔的機制。

因此，只要設定核心邏輯使用適當的操作集，就能將 DAMON 擴充至任意位址空間
或可用的硬體功能。若沒有適合特定用途的操作集，也可依照層間介面實作新的
操作集。

例如，實體記憶體、虛擬記憶體、交換空間，以及特定行程、NUMA 節點、檔案
和後端記憶體裝置的位址空間，都可能獲得支援。若某些架構或裝置提供特殊的
最佳化存取檢查功能，也可輕易加以設定。

DAMON 目前提供下列三種操作集。以下小節說明其運作方式。

- ``vaddr``：監測特定行程的虛擬位址空間。
- ``fvaddr``：監測固定的虛擬位址範圍。
- ``paddr``：監測系統的實體位址空間。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定，
請參閱 :ref:`operations <sysfs_context_zh_TW>` 檔案的說明。

.. _damon_design_vaddr_target_regions_construction_zh_TW:

基於 VMA 的目標位址範圍建構
---------------------------

``vaddr`` 操作集會自動初始化並更新監測目標區域，以涵蓋目標行程的全部
記憶體映射。

此機制僅適用於 ``vaddr`` 操作集。使用 ``fvaddr`` 或 ``paddr`` 操作集時，
使用者必須手動設定監測目標位址範圍。

行程極大的虛擬位址空間中，只有一小部分會映射至實體記憶體並被存取，
因此追蹤未映射區域只是浪費資源。不過，DAMON 的適應性區域調整機制可容忍
一定程度的雜訊，所以不必逐一追蹤每個映射；逐一追蹤有時反而會造成很高的
額外負擔。但監測目標中過大的未映射區域仍應排除，以免適應性機制在這些
區域上耗費時間。

因此，此實作將複雜的映射轉換為三個相異區域，涵蓋位址空間中的所有映射。
三個區域之間的兩個空隙，是該位址空間中最大的兩個未映射區域。通常分別是
堆積（heap）與最上方 mmap() 映射區域之間，以及最下方 mmap() 映射區域與
堆疊（stack）之間的空隙。由於這些空隙在一般位址空間中非常大，只要排除
它們，便能取得合理的取捨。如下所示：::

    <heap>
    <BIG UNMAPPED REGION 1>
    <uppermost mmap()-ed region>
    (small mmap()-ed regions and munmap()-ed regions)
    <lowermost mmap()-ed region>
    <BIG UNMAPPED REGION 2>
    <stack>

基於 PTE 存取位元的存取檢查
---------------------------

實體與虛擬位址空間的實作都使用 PTE 的 Accessed 位元進行基本存取檢查，
差異僅在於如何從位址找到相關位元。虛擬位址的實作走訪該位址所屬目標工作
的頁表；實體位址的實作則走訪所有映射至該位址的頁表。實作會找到並清除
下一個取樣目標位址的存取位元，再於一個取樣週期後檢查位元是否重新設為一。

這可能干擾其他使用存取位元的核心子系統，也就是閒置頁追蹤與回收邏輯。
DAMON 不會主動避免干擾閒置頁追蹤，因此系統管理員必須自行處理此干擾。
但它會像閒置頁追蹤一樣，利用 ``PG_idle`` 與 ``PG_young`` 頁面旗標，
解決與回收邏輯的衝突。

.. _damon_design_addr_unit_zh_TW:

位址單位
--------

DAMON 核心邏輯層使用 ``unsigned long`` 型別表示監測目標位址範圍。
有時操作集的位址空間太大，無法以此型別表示，例如具有大型實體位址擴充
功能的 32 位元 ARM。為此，DAMON 提供各操作集專屬的 ``address unit``
（位址單位）參數。它是縮放因子，將核心邏輯層的位址乘以此值，便能得到
該位址空間中的實際位址。是否支援此參數取決於各操作集的實作，目前僅
``paddr`` 支援。

若此值小於 ``PAGE_SIZE``，則必須使用 2 的冪次。

.. _damon_core_logic_zh_TW:

核心邏輯
========

.. _damon_design_monitoring_zh_TW:

監測
----

以下小節說明 DAMON 的核心監測機制，以及五個監測屬性：``取樣間隔``、
``彙整間隔``、``更新間隔``、``最少區域數`` 和 ``最多區域數``。

``最少區域數`` 必須至少為 3。這是因為虛擬位址空間監測的設計至少需要
三個區域，以容納一般虛擬位址空間中常見的兩個大型未映射區域。對 ``paddr``
等其他操作集而言，這不一定是必要限制，但目前為了一致性，所有 DAMON
操作集都必須遵守此限制。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
這些屬性，請參閱 :ref:`monitoring_attrs <sysfs_monitoring_attrs_zh_TW>`。

存取頻率監測
~~~~~~~~~~~~

DAMON 的輸出指出各頁面在指定時間內的存取頻率。頻率的解析度由 ``取樣間隔``
與 ``彙整間隔`` 控制。DAMON 每隔一個取樣間隔檢查各頁面的存取情形，並彙整
結果，也就是計算各頁面的存取次數。每當一個彙整間隔結束，DAMON 便呼叫
使用者先前註冊的回呼函式，讓使用者讀取彙整結果，然後清除結果。
可用下列虛擬碼表示：::

    while monitoring_on:
        for page in monitoring_target:
            if accessed(page):
                nr_accesses[page] += 1
        if time() % aggregation_interval == 0:
            for callback in user_registered_callbacks:
                callback(monitoring_target, nr_accesses)
            for page in monitoring_target:
                nr_accesses[page] = 0
        sleep(sampling interval)

此機制的監測負擔會隨著目標工作負載的大小增加，而無限制地成長。

.. _damon_design_region_based_sampling_zh_TW:

基於區域的取樣
~~~~~~~~~~~~~~

為避免額外負擔無限制地成長，DAMON 將假定具有相同存取頻率的相鄰頁面歸為
一個區域。只要此假設成立，每個區域就只須檢查一個頁面。因此，DAMON 在
每個 ``取樣間隔`` 隨機選取各區域的一個頁面，等待一個取樣間隔後，檢查
該頁面是否在這段期間被存取。若是，便遞增該區域的存取頻率計數器，稱為
``nr_accesses``。因此可透過設定區域數量控制監測負擔。DAMON 允許使用者
設定最少與最多區域數，以取得適當取捨。

但若此假設不成立，這種方法便無法維持輸出品質。

.. _damon_design_adaptive_regions_adjustment_zh_TW:

適應性區域調整
~~~~~~~~~~~~~~

即使初始監測區域符合假設（同區域的頁面具有相近存取頻率），存取模式仍
可能動態改變，導致監測品質下降。為盡可能維持此假設，DAMON 根據各區域
的存取頻率，適應性地合併或分割區域。

每個 ``彙整間隔``，DAMON 都會比較相鄰區域的存取頻率（``nr_accesses``）。
若差異較小，且兩區域大小的總和小於全部區域的總大小除以 ``最少區域數``，
就合併這兩個區域。若合併後的總區域數仍高於 ``最多區域數``，便提高存取
頻率差異門檻並重複合併，直到滿足區域數上限，或門檻高於可能的最大值
（``彙整間隔 / 取樣間隔``）。回報並清除各區域彙整後的存取頻率後，若
分割後的總區域數不超過使用者指定上限，就將各區域分成兩個或三個區域。

如此，DAMON 能在遵守使用者設定界限的同時，盡可能提高品質並降低負擔。

.. _damon_design_age_tracking_zh_TW:

存取模式持續時間追蹤
~~~~~~~~~~~~~~~~~~~~

分析監測結果也能得知區域目前的存取模式已維持多久，有助於理解存取模式。
例如，可據此實作同時考量存取頻率與最近存取情形的頁面配置演算法。
為簡化此分析，DAMON 在每個區域維護另一個稱為 ``age`` 的計數器。
每個 ``彙整間隔``，DAMON 都會檢查區域大小與存取頻率（``nr_accesses``）
是否明顯改變。若有改變，就將計數器重設為零；否則遞增計數器。

.. _damon_design_data_attrs_monitoring_zh_TW:

資料屬性監測
~~~~~~~~~~~~

資料存取模式只是資料屬性的一種。有些使用情境需要更多屬性資訊，例如
某個熱或冷記憶體區域中，有多少由匿名頁構成，或屬於特定 cgroup。
資料屬性監測功能就是為此而設。

使用者可將感興趣的資料屬性註冊至 DAMON
:ref:`情境 <damon_design_execution_model_and_data_structures_zh_TW>`，
為每個屬性指定一個探針。各探針以規則判斷記憶體區域是否具有相關屬性。
規則由多個篩選器組成，除了支援的類型不同，其運作方式與
:ref:`DAMOS 篩選器 <damon_design_damos_filters_zh_TW>` 相同。
目前資料屬性監測僅支援 ``anon`` 與 ``memcg`` 篩選器。

註冊探針後，DAMON 在進行存取
:ref:`取樣 <damon_design_region_based_sampling_zh_TW>` 時，會對各區域
取樣到的記憶體執行探針。每個
:ref:`彙整間隔 <damon_design_monitoring_zh_TW>` 內，判定具有該屬性的樣本
數量（探針命中次數）會記入各區域、各探針專屬的計數器。使用者可在彙整
間隔結束後讀取命中計數器，了解區域中有多少記憶體具有特定屬性。

此機制以取樣為基礎，負擔低，但可能包含測量誤差。使用者應充分理解其
統計意義後再使用輸出。

另一種較準確的方法，是使用 ``stat``
:ref:`動作 <damon_design_damos_action_zh_TW>` 搭配
:ref:`DAMOS 篩選器 <damon_design_damos_filters_zh_TW>`，再取得
``sz_ops_filter_passed`` :ref:`統計資料 <damon_design_damos_stat_zh_TW>`。
此方法提供頁面層級的屬性資訊，但由於逐頁處理，負擔與記憶體大小成正比。

.. _damon_design_attrs_only_monitoring_zh_TW:

僅監測資料屬性
~~~~~~~~~~~~~~

DAMON 主要監測的是資料存取，因此使用存取
:ref:`計數器 <damon_design_region_based_sampling_zh_TW>` （``nr_accesses``）
:ref:`調整區域 <damon_design_adaptive_regions_adjustment_zh_TW>`。
但有些使用情境希望以其他
:ref:`資料屬性 <damon_design_data_attrs_monitoring_zh_TW>` 作為主要資訊。

僅監測資料屬性模式支援此需求。每個屬性探針都有優先權重，使用者可藉由
設定權重，指定以哪些屬性的組合決定主要資訊。若權重總和不為零，即啟用
此模式；區域調整機制便改用
:ref:`探針命中次數 <damon_design_data_attrs_monitoring_zh_TW>` 的加權總和，
而不使用 ``nr_accesses``。

啟用此模式時，存取監測會自動關閉，存取計數器（``nr_accesses``）恆為零，
不再更新，因此稱為「僅」監測資料屬性。

使用方式請參閱 :ref:`管理指南 <damon_usage_sysfs_probes_zh_TW>`。

處理目標空間的動態更新
~~~~~~~~~~~~~~~~~~~~~~

監測目標位址範圍可能動態變化，例如虛擬記憶體的映射與解除映射，以及
實體記憶體的熱插拔。

由於這些變化有時非常頻繁，DAMON 允許監測操作檢查包括記憶體映射在內的
動態變化，但只在使用者指定的時間間隔（``更新間隔``），將變化套用至
監測操作相關的資料結構，例如抽象化的監測目標記憶體區域。

使用者空間可透過 DAMON sysfs 介面或追蹤點取得監測結果，詳情請分別參閱
:ref:`DAMOS 嘗試區域 <sysfs_schemes_tried_regions_zh_TW>` 與
:ref:`追蹤點 <tracepoint_zh_TW>` 文件。

.. _damon_design_monitoring_params_tuning_guide_zh_TW:

監測參數調校指南
~~~~~~~~~~~~~~~~

簡而言之，應將 ``彙整間隔`` 設為能擷取符合使用目的之存取量的時間。
可用彙整後監測結果快照中各區域的 ``nr_accesses`` 與 ``age`` 衡量存取量。
預設的 ``100ms`` 在許多情況下太短。``取樣間隔`` 應與 ``彙整間隔``
成比例，建議採用預設的 ``1/20``。

``彙整間隔`` 應足以讓工作負載產生監測目的所需的存取量。若間隔太短，
只能擷取少量存取，結果看起來會像所有區域都同樣很少被存取，對許多用途
並無幫助。若間隔太長，則視使用目的的時間尺度而定，
:ref:`區域調整機制 <damon_design_adaptive_regions_adjustment_zh_TW>`
可能需要太久才能收斂。例如，工作負載實際上很少存取記憶體，但使用者
設定了過高的監測存取量目標時，就可能發生這種情況。此時應重新評估每個
彙整間隔要擷取的存取量。

另外，存取量不僅由 ``nr_accesses`` 表示，也反映在 ``age`` 中。
即使所有區域的 ``nr_accesses`` 都為零，仍可利用 ``age`` 所提供的最近
存取情形來區分區域。

因此，最佳的 ``彙整間隔`` 取決於工作負載的存取密集程度，使用者應根據
各彙整快照擷取的存取量調校間隔。預設的 100 毫秒在許多情況下太短，
尤其是大型系統。

``取樣間隔`` 決定每次彙整的解析度。若設得太大，結果看起來會像每個區域
都同樣很少被存取，或同樣頻繁地被存取，無法根據存取模式區分區域，因此
對許多用途沒有幫助。若設得太小，雖不會降低解析度，卻會增加監測負擔。
只要已能提供符合使用目的的解析度，就不應再縮短。建議將取樣間隔設為
彙整間隔的固定比例，預設的 ``1/20`` 仍是建議值。

根據此手動調校指南，DAMON 提供以較直覺的控制參數為基礎的間隔自動調校
機制。詳情請參閱 :ref:`自動調校設計
<damon_design_monitoring_intervals_autotuning_zh_TW>`。

依此指南進行調校的範例，請參閱英文文件
:doc:`/mm/damon/monitoring_intervals_tuning_example`。

.. _damon_design_monitoring_intervals_autotuning_zh_TW:

監測間隔自動調校
~~~~~~~~~~~~~~~~

DAMON 根據 :ref:`調校指南 <damon_design_monitoring_params_tuning_guide_zh_TW>`
的概念，自動調校 ``取樣間隔`` 與 ``彙整間隔``。使用者可指定在一段時間內
希望 DAMON 觀測到的存取事件量，以若干次彙整（``aggrs``）期間，DAMON
觀測到的事件數相對於理論最大事件數的比率（``access_bp``）表示。

DAMON 根據 :ref:`區域假設 <damon_design_region_based_sampling_zh_TW>`，
以位元組為粒度計算存取事件。例如，若區域大小為 ``X`` 位元組，
``nr_accesses`` 為 ``Y``，表示觀測到 ``X * Y`` 個事件。理論最大事件數
以相同方式計算，但將 ``Y`` 換成理論最大的 ``nr_accesses``，也就是
``彙整間隔 / 取樣間隔``。

此機制計算 ``aggrs`` 次彙整期間的存取事件比率。若比率低於目標，就以
相同比例增大取樣與彙整間隔；若高於目標，就以相同比例縮小兩個間隔。
間隔的調整幅度與目前取樣比率和目標比率的差距成比例。

使用者還可透過 ``min_sample_us`` 與 ``max_sample_us``，設定調校機制
允許的取樣間隔下限與上限。由於兩個間隔一律以相同比例調整，彙整間隔的
下限與上限也會隨之決定。

自動調校預設關閉，需由使用者明確啟用。根據經驗法則與帕累托原則，建議
以 4% 作為存取樣本比率目標。這裡將帕累托原則（80/20 法則）套用兩次：
假設 4%（20% 乘以 20%）的 DAMON 觀測存取事件比率（來源），可擷取 64%
（80% 乘以 80%）的實際存取事件（結果）。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 使用，
請參閱 :ref:`intervals_goal <damon_usage_sysfs_monitoring_intervals_goal_zh_TW>`。

.. _damon_design_damos_zh_TW:

操作方案
--------

資料存取監測的常見用途之一，是以存取感知方式改善系統效率。例如：

    換出超過兩分鐘未被存取的記憶體區域。

或是：

    對大於 2 MiB、且高存取頻率維持超過一分鐘的區域使用 THP。

直接的做法是以剖析結果引導最佳化：使用 DAMON 取得工作負載或系統的監測
結果，分析出具有特定特徵的區域，再調整針對這些區域的系統操作。可以修改
軟體（應用程式或核心）、向軟體提供建議，或重新設定硬體。離線與線上方式
都可行。

其中，在執行期間向核心提供建議較有彈性且有效，因此廣泛使用。但實作
這類方案可能帶來不必要的重複工作與低效率。若感興趣的類型很常見，剖析
工作可能重複；在核心與使用者空間之間交換監測結果及操作建議，也可能
效率不佳。

為讓使用者將這些工作交給 DAMON，減少重複與低效率，DAMON 提供「基於
資料存取監測的操作方案」（DAMOS）。使用者可用高階方式指定期望的方案。
DAMON 便開始監測、找出符合目標存取模式的區域，並在每個使用者指定的
時間間隔（``apply_interval``），對這些區域套用指定的操作動作。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
``apply_interval``，請參閱 :ref:`apply_interval_us <sysfs_scheme_zh_TW>`。

.. _damon_design_damos_action_zh_TW:

操作動作
~~~~~~~~

使用者希望對目標區域套用的管理動作，例如換出、提高下次回收選取的優先
順序、建議 ``khugepaged`` 合併或分割頁面，或不進行操作而只收集統計資料。

支援的動作清單由 DAMOS 定義，但實作位於 DAMON 操作集層，因為各動作通常
依賴監測目標位址空間。例如，換出虛擬位址範圍的程式碼，與換出實體位址
範圍的程式碼不同。操作集也不一定支援清單中的所有動作，因此動作是否
可用，取決於搭配的操作集。

以下列出支援的動作、含義，以及支援該動作的操作集。

- ``willneed``：以 ``MADV_WILLNEED`` 對區域呼叫 ``madvise()``。
  由 ``vaddr`` 與 ``fvaddr`` 支援。
- ``cold``：以 ``MADV_COLD`` 對區域呼叫 ``madvise()``。
  由 ``vaddr`` 與 ``fvaddr`` 支援。
- ``pageout``：回收區域。由 ``vaddr``、``fvaddr`` 與 ``paddr`` 支援。
- ``hugepage``：以 ``MADV_HUGEPAGE`` 對區域呼叫 ``madvise()``。
  由 ``vaddr`` 與 ``fvaddr`` 支援。若未啟用 TRANSPARENT_HUGEPAGE，
  套用此動作會失敗。
- ``nohugepage``：以 ``MADV_NOHUGEPAGE`` 對區域呼叫 ``madvise()``。
  由 ``vaddr`` 與 ``fvaddr`` 支援。若未啟用 TRANSPARENT_HUGEPAGE，
  套用此動作會失敗。
- ``collapse``：以 ``MADV_COLLAPSE`` 對區域呼叫 ``madvise()``。
  由 ``vaddr`` 與 ``fvaddr`` 支援。若未啟用 TRANSPARENT_HUGEPAGE，
  套用此動作會失敗。
- ``lru_prio``：提高區域在 LRU 串列上的優先順序。由 ``paddr`` 支援。
- ``lru_deprio``：降低區域在 LRU 串列上的優先順序。由 ``paddr`` 支援。
- ``migrate_hot``：遷移區域，優先處理較熱的區域。
  由 ``vaddr``、``fvaddr`` 與 ``paddr`` 支援。
- ``migrate_cold``：遷移區域，優先處理較冷的區域。
  由 ``vaddr``、``fvaddr`` 與 ``paddr`` 支援。
- ``stat``：不進行操作，只收集統計資料。所有操作集都支援。

對區域套用 ``stat`` 以外的動作，會被視為改變該區域的特徵，因此 DAMOS
會重設該區域的 ``age``。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
動作，請參閱 :ref:`action <sysfs_scheme_zh_TW>`。

.. _damon_design_damos_access_pattern_zh_TW:

目標存取模式
~~~~~~~~~~~~

方案感興趣的存取模式，由 DAMON 監測結果提供的區域大小、存取頻率與
存取模式持續時間（``age``）構成。使用者可設定這三個屬性的最小值與
最大值，描述目標模式。若區域的三個屬性都落在範圍內，DAMOS 便將其
視為方案感興趣的區域。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
存取模式，請參閱 :ref:`access_pattern <sysfs_access_pattern_zh_TW>`。

.. _damon_design_damos_quotas_zh_TW:

配額
~~~~

配額用於控制 DAMOS 的額外負擔上限。若目標存取模式未適當調校，DAMOS
可能帶來很高的負擔。例如，對符合模式的大型區域中的所有頁面套用動作，
可能消耗過多系統資源。藉由調校存取模式避免此問題並不容易，尤其當
工作負載的存取模式高度動態變化時。

因此，DAMOS 提供配額功能，讓使用者指定在一段時間內，套用動作所能使用
的時間上限，或可套用動作的記憶體區域總位元組數上限。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
基本配額，請參閱 :ref:`quotas <sysfs_quotas_zh_TW>`。

.. _damon_design_damos_quotas_prioritization_zh_TW:

優先順序
^^^^^^^^

此機制協助 DAMOS 在配額限制下作出適當選擇。若配額不足以對所有目標區域
套用動作，DAMOS 會排序區域，只處理優先順序夠高的區域，以免超出配額。

不同動作應有不同的排序機制。例如，換出動作應優先處理較少被存取的冷區域，
而合併大頁的動作則應降低冷區域的優先順序。因此，各動作的排序機制與動作
本身一起實作於 DAMON 操作集中。

雖然實作由操作集決定，但通常會以區域的存取模式屬性計算優先順序。
使用者可能希望依使用情境調整，例如更重視最近存取情形（``age``），而非
存取頻率（``nr_accesses``）。DAMOS 允許指定各屬性的權重，再將資訊傳給
底層機制。然而，是否採用以及如何採用這些權重，仍由底層排序實作決定。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
權重，請參閱 :ref:`weights <sysfs_quotas_zh_TW>`。

.. _damon_design_damos_quotas_failed_memory_charging_ratio_zh_TW:

動作失敗記憶體的配額計入比率
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

對區域套用 DAMOS 動作時，部分記憶體可能處理失敗。例如，``pageout``
無法回收不可回收的頁面。失敗動作消耗的系統資源，通常不同於成功動作。
因此可對處理失敗的記憶體設定不同的配額計入比率，以 ``fail_charge_num``
與 ``fail_charge_denom`` 分別指定分子與分母。只有分母不為零時才啟用。

例如，對 1,000 MiB 區域套用動作，僅成功處理 700 MiB，且分子與分母分別
為 ``1`` 與 ``1024``，則計入配額的大小為 700 MiB 加上 300 KiB：
``700 MiB + 300 MiB * 1 / 1024``。

.. _damon_design_damos_quotas_auto_tuning_zh_TW:

目標導向、回饋驅動的自動調校
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用者不必設定絕對配額值，而可指定感興趣的指標與目標值。DAMOS 會根據
回饋，自動調校對應方案的積極程度（配額）。若尚未達標，就增加配額；
若超過目標，就減少配額。

使用者可選擇兩種調校演算法：

- ``consist``：以比例回饋迴路為基礎，尋找應持續維持的最佳配額，以持續
  達成目標。適用於動態、長時間執行環境中的純核心操作。這是預設選項，
  若不確定應使用哪一種，請選此項。
- ``temporal``：較直接的演算法，短時間使用允許的最大配額，以盡快達標。
  尚未達標時，持續將配額調至允許的最大值；一旦達成或超過目標，就將配額
  設為零。適用於需要確定性控制的環境。

目標由五個參數指定：``target_metric``、``target_value``、``current_value``、
``nid`` 和 ``path``。自動調校機制會嘗試讓 ``target_metric`` 的
``current_value`` 等於 ``target_value``。

- ``user_input``：由使用者提供的值，可使用任何感興趣的指標，例如使用者
  空間主要工作負載的延遲或吞吐量、空閒記憶體比率，或記憶體壓力停滯資訊
  （PSI）。此時使用者必須自行設定 ``current_value``，也就是反覆提供回饋。
- ``some_mem_psi_us``：從上次配額重設到下次重設之間，全系統的 ``some``
  記憶體 PSI，單位為微秒。DAMOS 自行測量，使用者只須在初始時設定
  ``target_value``；換言之，DAMOS 會自行提供回饋。
- ``node_mem_used_bp``：特定 NUMA 節點的已用記憶體比率，單位為 bp
  （萬分之一）。
- ``node_mem_free_bp``：特定 NUMA 節點的空閒記憶體比率，單位為 bp
  （萬分之一）。
- ``node_memcg_used_bp``：特定 cgroup 在指定 NUMA 節點上的已用記憶體
  占該節點記憶體的比率，單位為 bp（萬分之一）。
- ``node_memcg_free_bp``：特定 cgroup 在指定 NUMA 節點上未使用的記憶體
  占該節點記憶體的比率，單位為 bp（萬分之一）。
- ``active_mem_bp``：活躍記憶體占活躍與非活躍 LRU 記憶體總量的比率，
  單位為 bp（萬分之一）。
- ``inactive_mem_bp``：非活躍記憶體占活躍與非活躍 LRU 記憶體總量的比率，
  單位為 bp（萬分之一）。
- ``node_eligible_mem_bp``：節點中符合方案目標存取模式的記憶體比率，
  單位為 bp（萬分之一）。

使用 ``node_mem_used_bp``、``node_mem_free_bp``、``node_memcg_used_bp``、
``node_memcg_free_bp`` 或 ``node_eligible_mem_bp`` 時，需以 ``nid``
指定 NUMA 節點。

僅在使用 ``node_memcg_used_bp`` 或 ``node_memcg_free_bp`` 時，需以 ``path``
指定 cgroup；其值為相對於 cgroup 掛載點的記憶體 cgroup 路徑。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
目標指標、目標值與目前值，請參閱
:ref:`quota goals <sysfs_schemes_quota_goals_zh_TW>`。

.. _damon_design_damos_watermarks_zh_TW:

水位
~~~~

水位用於依條件自動啟動或停止 DAMOS 方案。使用者可能只希望 DAMOS 在
特定情況下執行。例如，空閒記憶體充足時，主動回收只會浪費系統資源。
若沒有此機制，使用者就必須手動監測空閒記憶體比率等指標，並開啟或關閉
DAMON/DAMOS。

DAMOS 允許使用者指定指標，以及高、中、低三個水位，將此工作交由 DAMOS
處理。指標高於高水位或低於低水位時，方案停止作用；低於中水位但高於
低水位時，方案開始作用。若所有方案都因水位而停止作用，監測也會停止。
此時 DAMON 工作執行緒只定期檢查水位，幾乎不產生額外負擔。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
水位，請參閱 :ref:`watermarks <sysfs_watermarks_zh_TW>`。

.. _damon_design_damos_filters_zh_TW:

篩選器
~~~~~~

篩選器依存取模式以外的條件篩選目標記憶體。使用者若執行自行撰寫的程式，
或有良好的剖析工具，可能比核心更了解未來的存取模式，或特定記憶體類型
的特殊需求。例如，使用者可能知道只有匿名頁影響效能，或持有對延遲敏感
的行程清單。

DAMOS 篩選器讓使用者利用這些資訊最佳化方案。每個方案可設定任意數量的
篩選器，各篩選器指定：

- 記憶體類型（``type``）。
- 針對該類型，還是該類型以外的所有記憶體（``matching``）。
- 允許（包含）或拒絕（排除）對該記憶體套用方案動作（``allow``）。

為提高效率，部分篩選器由核心邏輯層處理，其他則由操作集處理。後者是否
支援特定類型，取決於操作集。被核心邏輯層篩選器排除的區域，不會計入
方案嘗試處理的區域；被操作集層篩選器排除的區域則會計入。此差異會影響
統計資料。

安裝多個篩選器時，先評估核心邏輯層的篩選器，再評估操作集層的篩選器。
各組內依安裝順序評估。一旦某部分記憶體符合其中一個篩選器，就忽略
該組後續的篩選器。若未符合任何篩選器，則由最後一個篩選器的允許類型
決定：最後一個為允許時，拒絕該記憶體；最後一個為拒絕時，則允許。

例如，依序安裝允許匿名頁、拒絕 young 頁的兩個篩選器。對符合方案條件的
區域，匿名頁不論是否為 young，都符合第一個允許篩選器，因此會套用動作。
非匿名但為 young 的頁面則被第二個篩選器拒絕。若既非匿名也非 young，
因不符合任何篩選器，且最後一個篩選器為拒絕類型，反而會套用動作。

目前支援下列 ``type``：

- 核心邏輯層處理：

  - ``addr``：屬於指定位址範圍的頁面。
  - ``target``：屬於指定 DAMON 監測目標的頁面。

- 操作集層處理，僅 ``paddr`` 支援：

  - ``anon``：包含未儲存於檔案之資料的頁面。
  - ``active``：活躍頁。
  - ``memcg``：屬於指定 cgroup 的頁面。
  - ``young``：自方案上次存取檢查後被存取過的頁面。
  - ``hugepage_size``：以指定大小範圍管理的頁面。
  - ``unmapped``：未映射的頁面。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 設定
篩選器，請參閱 :ref:`filters <sysfs_filters_zh_TW>`。

.. _damon_design_damos_stat_zh_TW:

統計資料
~~~~~~~~

DAMOS 行為的統計資料，有助於監測、調校與除錯。每個方案從開始執行起，
累計下列統計資料：

- ``nr_tried``：方案嘗試套用的區域總數。
- ``sz_tried``：方案嘗試套用的區域總大小。
- ``sz_ops_filter_passed``：通過操作集層 DAMOS 篩選器的總位元組數。
- ``nr_applied``：方案成功套用的區域總數。
- ``sz_applied``：方案成功套用的區域總大小。
- ``qt_exceeds``：超過方案配額的總次數。
- ``nr_snapshots``：方案嘗試套用的 DAMON 快照總數。
- ``max_nr_snapshots``：``nr_snapshots`` 的上限。

「嘗試套用方案至區域」表示 DAMOS 核心邏輯判定區域符合套用方案
:ref:`動作 <damon_design_damos_action_zh_TW>` 的條件。
核心邏輯處理的 :ref:`存取模式 <damon_design_damos_access_pattern_zh_TW>`、
:ref:`配額 <damon_design_damos_quotas_zh_TW>`、
:ref:`水位 <damon_design_damos_watermarks_zh_TW>` 和
:ref:`篩選器 <damon_design_damos_filters_zh_TW>` 都可能影響此判定。
核心邏輯只是請求底層 :ref:`操作集 <damon_operations_set_zh_TW>` 套用動作，
不確定是否真的成功，因此稱為「嘗試」。

「方案已套用至區域」表示 :ref:`操作集 <damon_operations_set_zh_TW>` 已對
至少部分區域成功套用動作。操作集處理的
:ref:`篩選器 <damon_design_damos_filters_zh_TW>`、
:ref:`動作 <damon_design_damos_action_zh_TW>` 類型與頁面類型都可能影響結果。
例如，篩選器排除匿名頁，而區域中只有匿名頁；或動作為 ``pageout``，但
區域中的頁面都不可回收，套用動作就會失敗。

不同於一般統計值，``max_nr_snapshots`` 由使用者設定。若不為零，且
``nr_snapshots`` 大於或等於 ``max_nr_snapshots``，方案會停止作用。

使用者空間如何透過 :ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 讀取
統計資料，請參閱 :ref:`stats <sysfs_schemes_stats_zh_TW>`。

走訪區域
~~~~~~~~

此 DAMOS 功能讓使用者存取剛套用過 DAMOS 動作的各個區域。透過 DAMON
:ref:`API <damon_design_api_zh_TW>`，可取得區域的完整屬性，包括存取監測
結果，以及區域內通過 DAMOS 篩選器的記憶體量。
:ref:`DAMON sysfs 介面 <sysfs_interface_zh_TW>` 也提供特殊
:ref:`檔案 <sysfs_schemes_tried_regions_zh_TW>`，供使用者讀取這些資料。

.. _damon_design_api_zh_TW:

應用程式介面
------------

這是供核心空間的資料存取感知應用使用的程式設計介面。DAMON 是框架，
本身不執行任何工作，而是協助子系統、模組等核心元件，利用其核心功能
建構資料存取感知應用。為此，DAMON 透過 ``include/linux/damon.h`` 提供
所有功能。介面細節請參閱 API :doc:`文件 </mm/damon/api>`。

.. _damon_modules_zh_TW:

模組
====

DAMON 的核心部分是供核心元件使用的框架，不直接提供使用者空間介面。
介面應由使用 DAMON API 的各核心元件實作。DAMON 子系統也實作這類模組，
供通用 DAMON 控制及專用資料存取感知系統操作使用，並提供穩定的使用者
空間應用程式二進位介面（ABI）。使用者空間可藉此建構高效率的資料存取
感知應用。

通用使用者介面模組
------------------

這類 DAMON 模組提供使用者空間 ABI，供執行期間的一般 DAMON 用途使用。

如同其他 ABI，模組會在 ``sysfs`` 等虛擬檔案系統建立檔案，讓使用者透過
寫入提出請求，透過讀取取得回應。DAMON 使用者介面模組收到 I/O 請求後，
便透過 DAMON API 依請求控制 DAMON、取得結果，再傳回使用者空間。

這些 ABI 是為使用者空間應用程式開發而設計，並非供人手動操作。建議使用
使用者空間工具。一個以 Python 撰寫的工具可從 GitHub
（https://github.com/damonitor/damo）、PyPI
（https://pypistats.org/packages/damo），以及多個發行版
（https://repology.org/project/damo/versions）取得。

目前有一個此類模組，即 ``DAMON sysfs interface``。
詳情請參閱 ABI :ref:`文件 <sysfs_interface_zh_TW>`。

.. _damon_modules_special_purpose_zh_TW:

專用存取感知核心模組
--------------------

這類 DAMON 模組提供特定用途的使用者空間 ABI。

DAMON 使用者介面模組可在執行期間完整控制所有 DAMON 功能。但對主動回收
或 LRU 串列平衡等全系統的特定存取感知操作，可移除不必要的控制參數，
簡化介面，並擴充至開機時甚至編譯時控制。此類用途的控制參數預設值也
應針對目的最佳化。

因此，DAMON 還提供介面更簡單、經過最佳化的 DAMON API 使用者核心模組。
目前有存取監測統計、主動回收及 LRU 串列操作三種模組。詳情請參閱
:doc:`../../admin-guide/mm/damon/stat`、
:doc:`../../admin-guide/mm/damon/reclaim` 與
:doc:`../../admin-guide/mm/damon/lru_sort`。

.. _damon_design_special_purpose_modules_exclusivity_zh_TW:

這些模組目前以互斥方式執行。若其中一個已在執行，其他模組收到啟動請求
時會傳回 ``-EBUSY``。

DAMON 範例模組
--------------

這類模組示範如何使用 DAMON 核心 API。

核心程式設計者可使用 DAMON API 建構自己的專用或通用模組。為協助理解
API 用法，Linux 原始碼樹的 ``samples/damon/`` 目錄提供了數個範例模組。
這些模組並非供實際產品使用，只是示範如何以簡單方式使用 DAMON 核心 API。
