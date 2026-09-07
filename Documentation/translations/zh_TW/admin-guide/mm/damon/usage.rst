.. SPDX-License-Identifier: GPL-2.0
.. include:: ../../../disclaimer-zh_TW.rst

:Original: Documentation/admin-guide/mm/damon/usage.rst

:翻譯:

 司延騰 Yanteng Si <siyanteng@loongson.cn>
 Doehyun Baek <doehyunbaek@gmail.com>

:校譯:

========
詳細用法
========

DAMON 為不同使用者提供下列介面：

- *專用 DAMON 模組。*
  :ref:`此介面 <damon_modules_special_purpose_zh_TW>` 適合建置、散布或管理
  具有特定 DAMON 用途之核心的人員。使用者可在建置、開機或執行期間，
  以簡單方式使用 DAMON 的主要功能，達成特定目的。
- *DAMON 使用者空間工具。*
  `此工具 <https://github.com/damonitor/damo>`_ 適合系統管理員等具有權限、
  且希望使用簡單易用介面的使用者。它以方便操作的方式提供 DAMON 主要功能，
  但不一定針對特殊情境高度調校。詳情請參閱其 `使用文件
  <https://github.com/damonitor/damo/blob/next/USAGE.md>`_。
- *sysfs 介面。*
  :ref:`此介面 <sysfs_interface_zh_TW>` 適合具有權限、希望更有效運用 DAMON
  的使用者空間程式設計者。透過讀寫特殊的 sysfs 檔案，可使用 DAMON 的
  主要功能，也可撰寫客製化的 DAMON sysfs 封裝程式，代為讀寫檔案。
  `DAMON 使用者空間工具 <https://github.com/damonitor/damo>`_ 就是一例。
- *核心空間程式設計介面。*
  :doc:`此介面 </mm/damon/api>` 適合核心空間程式設計者。撰寫核心空間的
  DAMON 應用程式，可彈性且有效率地使用所有功能，甚至將 DAMON 擴充至
  不同位址空間。詳情請參閱 API :doc:`文件 </mm/damon/api>`。

.. _sysfs_interface_zh_TW:

sysfs 介面
==========

定義 ``CONFIG_DAMON_SYSFS`` 時，核心會建置 DAMON sysfs 介面，在
``<sysfs>/kernel/mm/damon/`` 下建立多個目錄與檔案。讀寫這些檔案即可
控制 DAMON。

例如，可用下列命令監測指定工作負載的虛擬位址空間：::

    # cd /sys/kernel/mm/damon/admin/
    # echo 1 > kdamonds/nr_kdamonds && echo 1 > kdamonds/0/contexts/nr_contexts
    # echo vaddr > kdamonds/0/contexts/0/operations
    # echo 1 > kdamonds/0/contexts/0/targets/nr_targets
    # echo $(pidof <workload>) > kdamonds/0/contexts/0/targets/0/pid_target
    # echo on > kdamonds/0/state

檔案階層
--------

下圖以縮排表示父子關係，目錄以 ``/`` 結尾，各目錄中的檔案以逗號分隔。

