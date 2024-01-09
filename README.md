# Linux Project 2


:::info
### 作業需求 
*  新增 system call 設定 process 的 priority
*  顯示 priority 如何影響程式運作時間
*  根據結果分析 process 與 priority 之間的關係

:::
:::success
### 加分題
* Find the **right data** and the **right location** to change the data so that **the execution time** of the tested process can be **adjusted regularly** 
:::
## 環境
* Ubuntu版本:ubuntu 22.04
* kernel板本:5.15.74
* 虛擬機軟體:VMware Workstation 17 pro
---
## 壹、kernel 原始碼新增:
* ### struct task_struct:
    + 路徑:  
    ```
    include/linux/sched.h
    ```
    + 在 **struct task_struct** 中新增 **my_fixed_priority** 欄位與 **cswitch_number**
        + 用來確認context switch發生的次數
    > 
    ```csharp=
    struct task_struct {
        ......
    #endif

        int my_fixed_priority;//加在1491行
        int cswitch_number;//加在1492行
        /*
         * New fields for task_struct should be added above here, so that
         * they are included in the randomized portion of task_struct.
         */
        randomized_struct_fields_end

        /* CPU-specific state of this task: */
        struct thread_struct		thread;

        /*
         * WARNING: on x86, 'thread_struct' contains a variable-sized
         * structure.  It *MUST* be at the end of 'task_struct'.
         *
         * Do not put anything below here!
         */
        ......
    };
    ```
* ### copy_process:
    + 路徑:
    ```
    kernal/fork.c
    ```
    
    + 在 **copy_process** 中**初始化** **int** **my_fixed_priority** 與 **int** **cswitch_number** 為 0
    >
    ```csharp=
    static __latent_entropy struct task_struct *copy_process(
					struct pid *pid,
					int trace,
					int node,
					struct kernel_clone_args *args)
    {
        ......
        if (pidfile)
		fd_install(pidfd, pidfile);

        proc_fork_connector(p);
        sched_post_fork(p);
        cgroup_post_fork(p, args);
        perf_event_fork(p);

        trace_task_newtask(p, clone_flags);
        uprobe_copy_process(p, clone_flags);

        copy_oom_score_adj(clone_flags, p);
        p->my_fixed_priority=0;//2428行
        p->cswitch_number=0;//2429行
        return p;
        ......
    }
    ```
* ###  __sched notrace __schedule:
  + 路徑:
    ```
    kernel/sched/core.c
    ```
  + 當呼叫 **sched notrace __schedule** 並發生 context switch 時，增加 **prev** 與 **next** 的 task_struct 產生 context switch 時的切換次數
    ```csharp
    static void __sched notrace __schedule(unsigned int     sched_mode){
	    struct task_struct *prev, *next;
        unsigned long *switch_count;
        unsigned long prev_state;
        struct rq_flags rf;
        struct rq *rq;
        int cpu;
        ......
        if (likely(prev != next)) {
		rq->nr_switches++;
		/*
		 * RCU users of rcu_dereference(rq->curr) may not see
		 * changes to task_struct made by pick_next_task().
		 */
		RCU_INIT_POINTER(rq->curr, next);
	    ......
		++*switch_count;
		prev->cswitch_number++;//被切換的task_struct
		next->cswitch_number++;//即將切往的目標task_struct
		migrate_disable_switch(rq, prev);
		psi_sched_switch(prev, next, !task_on_rq_queued(prev));

		trace_sched_switch(sched_mode & SM_MASK_PREEMPT, prev, next);

		/* Also unlocks the rq: */
		rq = context_switch(rq, prev, next, &rf);
	    } else {
		rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);

		rq_unpin_lock(rq, &rf);
		__balance_callbacks(rq);
		raw_spin_rq_unlock_irq(rq);
	    }
        ......
    }
    ```
## 貳、新增system call:

+ task_struct 中的成員變數 **prio 越小，行程的優先權越高**，prio 值的範圍在0~139
+ task_struct 中標示 linux 進程優先權的幾個重要變數
    + **prio 調度優先權**
    >+ 行程的動態優先級
    >+ 正常情況下，prio 會等於 normal_prio
    + **static_prio 靜態優先權**
    >+ 在行程啟動時進行分配
    >+ 在 **被CFS 調度**的行程才有意義
    >+ 使用者可以透過修改 nice 值來修改靜態優先權
    >+ kernal 不會儲存 nice 值，只會透過 NICE_TO_PRIO 巨集將 nice 值轉換為 static_prio
    + **normal_prio**
    >+ 根據調度器類型計算出來的
    >>+ 實時進程 : 99 - priority
    >>+ 非實時進程 : static_prio
    + **rt_prio**
    >+ 即時優先級
    >+ 在**被即時調度器**調度的進程中才有意義

* ### mysys:
```csharp=
#include <linux/kernel.h>
#include <linux/syscalls.h>
#include <linux/sched.h>
#include <uapi/linux/sched/types.h>

SYSCALL_DEFINE1 (mypri,int , new_pri){
    printk("syscall start\n");
    if(new_pri<101 ||new_pri >139){
        printk("range wrong\n");
        return 0;
    }
    printk("original static_prio=%d\n",current->static_prio);
    printk("original prio=%d\n",current->prio);
    printk("original rt_prio=%d\n",current->rt_priority);
    printk("original normal_prio=%d\n",current->normal_prio);
    printk("original policy=%d\n",current->policy);
    printk("original cswitch_num=%d\n",current->cswitch_number);
    //printk("Current process vruntime: %llu\n", current->se.vruntime);
    if(current->my_fixed_priority != 0){
        current->my_fixed_priority = new_pri;
        current->static_prio = current->my_fixed_priority;
        current->policy = 0;/*SCHED_NORMAL*/
        current->rt_priority = 0;
        current->normal_prio = current->static_prio;
        current->prio = current->my_fixed_priority;
        printk("new static_prio=%d\n",current->static_prio);
        printk("new prio=%d\n",current->prio);
        printk("new rt_prio=%d\n",current->rt_priority);
        printk("new normal_prio=%d\n",current->normal_prio);
        printk("new policy=%d\n",current->policy);
        printk("new cswitch_num=%d\n",current->cswitch_number);
        //printk("Current process vruntime: %llu\n", current->se.vruntime);
        const struct sched_param param = {
            .sched_priority = 0
        };
        sched_setscheduler(current,SCHED_NORMAL,&param);
        return 1;
    }
    else{
        printk("pri = 0,no need change\n");
        return 1;
    }
}
```

```csharp=
current->my_fixed_priority = current->static_prio;
```
+ 這行程式碼將當前 process 的 static_prio 賦值給一個自定義的成員變量 my_fixed_priority。
+ 這裡 current 指向當前正在執行的 process 的任務結構（task struct）。
```csharp=
current->my_fixed_priority = new_pri;
```
+ 這行程式碼將新的優先級（由系統呼叫傳入的 new_pri）賦值給 my_fixed_priority。
```csharp=
current->static_prio = current->my_fixed_priority;
```
+ 更新當前的靜態優先級 (static_prio) 為新設定的優先級。
```csharp=
current->policy = 0;/*SCHED_NORMAL*/
```
+ 將當前的調度策略設定為普通調度（SCHED_NORMAL），在 Linux 中，這對應於數值 0。
```csharp=
current->rt_priority = 0;
```
+ 將實時（real-time）優先級設定為 0。在非實時調度策略下，這個值通常不被使用
```csharp=
current->normal_prio = current->static_prio;
```
+ 更新 process 的普通優先級（normal_prio）為新的靜態優先級。這通常用於普通調度策略
```csharp=
current->prio = current->my_fixed_priority;
```
+ 將當前動態優先級 (prio) 更新為新設定的優先級。