.. parsed-literal::

    :ref:`/sys/kernel/mm/damon <sysfs_root_zh_TW>`/admin
    │ :ref:`kdamonds <sysfs_kdamonds_zh_TW>`/nr_kdamonds
    │ │ :ref:`0 <sysfs_kdamond_zh_TW>`/state,pid,refresh_ms
    │ │ │ :ref:`contexts <sysfs_contexts_zh_TW>`/nr_contexts
    │ │ │ │ :ref:`0 <sysfs_context_zh_TW>`/avail_operations,operations,addr_unit,
    │ │ │ │   pause
    │ │ │ │ │ :ref:`monitoring_attrs <sysfs_monitoring_attrs_zh_TW>`/
    │ │ │ │ │ │ intervals/sample_us,aggr_us,update_us
    │ │ │ │ │ │ │ intervals_goal/access_bp,aggrs,min_sample_us,max_sample_us
    │ │ │ │ │ │ nr_regions/min,max
    │ │ │ │ │ │ :ref:`probes <damon_usage_sysfs_probes_zh_TW>`/nr_probes
    │ │ │ │ │ │ │ 0/weight
    │ │ │ │ │ │ │ │ filters/nr_filters
    │ │ │ │ │ │ │ │ │ 0/type,matching,allow,path
    │ │ │ │ │ │ │ │ │ ...
    │ │ │ │ │ │ │ ...
    │ │ │ │ │ :ref:`targets <sysfs_targets_zh_TW>`/nr_targets
    │ │ │ │ │ │ :ref:`0 <sysfs_target_zh_TW>`/pid_target,obsolete_target
    │ │ │ │ │ │ │ :ref:`regions <sysfs_regions_zh_TW>`/nr_regions
    │ │ │ │ │ │ │ │ :ref:`0 <sysfs_region_zh_TW>`/start,end
    │ │ │ │ │ │ │ │ ...
    │ │ │ │ │ │ ...
    │ │ │ │ │ :ref:`schemes <sysfs_schemes_zh_TW>`/nr_schemes
    │ │ │ │ │ │ :ref:`0 <sysfs_scheme_zh_TW>`/action,target_nid,apply_interval_us
    │ │ │ │ │ │ │ :ref:`access_pattern <sysfs_access_pattern_zh_TW>`/
    │ │ │ │ │ │ │ │ sz/min,max
    │ │ │ │ │ │ │ │ nr_accesses/min,max
    │ │ │ │ │ │ │ │ age/min,max
    │ │ │ │ │ │ │ :ref:`quotas <sysfs_quotas_zh_TW>`/ms,bytes,reset_interval_ms,
    │ │ │ │ │ │ │     effective_bytes,goal_tuner,
    │ │ │ │ │ │ │     fail_charge_num,fail_charge_denom
    │ │ │ │ │ │ │ │ weights/sz_permil,nr_accesses_permil,age_permil
    │ │ │ │ │ │ │ │ :ref:`goals <sysfs_schemes_quota_goals_zh_TW>`/nr_goals
    │ │ │ │ │ │ │ │ │ 0/target_metric,target_value,current_value,nid,path
    │ │ │ │ │ │ │ :ref:`watermarks <sysfs_watermarks_zh_TW>`/metric,interval_us,high,mid,low
    │ │ │ │ │ │ │ :ref:`{core_,ops_,}filters <sysfs_filters_zh_TW>`/nr_filters
    │ │ │ │ │ │ │ │ 0/type,matching,allow,memcg_path,addr_start,addr_end,damon_target_idx,min,max
    │ │ │ │ │ │ │ :ref:`dests <damon_sysfs_dests_zh_TW>`/nr_dests
    │ │ │ │ │ │ │ │ 0/id,weight
    │ │ │ │ │ │ │ :ref:`stats <sysfs_schemes_stats_zh_TW>`/nr_tried,sz_tried,nr_applied,sz_applied,sz_ops_filter_passed,qt_exceeds,nr_snapshots,max_nr_snapshots
    │ │ │ │ │ │ │ :ref:`tried_regions <sysfs_schemes_tried_regions_zh_TW>`/total_bytes
    │ │ │ │ │ │ │ │ 0/start,end,nr_accesses,age,sz_filter_passed
    │ │ │ │ │ │ │ │ │ probes
    │ │ │ │ │ │ │ │ │ │ 0/hits
    │ │ │ │ │ │ │ │ │ │ ...
    │ │ │ │ │ │ │ │ ...
    │ │ │ │ │ │ ...
    │ │ │ │ ...
    │ │ ...

.. _sysfs_root_zh_TW:

根目錄
------

DAMON sysfs 介面的根目錄是 ``<sysfs>/kernel/mm/damon/``，其中有一個
``admin`` 目錄，包含供具有權限的使用者空間程式控制 DAMON 的檔案。
具有 root 權限的使用者空間工具或常駐程式可使用此目錄。

.. _sysfs_kdamonds_zh_TW:

kdamonds/
---------

``admin`` 下的 ``kdamonds`` 目錄，包含控制 kdamond 的檔案。
kdamond 的詳情請參閱 :ref:`設計
<damon_design_execution_model_and_data_structures_zh_TW>`。
初始時只有 ``nr_kdamonds`` 檔案，寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個 kdamond。

.. _sysfs_kdamond_zh_TW:

kdamonds/<N>/
-------------

各 kdamond 目錄有 ``state``、``pid``、``refresh_ms`` 三個檔案，以及
``contexts`` 目錄。

讀取 ``state`` 時，若 kdamond 正在執行則傳回 ``on``，否則傳回 ``off``。

使用者可將下列命令寫入 ``state``：

- ``on``：開始執行。
- ``off``：停止執行。
- ``commit``：重新讀取 ``state`` 以外的 sysfs 檔案中的使用者輸入。
  若未指定監測 :ref:`目標區域 <sysfs_regions_zh_TW>`，則也忽略目標區域
  的輸入。
- ``update_tuned_intervals``：以自動調校後的取樣與彙整間隔，更新此
  kdamond 的 ``sample_us`` 和 ``aggr_us`` 檔案。詳情請參閱
  :ref:`intervals_goal <damon_usage_sysfs_monitoring_intervals_goal_zh_TW>`。
- ``commit_schemes_quota_goals``：讀取基於 DAMON 的操作方案的
  :ref:`配額目標 <sysfs_schemes_quota_goals_zh_TW>`。
- ``update_schemes_stats``：更新此 kdamond 各方案的統計檔案。
  詳情請參閱 :ref:`stats <sysfs_schemes_stats_zh_TW>`。
- ``update_schemes_tried_regions``：更新此 kdamond 各方案嘗試套用動作的
  區域目錄。詳情請參閱 :ref:`tried_regions <sysfs_schemes_tried_regions_zh_TW>`。
- ``update_schemes_tried_bytes``：僅更新 ``.../tried_regions/total_bytes``。
- ``clear_schemes_tried_regions``：清除此 kdamond 各方案嘗試套用動作的
  區域目錄。
- ``update_schemes_effective_quotas``：更新此 kdamond 各方案的
  ``effective_bytes`` 檔案。詳情請參閱 :ref:`quotas <sysfs_quotas_zh_TW>`。

若狀態為 ``on``，讀取 ``pid`` 可取得 kdamond 執行緒的 PID。

使用者不必手動將 ``update_tuned_intervals`` 等關鍵字寫入 ``state``，
也可要求核心定期更新顯示自動調校參數與 DAMOS 統計資料的檔案。
將期望的更新間隔（毫秒）寫入 ``refresh_ms`` 即可。設為零會停用定期更新；
讀取此檔案則顯示目前的間隔設定。

``contexts`` 目錄包含控制此 kdamond 所執行的監測情境的檔案。

.. _sysfs_contexts_zh_TW:

kdamonds/<N>/contexts/
----------------------

初始時只有 ``nr_contexts`` 檔案。寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個監測情境。詳情請參閱
:ref:`設計 <damon_design_execution_model_and_data_structures_zh_TW>`。
目前每個 kdamond 只支援一個情境，因此只能寫入 ``0`` 或 ``1``。

.. _sysfs_context_zh_TW:

contexts/<N>/
-------------

各情境目錄有四個檔案：``avail_operations``、``operations``、``addr_unit``
和 ``pause``，以及三個目錄：``monitoring_attrs``、``targets`` 和 ``schemes``。

DAMON 支援多種 :ref:`監測操作 <damon_design_configurable_operations_set_zh_TW>`，
包括虛擬與實體位址空間監測。讀取 ``avail_operations``，可取得目前執行
的核心所支援的操作集清單；此清單依核心組態而異。所有可用操作集及其
簡要說明，請參閱 :ref:`設計 <damon_operations_set_zh_TW>`。

將 ``avail_operations`` 所列的關鍵字寫入 ``operations``，可選擇此情境
使用的操作集；讀取 ``operations`` 則取得目前的選擇。

``addr_unit`` 用於設定及取得操作集的
:ref:`位址單位 <damon_design_addr_unit_zh_TW>` 參數。

``pause`` 用於設定及取得此情境的 :ref:`暫停請求
<damon_design_execution_model_and_data_structures_zh_TW>` 參數。

.. _sysfs_monitoring_attrs_zh_TW:

contexts/<N>/monitoring_attrs/
------------------------------

``monitoring_attrs`` 包含指定監測屬性的檔案，用於設定所需的監測品質與
效率。其中有 ``intervals``、``nr_regions`` 和 ``probes`` 三個目錄。