## 參、測試用程式:
```csharp=
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <time.h>
#include <sys/time.h>
#include <pthread.h>
#include <signal.h>

#define TOTAL_ITERATION_NUM  100000000
#define NUM_OF_PRIORITIES_TESTED  40
 
int main(void){
    int index=0;
    int priority,i;
    struct timeval start[NUM_OF_PRIORITIES_TESTED], end[NUM_OF_PRIORITIES_TESTED];
    gettimeofday(&start[index], NULL);           //begin
    for(i=1;i<=TOTAL_ITERATION_NUM;i++){
        rand();
    }
    gettimeofday(&end[index], NULL);             //end 

    /*================================================================================*/
    pid_t pid;
    int status;
    pid = fork();
    if(pid < 0){
        printf("fork error");
    }
    else if(pid==0){
        int i=0;
        printf("This is child process with pid %d\n", getpid());
        while(1){
            i++;
        }
    }
    else{
        for(index=1, priority=101;priority<=139;++priority,++index)
        {
            if(!syscall(449,priority)){
                printf("Cannot set priority %d\n" ,priority);
            }
            gettimeofday(&start[index], NULL);           //begin
            for(i=1;i<=TOTAL_ITERATION_NUM;i++){
                rand();
            }
            gettimeofday(&end[index], NULL);             //end  
            sleep(1);
            printf("Priority%d:finish\n",priority);                     
        }

        /*================================================================================*/
        kill(pid, SIGKILL);
        wait(&status);
        printf("The process spent %ld uses to execute when priority is not adjusted.\n", 
            ((end[0].tv_sec * 1000000 + end[0].tv_usec) - (start[0].tv_sec * 1000000 + start[0].tv_usec)));

        for(i=1;i<=NUM_OF_PRIORITIES_TESTED-1;i++)
            printf("The process spent %ld uses to execute when priority is %d.\n", 
            ((end[i].tv_sec * 1000000 + end[i].tv_usec) - (start[i].tv_sec * 1000000 + start[i].tv_usec)), i+100);
    }
    return 0;
}
```
## 肆、結果分析:
+ 測試用程式執行後，刻意創造另一個無窮迴圈 process 使主程式持續產生 **context switch**。
+  主程式針對不同 priority 進入亂數迴圈。
+  以下為執行過程中根據 system call 產生的部分相關資訊
    +  確認程式執行過程中有正確更動 priority 以及發生 context_switch

    !![螢幕擷取畫面 2024-01-07 161900](https://hackmd.io/_uploads/B1C5A0DOa.png)
+ 以下為測試用程式的輸出結果(時間單位為微秒):

    ![3](https://hackmd.io/_uploads/HyGaOyvdT.png)
    >根據上圖顯示，priority 為相對較大的值(130)之後開始有較明顯的延遲。在這之前的延遲時間並沒有因為 priority 的更動產生較有規律性的變化。
    
 + 以下為執行過程中，不同 prio 所對應到的 vruntime      
     >透過表格的觀察，可以明確地發現 vruntime 的值是逐漸變大的。

| Prio |        vruntime | Prio | vruntime        | Prio | vruntime        | Prio | vruntime        |
| ---- | ---------------:| ---- | --------------- | ---- | --------------- | ---- | --------------- |
| 120  |     64219087550 | 110  | **18**063168525 | 120  | **34**589878784 | 130  | **51**187899281 |
| 101  |  **3**207049035 | 111  | **19**789593749 | 121  | **36**260995736 | 131  | **52**818237161 |
| 102  |  **4**891145820 | 112  | **21**416370224 | 122  | **37**926686708 | 132  | **54**457831827 |
| 103  |  **6**531129759 | 113  | **23**043941327 | 123  | **39**642207533 | 133  | **56**094920290 |
| 104  |  **8**205113769 | 114  | **24**667667354 | 124  | **41**280099020 | 134  | **57**724067727 |
| 105  |  **9**875868382 | 115  | **26**312534028 | 125  | **42**957483914 | 135  | **59**358695647 |
| 106  | **11**499668808 | 116  | **27**934085310 | 126  | **44**591718148 | 136  | **60**970982437 |
| 107  | **13**125189352 | 117  | **29**652953745 | 127  | **46**228721935 | 137  | **62**617272338 |
| 108  | **14**776146852 | 118  | **31**293012656 | 128  | **47**861013214 | 138  | **64**263931930 |
| 109  | **16**424367080 | 119  | **32**967167568 | 129  | **49**489747193 | 139  | **65**891925358 |
+ **說明** ([參考資料](https://blog.csdn.net/weixin_45030965/article/details/128566265))
    + **權重介紹**
        **1. nice值**
        + 在 kernal 中，進程優先級由0至139的數字表示，**數值越小，優先級越高**。0至99用於實時進程（優先級0至99），而100至139用於普通進程（優先級100至139）。在使用者空間，普通進程的優先級以-20至19的nice值表示，對應核心的100至139優先級，其中 **-20為最高**，**19為最低**。普通進程的預設nice值為0。靜態優先級（static_prio）範圍為100至139，創建時設定，通常不變，但可透過特定命令或系統調用更改。
        
        **2. 權重簡介**
        + 在Linux中，CFS（完全公平調度器）之前使用的 O(1) 調度器會根據進程的nice 值分配固定的時間片。而 CFS 調度器不再使用時間片，而是將nice值視為進程獲得處理器運行時間的權重。nice 值越小（優先級越高），獲得的處理器運行權重越大。      
        + 在CFS中，每個nice值對應一個特定的權重值，**nice值越小，對應的權重數值越大，進而獲得更高的處理器運行時間比例**。
        + 例如 nice 值為-20的進程擁有最高的權重**88761**，而 nice 值為15的進程權重最低為**15**，prio_to_weight 數組中，相鄰 nice 值的權重倍數約為1.25，意味著每增加1的 nice 值，進程獲得的 CPU 時間約減少10%。
        + 例如 nice 值為0的進程權重為1024，nice 值為1的權重為820，nice值為0的進程比nice 值為1的進程多獲得約10%的 CPU 時間。 
        ```csharp=
        /*
         * Nice levels are multiplicative, with a gentle 10% change for every
         * nice level changed. I.e. when a CPU-bound task goes from nice 0 to
         * nice 1, it will get ~10% less CPU time than another CPU-bound task
         * that remained on nice 0.
         *
         * The "10% effect" is relative and cumulative: from _any_ nice level,
         * if you go up 1 level, it's -10% CPU usage, if you go down 1 level
         * it's +10% CPU usage. (to achieve that we use a multiplier of 1.25.
         * If a task goes up by ~10% and another task goes down by ~10% then
         * the relative distance between them is ~25%.)
         */
        const int sched_prio_to_weight[40] = {
         /* -20 */     88761,     71755,     56483,     46273,     36291,
         /* -15 */     29154,     23254,     18705,     14949,     11916,
         /* -10 */      9548,      7620,      6100,      4904,      3906,
         /*  -5 */      3121,      2501,      1991,      1586,      1277,
         /*   0 */      1024,       820,       655,       526,       423,
         /*   5 */       335,       272,       215,       172,       137,
         /*  10 */       110,        87,        70,        56,        45,
         /*  15 */        36,        29,        23,        18,        15,
        };
        ```        
        
    + **vruntime 的介紹與計算**
    
        **1.** **介紹**
        + Linux 的 CFS （完全公平調度器）為每個進程（sched_entity）維護一個虛擬運行時間（vruntime），用於選擇下一個運行的進程。CFS 通過紅黑樹對進程按vruntime進行排序，每次調度都選擇擁有最小 vruntime 的進程。這個 vruntime 是基於進程的實際運行時間，結合其 nice 值（影響權重）進行加權計算得出。
        + CFS 中還包括了一些關鍵的組件和變數：
            + run_node：用於將調度實體組織在紅黑樹中的節點。
            + exec_start：記錄進程最近一次開始運行的時間。
            + sum_exec_runtime：調度實體自創建以來的總實際運行時間。
            + prev_sum_exec_runtime：至上次離開 CPU 的累計運行時間。
            + 每個CPU都有自己的運行隊列，定義在 per-CPU 陣列 runqueues 中。
            + CFS 運行隊列（cfs_rq）中有一個本地時鐘（clock）和一個不包含中斷處理時間的時鐘（clock_task）。
            + min_vruntime 是運行隊列中所有進程的最小虛擬運行時間。
            + tasks_timeline 是紅黑樹的根節點，而 rb_leftmost 是最左邊的葉子節點。     
            + curr 則是當前正在運行的調度實體。    
![螢幕擷取畫面 2024-01-07 214449](https://hackmd.io/_uploads/rkluQHuda.png)

        **2.** **update_curr()**       
        + update_curr() 是 CFS 調度器的核心函數，用來**更新調度實體的 vruntime 值**。
        + Linux CFS 調度器的 update_curr 函數主要執行以下步驟來更新進程的虛擬運行時間（vruntime）：
            + 獲取不包含中斷處理時間的當前運行隊列時脈（clock_task）。
            + 計算進程自上次 update_curr 調用以來的實際運行時間增量（delta_exec），即當前時刻與進程上次調度開始時刻（exec_start）之差。
            + 更新進程的累計實際運行時間（sum_exec_runtime）。
            + 根據進程的權重對 delta_exec 進行加權，得到加權的虛擬運行時間增量（delta_exec_weighted）。
            + 使用加權後的時間增量更新進程的虛擬運行時間（vruntime）。
            + 更新CFS運行隊列的最小虛擬運行時間（min_vruntime），這個值反映隊列中所有進程的最小 vruntime，且是單調遞增的。
            + 將進程的最近一次調度開始時間（exec_start）更新為當前時刻（now），為下次運行時間差值計算做準備。
        
        
        **3.** **vruntime 計算**
        
        + CFS 調度器在 Linux 中使用虛擬運行時間（vruntime）來確保進程公平地獲得 CPU 時間， vruntime 的增長取決於進程的實際運行時間（Δdelta_exec）和其 nice 值決定的權重。對於 nice 值為0的進程，其權重設定為NICE_0_LOAD（1024），且Δvruntime等於 Δdelta_exec。 nice 值較小（優先級較高）的進程擁有較高的權重，導致其 vruntime 增長較慢；相反地， nice 值較大（優先級較低）的進程擁有較低的權重， vruntime 增長較快。        
        + 這種設計確保了高優先級的進程在實際運行相同時間後，累計的虛擬運行時間較短，因此在紅黑樹中移動速度較慢，獲得調度的機會較多。CFS 始終選擇虛擬時鐘最慢的進程進行調度，從而確保即使是低優先級的進程也有機會被調度。隨著時間推移，所有進程的 vruntime 趨於平衡，實現公平調度。

    ![螢幕擷取畫面 2024-01-07 225802](https://hackmd.io/_uploads/HyTZnE_Op.png)


## 伍、問題討論:

**為何 static prio 會影響排程?**
+ 在系統使用sched_setscheduler()設置priority時，會依照以下順序呼叫函式:
    + sched_setscheduler()
        > __sched_setscheduler()
        > __setscheduler()
    
+ 其中在 **__setscheduler** 內的設置如下:
```csharp
static void
__setscheduler(struct rq *rq, struct task_struct *p, int policy, int prio)
{
	p->policy = policy;//根據輸入的策略而定
	p->rt_priority = prio;//僅對實時進程作用
	p->normal_prio = normal_prio(p);
	/* we are holding p->pi_lock already */
	p->prio = rt_mutex_getprio(p);
	if (rt_prio(p->prio))
		p->sched_class = &rt_sched_class;
	else
		p->sched_class = &fair_sched_class;
	set_load_weight(p);
}

```
+ 由於我們的程式根據策略選擇(**SCHED_NORMAL**)屬於普通進程，因此實際遵循的 priority 是**normal_prio**，再往下觀察至 normal_prio():
```csharp
static inline int normal_prio(struct task_struct *p)
{
	int prio;
 
	if (task_has_rt_policy(p))
		prio = MAX_RT_PRIO-1 - p->rt_priority;
	else
		prio = __normal_prio(p);
	return prio;
}
```
+ 經過判斷是否為實時進程後，呼叫 __normal_prio():
```cs
/*
 * __normal_prio - return the priority that is based on the static prio
 */
static inline int __normal_prio(struct task_struct *p)
{
	return p->static_prio;
}
```
+ 可發現最終回傳的值是直接取得當前 task_struct 的 **static_prio**


## 陸、加分題:
+ 嘗試透過修改 sched_prio_to_weight 來影響 vruntime
    + Path
    ```
    /kernal/sched/core.c
    ```
    ![螢幕擷取畫面 2024-01-07 222557](https://hackmd.io/_uploads/ByVpNNuOa.png)
    
+ 更新權重後，prio 與 vruntime 的對照表
    | Prio |    vruntime | Prio | vruntime    | Prio | vruntime    | Prio |    vruntime |
    | ---- | -----------:| ---- | ----------- | ---- | ----------- | ---- | -----------:|
    | **** |        **** | 110  | 16203410461 | 120  | 17007383541 | 130  | 34404178169 |
    | 101  |  1663599059 | 111  | 17809885462 | 121  | 18783536039 | 131  | 36105659861 |
    | 102  |  3256915385 | 112  | 3245673010  | 122  | 20477242306 | 132  | 37897242558 |
    | 103  |  4835494948 | 113  | 4882546701  | 123  | 22088587395 | 133  | 39558540075 |
    | 104  |  6447565814 | 114  | 6556064678  | 124  | 23912285385 | 134  | 41212508661 |
    | 105  |  8062589433 | 115  | 8343714173  | 125  | 25720979791 | 135  | 42860938404 |
    | 106  |  9666157236 | 116  | 10057318529 | 126  | 27415877468 | 136  | 44472672410 |
    | 107  | 11271143176 | 117  | 11810318001 | 127  | 29127390714 | 137  | 46083659967 |
    | 108  | 12948922608 | 118  | 13534385029 | 128  | 30869529686 | 138  | 47720769552 |
    | 109  | 14584985790 | 119  | 15257631115 | 129  | 32633365132 | 139  | 49329515605 |


+ 猜測可能與 **update_curr(struct** **cfs_rq** ***cfs_rq)** 有關

    + Path
    ```
    /kernal/sched/fair.c
    ```
    + 功能介紹
    ```csharp=
    static void update_curr(struct cfs_rq *cfs_rq)
    {
        struct sched_entity *curr = cfs_rq->curr;
        u64 now = rq_clock_task(rq_of(cfs_rq));
        u64 delta_exec;

        if (unlikely(!curr))
            return;

        delta_exec = now - curr->exec_start;
        if (unlikely((s64)delta_exec <= 0))
            return;

        curr->exec_start = now;

        schedstat_set(curr->statistics.exec_max,
                  max(delta_exec, curr->statistics.exec_max));

        curr->sum_exec_runtime += delta_exec;
        schedstat_add(cfs_rq->exec_clock, delta_exec);

        curr->vruntime += calc_delta_fair(delta_exec, curr);
        update_min_vruntime(cfs_rq);

        if (entity_is_task(curr)) {
            struct task_struct *curtask = task_of(curr);

            trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
            cgroup_account_cputime(curtask, delta_exec);
            account_group_exec_runtime(curtask, delta_exec);
        }

        account_cfs_rq_runtime(cfs_rq, delta_exec);
    }
    ```

## 柒、參考資料:
+ [linux kernel scheduler -- 進程優先權](https://blog.csdn.net/fervor_heart/article/details/46927469)
+ [深入理解Linux核心-進程-進程調度](https://blog.csdn.net/x13262608581/article/details/131908901)
+ [玩轉Linux核心行程調度，這篇就夠(所有的知識點)](https://blog.csdn.net/m0_50662680/article/details/129221045?ops_request_misc=&request_id=&biz_id=102&utm_term=Linux%20kernel%20context%20switch&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-4-129221045.nonecase&spm=1018.2226.3001.4187)
+ [linux kernel cfs scheduler](https://blog.csdn.net/weixin_45030965/article/details/128566265)
+ [Linux CFS調度器vruntime 的計算](https://blog.csdn.net/weixin_45030965/article/details/128566265) 
+ [task_struct結構體的優先權參數詳解：prio、static_prio、normal_prio、rt_priority](https://blog.csdn.net/weixin_42314225/article/details/121708134)
+ [Linux CFS and task group](https://mechpen.github.io/posts/2020-04-27-cfs-group/index.html)
+ [CFS 調度器資料結構篇](https://blog.csdn.net/longwang155069/article/details/104551212)
+ [Linux CFS 調度器：原理、設計與核心實作（2023）](https://arthurchiao.art/blog/linux-cfs-design-and-implementation-zh/)
+ [Linux 核心設計: Scheduler(2): 概述 CFS Scheduler](https://hackmd.io/@RinHizakura/B18MhT00t)
+ [CFS對vruntime的迭代更新](https://blog.csdn.net/weixin_37948055/article/details/108062362)
+ [Linux CFS調度器之喚醒搶佔--Linux進程的管理與調度(三十）](https://blog.csdn.net/weixin_39094034/article/details/113094836)
+ [LKD讀書筆記：Linux的CFS調度器之Virtual Runtime](https://marvinsblog.net/post/2016-02-14-lkd-notes-cfv-virtual-runtime/)
+ [CFS調度器-源碼解析](http://www.wowotech.net/process_management/448.html)
+ [Linux 核心設計: Scheduler(3): 深入剖析 CFS Scheduler](https://hackmd.io/@RinHizakura/BJ9m_qs-5)