``intervals`` 包含取樣間隔（``sample_us``）、彙整間隔（``aggr_us``）與
更新間隔（``update_us``）三個檔案。讀寫這些檔案，可取得或設定以微秒為
單位的值。

``nr_regions`` 包含監測區域數下限（``min``）與上限（``max``）兩個檔案，
用來控制監測的額外負擔。讀寫檔案可取得或設定這些值。

間隔與區域數範圍的詳情，請參閱
:ref:`監測設計 <damon_design_monitoring_zh_TW>`。

.. _damon_usage_sysfs_monitoring_intervals_goal_zh_TW:

contexts/<N>/monitoring_attrs/intervals/intervals_goal/
-------------------------------------------------------

``intervals`` 下另有 ``intervals_goal``，用於自動調校 ``sample_us`` 和
``aggr_us``。其中的 ``access_bp``、``aggrs``、``min_sample_us`` 和
``max_sample_us`` 四個檔案，控制自動調校。機制細節請參閱
:ref:`設計 <damon_design_monitoring_intervals_autotuning_zh_TW>`。
讀寫這四個檔案，可取得或更新設計文件中同名的調校參數。

調校從使用者設定的 ``sample_us`` 和 ``aggr_us`` 開始。將
``update_tuned_intervals`` 寫入 ``state`` 後，便可從 ``sample_us`` 和
``aggr_us`` 讀取調校後的目前間隔值。

.. _damon_usage_sysfs_probes_zh_TW:

contexts/<N>/monitoring_attrs/probes/
-------------------------------------

用於註冊 :ref:`資料屬性監測 <damon_design_data_attrs_monitoring_zh_TW>` 探針。

初始時只有 ``nr_probes`` 檔案。寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個監測探針。

各探針目錄有 ``filters`` 目錄，包含為探針安裝篩選器的檔案，以定義探針
要識別的資料屬性。

各探針目錄另有 ``weight`` 檔案，讀寫可取得或設定探針屬性在
:ref:`僅監測資料屬性 <damon_design_attrs_only_monitoring_zh_TW>` 模式中的權重。

``filters`` 初始只有 ``nr_filters`` 檔案。寫入數字 ``N`` 會建立 ``N``
個子目錄，依序命名為 ``0`` 至 ``N-1``，各代表一個篩選器。其運作方式
類似 :ref:`DAMOS 篩選器 <sysfs_filters_zh_TW>`。當 ``type`` 為 ``memcg``，
``path`` 檔案的作用相當於 DAMOS 篩選器的 ``memcg_path``。

.. _sysfs_targets_zh_TW:

contexts/<N>/targets/
---------------------

初始時只有 ``nr_targets`` 檔案。寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個監測目標。

.. _sysfs_target_zh_TW:

targets/<N>/
------------

各目標目錄有 ``pid_target`` 與 ``obsolete_target`` 兩個檔案，以及
``regions`` 目錄。

若將 ``vaddr`` 寫入 ``contexts/<N>/operations``，各目標應為一個行程。
將行程的 PID 寫入 ``pid_target``，即可指定 DAMON 要監測的行程。

將非零值寫入 ``obsolete_target``，再提交設定（將 ``commit`` 寫入
``state``），可選擇性地移除目標陣列中間的目標。DAMON 會從內部目標陣列
移除對應目標，使用者則必須重新建立目標目錄，正確反映變更後的內部陣列。

.. _sysfs_regions_zh_TW:

targets/<N>/regions
-------------------

使用 ``fvaddr`` 或 ``paddr`` 操作集時，必須設定監測目標位址範圍。
使用 ``vaddr`` 時則非必要，但可選擇設定初始監測區域的位址範圍。
詳情請參閱 :ref:`設計 <damon_design_vaddr_target_regions_construction_zh_TW>`。

使用者可將適當的值寫入此目錄下的檔案，明確指定初始監測目標區域。

初始時只有 ``nr_regions`` 檔案。寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個初始監測目標區域。

執行期間提交新參數（將 ``commit`` 寫入 :ref:`kdamond <sysfs_kdamond_zh_TW>`
的 ``state``）時，若 ``nr_regions`` 為零，提交邏輯會忽略目標區域設定，
也就是保留該目標目前的監測結果。

.. _sysfs_region_zh_TW:

regions/<N>/
------------

各區域目錄有 ``start`` 與 ``end`` 兩個檔案。讀寫可分別取得或設定初始
監測目標區域的起始與結束位址。

區域之間不得重疊。目錄 ``N`` 的 ``end`` 應小於或等於目錄 ``N+1`` 的
``start``。

.. _sysfs_schemes_zh_TW:

contexts/<N>/schemes/
---------------------

用於基於 DAMON 的操作方案（:ref:`DAMOS <damon_design_damos_zh_TW>`）的目錄。
使用者可透過讀寫檔案取得或設定方案。

初始時只有 ``nr_schemes`` 檔案。寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個方案。

.. _sysfs_scheme_zh_TW:

schemes/<N>/
------------

各方案目錄有九個目錄：``access_pattern``、``quotas``、``watermarks``、
``core_filters``、``ops_filters``、``filters``、``dests``、``stats`` 和
``tried_regions``，以及三個檔案：``action``、``target_nid`` 和
``apply_interval_us``。

``action`` 用於設定及取得方案的 :ref:`動作 <damon_design_damos_action_zh_TW>`。
可讀寫的關鍵字及其含義，與設計文件中的動作清單相同。

``target_nid`` 用於設定遷移目標節點，僅在動作為 ``migrate_hot`` 或
``migrate_cold`` 時有意義。

``apply_interval_us`` 用於設定及取得方案的
:ref:`apply_interval <damon_design_damos_zh_TW>`，單位為微秒。

.. _sysfs_access_pattern_zh_TW:

schemes/<N>/access_pattern/
---------------------------

用於方案的 :ref:`目標存取模式 <damon_design_damos_access_pattern_zh_TW>`。

``access_pattern`` 下有 ``sz``、``nr_accesses`` 和 ``age`` 三個目錄，
各有 ``min`` 和 ``max`` 兩個檔案。讀寫可分別取得或設定區域大小、存取
次數與存取模式持續時間的範圍。``min`` 與 ``max`` 構成包含兩端點的閉區間。

.. _sysfs_quotas_zh_TW:

schemes/<N>/quotas/
-------------------

用於方案的 :ref:`配額 <damon_design_damos_quotas_zh_TW>`。

``quotas`` 下有七個檔案：``ms``、``bytes``、``reset_interval_ms``、
``effective_bytes``、``goal_tuner``、``fail_charge_num`` 和 ``fail_charge_denom``，
以及 ``weights`` 和 ``goals`` 兩個目錄。

將值分別寫入 ``ms``、``bytes`` 和 ``reset_interval_ms``，可設定時間配額
（毫秒）、大小配額（位元組）與重設間隔（毫秒）。在每個重設間隔內，
DAMON 最多使用 ``ms`` 毫秒，對符合 ``access_pattern`` 的區域套用 ``action``，
且最多處理 ``bytes`` 位元組。將 ``ms`` 與 ``bytes`` 都設為零會停用配額
限制，除非設定了至少一個 :ref:`目標 <sysfs_schemes_quota_goals_zh_TW>`。

將演算法名稱寫入 ``goal_tuner``，可選擇根據目標自動調校有效配額的演算法；
讀取則傳回目前選擇。設計背景與可用演算法名稱，請參閱
:ref:`自動配額調校 <damon_design_damos_quotas_auto_tuning_zh_TW>`。
目標的設定方式請參閱 :ref:`goals <sysfs_schemes_quota_goals_zh_TW>`。

將分子與分母分別寫入 ``fail_charge_num`` 和 ``fail_charge_denom``，可設定
動作處理失敗的記憶體計入配額的比率；讀取則取得目前設定。詳情請參閱
:ref:`設計 <damon_design_damos_quotas_failed_memory_charging_ratio_zh_TW>`。

時間配額在內部會轉換為大小配額，並與使用者指定的大小配額取較小者。
有效大小配額還會根據使用者指定的
:ref:`目標 <sysfs_schemes_quota_goals_zh_TW>` 進一步調整。
讀取 ``effective_bytes`` 可取得目前的有效大小配額。此檔案不會即時更新，
使用者應將 ``update_schemes_effective_quotas`` 寫入對應的
``kdamonds/<N>/state``，要求 sysfs 介面更新其內容。

``weights`` 下有 ``sz_permil``、``nr_accesses_permil`` 和 ``age_permil``
三個檔案。寫入可分別設定區域大小、存取頻率與存取模式持續時間的
:ref:`優先權重 <damon_design_damos_quotas_prioritization_zh_TW>`，
單位為千分之一。

.. _sysfs_schemes_quota_goals_zh_TW:

schemes/<N>/quotas/goals/
-------------------------

用於方案的 :ref:`自動配額調校目標 <damon_design_damos_quotas_auto_tuning_zh_TW>`。

初始時只有 ``nr_goals`` 檔案。寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個目標及其目前達成情形。
若有多個回饋值，採用達成程度最高的一個。

各目標目錄有五個檔案：``target_metric``、``target_value``、``current_value``、
``nid`` 和 ``path``。讀寫可取得或設定
:ref:`設計文件 <damon_design_damos_quotas_auto_tuning_zh_TW>` 說明的五個參數。
核心不會更新 ``current_value``，因此只有當 ``target_metric`` 為
``user_input`` 時，讀取此檔案才有意義。使用者還須將
``commit_schemes_quota_goals`` 寫入 :ref:`kdamond <sysfs_kdamond_zh_TW>` 的
``state``，將回饋傳給 DAMON。

.. _sysfs_watermarks_zh_TW:

schemes/<N>/watermarks/
-----------------------

用於方案的 :ref:`水位 <damon_design_damos_watermarks_zh_TW>`。

此目錄有 ``metric``、``interval_us``、``high``、``mid`` 和 ``low``
五個檔案，分別用於設定指標、指標檢查間隔與三個水位。讀寫可取得或設定
這些值。

``metric`` 可寫入的關鍵字如下：

- ``none``：忽略水位。
- ``free_mem_rate``：系統空閒記憶體比率，單位為千分之一。

``interval_us`` 的單位為微秒。

.. _sysfs_filters_zh_TW:

schemes/<N>/{core\_,ops\_,}filters/
-----------------------------------

用於方案的 :ref:`篩選器 <damon_design_damos_filters_zh_TW>`。

``core_filters`` 與 ``ops_filters`` 分別用於核心邏輯層及操作集層處理的
篩選器。``filters`` 則可安裝任一層處理的篩選器。``core_filters`` 與
``ops_filters`` 要求的篩選器會先於 ``filters`` 的篩選器安裝。
三個目錄具有相同檔案。

``filters`` 會讓篩選器的評估順序難以預期，因此已棄用。它目前仍可使用，
但預計不久後移除。使用者應改用 ``core_filters`` 與 ``ops_filters``。

初始時只有 ``nr_filters`` 檔案。寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個篩選器，並按數字順序評估。

各篩選器目錄有九個檔案：``type``、``matching``、``allow``、``memcg_path``、
``addr_start``、``addr_end``、``min``、``max`` 和 ``damon_target_idx``。
將篩選器類型寫入 ``type``；可用名稱、含義與處理層次，請參閱
:ref:`設計文件 <damon_design_damos_filters_zh_TW>`。

``memcg`` 類型以 ``memcg_path`` 指定相對於 cgroup 掛載點的記憶體 cgroup
路徑。``addr`` 類型以 ``addr_start`` 與 ``addr_end`` 指定位址範圍，包含
起點但不包含終點。``hugepage_size`` 類型以 ``min`` 與 ``max`` 指定大小
範圍，包含兩端點。``target`` 類型以 ``damon_target_idx`` 指定目標在
DAMON 情境的監測目標清單中的索引。

將 ``Y`` 或 ``N`` 寫入 ``matching``，可指定篩選器針對符合或不符合
``type`` 的記憶體。將 ``Y`` 或 ``N`` 寫入 ``allow``，可指定是否允許
對符合 ``type`` 與 ``matching`` 條件的記憶體套用動作。

例如，下列命令僅允許對 ``/having_care_already`` 以外各 cgroup 的
非匿名頁套用 DAMOS 動作。從方案目錄開始執行：::

    # cd ops_filters/
    # echo 2 > nr_filters
    # # disallow anonymous pages
    # echo anon > 0/type
    # echo Y > 0/matching
    # echo N > 0/allow
    # # disallow pages belonging to /having_care_already
    # echo memcg > 1/type
    # echo /having_care_already > 1/memcg_path
    # echo Y > 1/matching
    # echo N > 1/allow

不同 ``allow`` 設定的多個篩選器如何運作、何時支援各類篩選器，以及統計
資料的差異，請參閱 :ref:`DAMOS 篩選器設計 <damon_design_damos_filters_zh_TW>`。

.. _damon_sysfs_dests_zh_TW:

schemes/<N>/dests/
------------------

用於指定方案動作的目的地。若動作不支援多個目的地，此目錄會被忽略。
目前僅 ``DAMOS_MIGRATE_{HOT,COLD}`` 支援多個目的地。

初始時只有 ``nr_dests`` 檔案。寫入數字 ``N`` 會建立 ``N`` 個子目錄，
依序命名為 ``0`` 至 ``N-1``，各代表一個動作目的地。

各目的地目錄有 ``id`` 與 ``weight`` 兩個檔案。``id`` 用於讀寫目的地
識別碼；對 ``DAMOS_MIGRATE_{HOT,COLD}``，應指定遷移目的節點的 ID。
``weight`` 用於讀寫此目的地相對於其他目的地的權重，可為任意整數。
DAMOS 對區域內各實體套用動作時，會依目的地的相對權重選擇目的地。

.. _sysfs_schemes_stats_zh_TW:

schemes/<N>/stats/
------------------

DAMON 為各方案累計統計資料，供執行期間分析或調校使用。
詳情請參閱 :ref:`設計文件 <damon_design_damos_stat_zh_TW>`。

讀取 ``stats`` 下的 ``nr_tried``、``sz_tried``、``nr_applied``、``sz_applied``、
``sz_ops_filter_passed``、``qt_exceeds``、``nr_snapshots`` 和
``max_nr_snapshots``，可取得相應資料。

這些檔案預設不會即時更新。使用者可透過 ``refresh_ms`` 要求定期更新，
或將 ``update_schemes_stats`` 寫入對應的 ``kdamonds/<N>/state``，
進行單次更新。詳情請參閱 :ref:`kdamond <sysfs_kdamond_zh_TW>`。

.. _sysfs_schemes_tried_regions_zh_TW:

schemes/<N>/tried_regions/
--------------------------

此目錄初始時只有 ``total_bytes`` 檔案。

將 ``update_schemes_tried_regions`` 寫入對應的 ``kdamonds/<N>/state`` 後，
DAMON 會更新 ``total_bytes``，使其顯示方案嘗試處理的區域總大小，並建立
從 ``0`` 開始以整數命名的子目錄。各目錄的檔案提供對應方案在下一個
:ref:`套用間隔 <damon_design_damos_zh_TW>` 內，嘗試套用 ``action`` 的
各記憶體區域的詳細資訊，包括位址範圍、``nr_accesses`` 與 ``age``。

將 ``update_schemes_tried_bytes`` 寫入對應的 ``kdamonds/<N>/state``，
只會更新 ``total_bytes``，不建立子目錄。

將 ``clear_schemes_tried_regions`` 寫入對應的 ``kdamonds/<N>/state``，
則會刪除這些子目錄。

此目錄可用來研究方案行為，或以類似查詢的方式有效率地取得監測結果。
後者可將 ``action`` 設為 ``stat``，並將 ``access pattern`` 設為想查詢
的模式。

.. _sysfs_schemes_tried_region_zh_TW:

tried_regions/<N>/
------------------

各區域目錄有 ``start``、``end``、``nr_accesses``、``age`` 與
``sz_filter_passed`` 五個檔案。讀取可取得對應方案嘗試套用 ``action`` 的
區域屬性。

tried_regions/<N>/probes/
-------------------------

各區域目錄另有 ``probes`` 目錄，其中的子目錄依序命名為 ``0`` 至 ``N-1``，
``N`` 為已安裝的探針數。各子目錄有 ``hits`` 檔案，讀取可取得該區域
資料屬性監測探針命中的正樣本數量。

範例
~~~~

下列命令設定此方案：「若區域大小在 [4 KiB, 8 KiB] 內、每個彙整間隔的
存取次數在 [0, 5] 內，且此模式持續了 [10, 20] 個彙整間隔，則換出該
區域。換出操作每秒最多使用 10 毫秒，且最多換出 1 GiB。在此限制下，
優先換出 ``age`` 較大的區域。另外，每 5 秒檢查系統的空閒記憶體比率，
低於 50% 時開始監測與換出；高於 60% 或低於 30% 時則停止。」::

    # cd <sysfs>/kernel/mm/damon/admin
    # # populate directories
    # echo 1 > kdamonds/nr_kdamonds; echo 1 > kdamonds/0/contexts/nr_contexts;
    # echo 1 > kdamonds/0/contexts/0/schemes/nr_schemes
    # cd kdamonds/0/contexts/0/schemes/0
    # # set the basic access pattern and the action
    # echo 4096 > access_pattern/sz/min
    # echo 8192 > access_pattern/sz/max
    # echo 0 > access_pattern/nr_accesses/min
    # echo 5 > access_pattern/nr_accesses/max
    # echo 10 > access_pattern/age/min
    # echo 20 > access_pattern/age/max
    # echo pageout > action
    # # set quotas
    # echo 10 > quotas/ms
    # echo $((1024*1024*1024)) > quotas/bytes
    # echo 1000 > quotas/reset_interval_ms
    # # set watermark
    # echo free_mem_rate > watermarks/metric
    # echo 5000000 > watermarks/interval_us
    # echo 600 > watermarks/high
    # echo 500 > watermarks/mid
    # echo 300 > watermarks/low

強烈建議使用 `damo <https://github.com/damonitor/damo>`_ 等使用者空間工具，
而非手動讀寫上述檔案。以上僅供示範。

.. _tracepoint_zh_TW:

監測結果的追蹤點
================

使用者可透過 :ref:`tried_regions <sysfs_schemes_tried_regions_zh_TW>` 取得
監測快照，但此介面不一定適合完整記錄所有結果。為此，DAMON 提供兩個
追蹤點：``damon:damon_aggregated`` 與 ``damon:damos_before_apply``。
前者提供完整監測結果，後者提供各基於 DAMON 的操作方案
（:ref:`DAMOS <damon_design_damos_zh_TW>`）即將套用動作的區域的監測結果。
因此，後者更適合記錄 DAMOS 內部行為，或依方案的
:ref:`目標存取模式 <damon_design_damos_access_pattern_zh_TW>`，以類似
查詢的方式有效率地記錄結果。

開啟監測後，可記錄追蹤點事件，並用 ``perf`` 等支援追蹤點的工具顯示結果。
例如，先依前述方式設定 kdamond，再執行：::

    # cd /sys/kernel/mm/damon/admin
    # echo on > kdamonds/0/state
    # perf record -e damon:damon_aggregated &
    # perf_pid=$!
    # sleep 5
    # kill -INT "$perf_pid"
    # wait "$perf_pid"
    # echo off > kdamonds/0/state
    # perf script
    kdamond.0 46568 [027] 79357.842179: damon:damon_aggregated: target_id=0 nr_regions=11 122509119488-135708762112: 0 864
    [...]

``perf script`` 每行輸出代表一個監測區域。前五個欄位與一般追蹤點輸出相同。
第六個欄位 ``target_id=X`` 是監測目標 ID，第七個 ``nr_regions=X`` 是該
目標的監測區域總數，第八個 ``X-Y:`` 是區域起始位址 ``X`` 與結束位址 ``Y``，
單位為位元組。第九個欄位為 ``nr_accesses``，詳情請參閱
:ref:`區域取樣 <damon_design_region_based_sampling_zh_TW>`。
第十個欄位為 ``age``，詳情請參閱
:ref:`存取模式持續時間追蹤 <damon_design_age_tracking_zh_TW>`。

若事件為 ``damon:damos_before_apply``，輸出大致如下：::

    kdamond.0 47293 [000] 80801.060214: damon:damos_before_apply: ctx_idx=0 scheme_idx=0 target_idx=0 nr_regions=11 121932607488-135128711168: 0 136
    [...]

每行代表在追蹤當下，方案即將套用動作的一個監測區域。前五個欄位與一般
輸出相同。相較於 ``damon_aggregated``，另有 ``ctx_idx=X``，表示方案所屬
情境在其 kdamond 情境清單中的索引；以及 ``scheme_idx=X``，表示方案在
該情境方案清單中的索引。
