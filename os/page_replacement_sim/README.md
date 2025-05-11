# 页面替换算法模拟实验 - C语言实现框架

## 项目结构：

```
page_replacement_sim/
├── include/                # 头文件目录
│   ├── simulator.h
│   ├── algorithms.h
│   ├── workload.h
│   └── utils.h
├── src/                  # 源文件目录
│   ├── main.c
│   ├── simulator.c
│   ├── fifo.c
│   ├── random_alg.c
│   ├── lru.c
│   ├── clock_alg.c
│   ├── lfu.c
│   ├── wtinylfu.c
│   ├── workload.c
│   └── utils.c
├── data/                 # 存放生成的页面访问序列
├── results/              # 存放实验结果（CSV文件）
└── Makefile              # 编译脚本
```

---

## I. C语言实现思路与关键代码片段

### 1. `include/utils.h` 和 `src/utils.c` (辅助工具)

```c
// include/utils.h
#ifndef UTILS_H
#define UTILS_H

#include <stdio.h>
#include <stdlib.h>
#include <time.h> // 用于随机种子

// 生成指定范围[min, max]内的随机整数
int generate_random_int(int min, int max);

// 生成0到1之间的随机浮点数
float generate_random_float();

// 简单的错误处理函数
void die(const char *message);

#endif

// src/utils.c
#include "utils.h"

int generate_random_int(int min, int max) {
    return min + rand() % (max - min + 1);
}

float generate_random_float() {
    return (float)rand() / (float)RAND_MAX;
}

void die(const char *message) {
    perror(message);
    exit(EXIT_FAILURE);
}
```

### 2. `include/workload.h` 和 `src/workload.c` (工作负载生成)

```c
// include/workload.h
#ifndef WORKLOAD_H
#define WORKLOAD_H

#include <stdio.h>

#define MAX_ACCESSES 10000  // 访问序列中的最大页面访问次数
#define MAX_UNIQUE_PAGES 100 // 最大唯一页面数

typedef struct {
    int accesses[MAX_ACCESSES];  // 页面访问序列
    int num_accesses;            // 实际访问次数
    int unique_pages;            // 唯一页面数
} Workload;

// 生成无局部性的工作负载
Workload generate_no_locality_workload(int total_accesses, int unique_pages);

// 生成80-20局部性工作负载
Workload generate_80_20_workload(int total_accesses, int unique_pages);

// 生成顺序循环工作负载
Workload generate_sequential_loop_workload(int total_accesses, int unique_pages, int loop_length);

// （可选）将工作负载保存到文件
void save_workload_to_file(const Workload *workload, const char *filename);

// （可选）从文件加载工作负载
Workload load_workload_from_file(const char *filename);

#endif

// src/workload.c
#include "workload.h"
#include "utils.h" // 用于随机数生成

Workload generate_no_locality_workload(int total_accesses, int unique_pages) {
    Workload wl;
    wl.num_accesses = total_accesses > MAX_ACCESSES ? MAX_ACCESSES : total_accesses;
    wl.unique_pages = unique_pages > MAX_UNIQUE_PAGES ? MAX_UNIQUE_PAGES : unique_pages;

    for (int i = 0; i < wl.num_accesses; ++i) {
        wl.accesses[i] = generate_random_int(0, wl.unique_pages - 1);
    }
    return wl;
}

Workload generate_80_20_workload(int total_accesses, int unique_pages) {
    Workload wl;
    wl.num_accesses = total_accesses > MAX_ACCESSES ? MAX_ACCESSES : total_accesses;
    wl.unique_pages = unique_pages > MAX_UNIQUE_PAGES ? MAX_UNIQUE_PAGES : unique_pages;

    // 20%的页面被访问80%的时间
    int hot_pages_count = (int)(0.20 * wl.unique_pages);
    if (hot_pages_count == 0 && wl.unique_pages > 0) hot_pages_count = 1; // 确保至少有一个热页面

    for (int i = 0; i < wl.num_accesses; ++i) {
        if (generate_random_float() < 0.80 && hot_pages_count > 0) {
            // 80%的概率访问热页面
            wl.accesses[i] = generate_random_int(0, hot_pages_count - 1);
        } else {
            // 20%的概率访问冷页面
            if (wl.unique_pages - hot_pages_count > 0) {
                 wl.accesses[i] = generate_random_int(hot_pages_count, wl.unique_pages - 1);
            } else if (hot_pages_count > 0) { // 只有热页面存在
                 wl.accesses[i] = generate_random_int(0, hot_pages_count - 1);
            } else { // 没有定义页面（适当输入不应该发生）
                wl.accesses[i] = 0;
            }
        }
    }
    return wl;
}

Workload generate_sequential_loop_workload(int total_accesses, int unique_pages, int loop_length) {
    Workload wl;
    wl.num_accesses = total_accesses > MAX_ACCESSES ? MAX_ACCESSES : total_accesses;
    // 确保loop_length在unique_pages范围内
    wl.unique_pages = (loop_length > unique_pages) ? loop_length : unique_pages;
    wl.unique_pages = wl.unique_pages > MAX_UNIQUE_PAGES ? MAX_UNIQUE_PAGES : wl.unique_pages;

    int current_loop_length = (loop_length > wl.unique_pages) ? wl.unique_pages : loop_length;
    if (current_loop_length <= 0) current_loop_length = 1; // 避免除零

    for (int i = 0; i < wl.num_accesses; ++i) {
        wl.accesses[i] = i % current_loop_length;
    }
    return wl;
}
// 根据需要实现保存/加载函数
```

### 3. `include/algorithms.h` (通用算法接口和数据结构)

```c
// include/algorithms.h
#ifndef ALGORITHMS_H
#define ALGORITHMS_H

#include <stdbool.h>

#define MAX_CACHE_SIZE 100 // 最大物理页帧数

// 通用页面结构（可扩展）
typedef struct {
    int page_number;        // 页面号
    // 算法特定的元数据（指针或直接值）
    int frequency;          // 用于LFU、W-TinyLFU
    int last_used_time;     // 用于LRU（可以是计数器）
    bool use_bit;           // 用于Clock
    // ... W-TinyLFU分段等其他元数据
} PageFrame;

// 缓存结构
typedef struct {
    PageFrame frames[MAX_CACHE_SIZE];   // 页帧数组
    int capacity;                       // 最大帧数
    int current_size;                   // 当前占用的帧数

    // 算法特定状态（如指针、计数器）
    void* alg_data; // 指向算法特定数据结构的指针
} Cache;

// 算法操作的函数指针类型
typedef void (*init_algorithm_func)(Cache* cache, int capacity);
typedef int (*access_page_func)(Cache* cache, int page_num, int current_time_step); // 返回1表示缺页，0表示命中
typedef void (*destroy_algorithm_func)(Cache* cache);

// --- 算法特定结构（示例） ---
// 用于FIFO
typedef struct {
    int* queue;     // 存储页面号的FIFO顺序数组
    int head;       // 队头索引
    int tail;       // 队尾索引
    int count;      // 队列中的项目数
} FIFO_Data;

// 用于LRU
typedef struct {
    // 可以使用简单数组和移位，或更复杂的链表以提高效率
    // 为了简单起见，在C中使用数组，其中索引0是MRU，最后是LRU
    // 或者，在PageFrame中存储last_used_time并搜索
    int time_counter; // LRU时间戳的全局时间步
} LRU_Data;

// 用于Clock
typedef struct {
    int clock_hand; // 时钟指针
} Clock_Data;

// 用于LFU
typedef struct {
    // 频率存储在PageFrame.frames[i].frequency中
    // 平局处理可能需要额外信息（例如LRU平局处理）
} LFU_Data;

// 用于W-TinyLFU（简化版）
#define WTINYLFU_PROBATION_RATIO 0.20 // 20%用于试用分段
typedef struct {
    // 全局频率计数（如果页面号较小，可以是哈希映射或数组）
    int* global_page_frequencies; // 假设页面号从0到U_PAGES-1
    int max_unique_pages_for_freq;

    // SLRU结构：
    // 我们可以通过定义开始/结束索引或通过拥有单独的索引列表，
    // 在主cache->frames数组中管理试用和保护分段。
    // 为简单起见，让我们假设主cache->frames数组的分割。
    int probation_capacity;     // 试用分段容量
    int protected_capacity;     // 保护分段容量

    // 要在分段内管理LRU，我们需要跟踪顺序。
    // 选项1：PageFrame索引的实际链表。
    // 选项2：PageFrame上的时间戳和搜索（更简单的代码，效率较低）。
    // 让我们使用时间戳来简化C代码。
    int time_counter; // 全局时间步

    // 我们需要知道哪些帧属于试用，哪些属于保护。
    // 如果我们重新排列帧，这可以是隐式的，或者用标志/单独列表显式。
    // 为简单起见，让我们假设基于索引的隐式分割，
    // 尽管这使得在分段之间移动成本高昂（memmove）。
    // 更好的方法是为每个分段使用占用帧索引的链表。

} WTinyLFU_Data;

// --- 算法特定函数的声明 ---
// 这些将在各自的.c文件中实现

// FIFO
void fifo_init(Cache* cache, int capacity);
int fifo_access(Cache* cache, int page_num, int current_time_step);
void fifo_destroy(Cache* cache);

// Random
void random_alg_init(Cache* cache, int capacity);
int random_alg_access(Cache* cache, int page_num, int current_time_step);
void random_alg_destroy(Cache* cache); // 可能什么也不做

// LRU
void lru_init(Cache* cache, int capacity);
int lru_access(Cache* cache, int page_num, int current_time_step);
void lru_destroy(Cache* cache);

// Clock
void clock_alg_init(Cache* cache, int capacity);
int clock_alg_access(Cache* cache, int page_num, int current_time_step);
void clock_alg_destroy(Cache* cache);

// LFU
void lfu_init(Cache* cache, int capacity);
int lfu_access(Cache* cache, int page_num, int current_time_step);
void lfu_destroy(Cache* cache);

// W-TinyLFU
void wtinylfu_init(Cache* cache, int capacity, int max_unique_pages); // 需要max_unique_pages用于全局频率数组
int wtinylfu_access(Cache* cache, int page_num, int current_time_step);
void wtinylfu_destroy(Cache* cache);

#endif
```

### 4. `src/fifo.c`, `src/random_alg.c`, 等 (算法实现)

#### `fifo.c` (示例)：

```c
#include "algorithms.h"
#include "utils.h" // 用于die()
#include <string.h> // 用于memset

void fifo_init(Cache* cache, int capacity) {
    cache->capacity = capacity > MAX_CACHE_SIZE ? MAX_CACHE_SIZE : capacity;
    cache->current_size = 0;
    for (int i = 0; i < cache->capacity; ++i) {
        cache->frames[i].page_number = -1; // 标记为空
    }

    FIFO_Data* data = (FIFO_Data*)malloc(sizeof(FIFO_Data));
    if (!data) die("分配FIFO_Data失败");
    data->queue = (int*)malloc(sizeof(int) * cache->capacity);
    if (!data->queue) die("分配FIFO队列失败");
    data->head = 0;
    data->tail = 0;
    data->count = 0;
    cache->alg_data = data;
}

int fifo_access(Cache* cache, int page_num, int current_time_step /*未使用*/) {
    FIFO_Data* data = (FIFO_Data*)cache->alg_data;

    // 检查是否命中
    for (int i = 0; i < cache->current_size; ++i) {
        if (cache->frames[i].page_number == page_num) {
            return 0; // 命中
        }
    }

    // 缺页处理
    if (cache->current_size < cache->capacity) { // 缓存有空间
        // 找到第一个空帧
        int empty_frame_idx = -1;
        for(int i=0; i<cache->capacity; ++i) {
            if(cache->frames[i].page_number == -1) {
                empty_frame_idx = i;
                break;
            }
        }
        cache->frames[empty_frame_idx].page_number = page_num;
        cache->current_size++;

        // 添加到FIFO队列
        data->queue[data->tail] = page_num; // 存储page_num以便稍后识别受害者
                                            // 或者更好，存储frame_index
        data->tail = (data->tail + 1) % cache->capacity;
        data->count++;
    } else { // 缓存已满，需要替换
        int victim_page_num = data->queue[data->head];
        data->head = (data->head + 1) % cache->capacity;
        // 如果tail立即覆盖，这里不需要递减data->count

        // 在cache->frames中找到victim_page_num并替换它
        int victim_frame_idx = -1;
        for (int i = 0; i < cache->capacity; ++i) {
            if (cache->frames[i].page_number == victim_page_num) {
                victim_frame_idx = i;
                break;
            }
        }
        if (victim_frame_idx != -1) {
            cache->frames[victim_frame_idx].page_number = page_num;
        } else {
            // 如果队列和缓存一致，这不应该发生
            fprintf(stderr, "错误：FIFO队列中的受害页面未在缓存帧中找到。\n");
        }

        // 将新页面添加到FIFO队列的尾部
        data->queue[data->tail] = page_num;
        data->tail = (data->tail + 1) % cache->capacity;
        // data->count保持为cache->capacity
    }
    return 1; // 发生缺页
}

void fifo_destroy(Cache* cache) {
    if (cache->alg_data) {
        FIFO_Data* data = (FIFO_Data*)cache->alg_data;
        free(data->queue);
        free(data);
        cache->alg_data = NULL;
    }
}
```

#### `lru.c` (使用`PageFrame`中的`last_used_time`简化版)：

```c
// src/lru.c
#include "algorithms.h"
#include "utils.h"
#include <limits.h> // 用于INT_MAX

void lru_init(Cache* cache, int capacity) {
    cache->capacity = capacity > MAX_CACHE_SIZE ? MAX_CACHE_SIZE : capacity;
    cache->current_size = 0;
    for (int i = 0; i < cache->capacity; ++i) {
        cache->frames[i].page_number = -1;
        cache->frames[i].last_used_time = 0;
    }
    LRU_Data* data = (LRU_Data*)malloc(sizeof(LRU_Data));
    if (!data) die("分配LRU_Data失败");
    data->time_counter = 0;
    cache->alg_data = data;
}

int lru_access(Cache* cache, int page_num, int current_time_step) {
    LRU_Data* data = (LRU_Data*)cache->alg_data;
    data->time_counter = current_time_step; // 或者只是在内部递增data->time_counter

    // 检查是否命中
    for (int i = 0; i < cache->current_size; ++i) { // 仅迭代占用的帧
        // 如果不迭代所有cache->capacity，需要找到实际的帧索引
        int frame_idx = -1;
        for(int k=0; k < cache->capacity; ++k) {
            if(cache->frames[k].page_number == page_num) {
                frame_idx = k;
                break;
            }
        }
        if (frame_idx != -1) { // 命中
            cache->frames[frame_idx].last_used_time = data->time_counter;
            return 0; // 命中
        }
    }

    // 缺页处理
    if (cache->current_size < cache->capacity) {
        // 找到第一个空帧
        int empty_frame_idx = -1;
        for(int i=0; i<cache->capacity; ++i) {
            if(cache->frames[i].page_number == -1) {
                empty_frame_idx = i;
                break;
            }
        }
        cache->frames[empty_frame_idx].page_number = page_num;
        cache->frames[empty_frame_idx].last_used_time = data->time_counter;
        cache->current_size++;
    } else { // 缓存已满，找到LRU页面
        int lru_frame_idx = -1;
        int min_time = INT_MAX;
        for (int i = 0; i < cache->capacity; ++i) {
            if (cache->frames[i].last_used_time < min_time) {
                min_time = cache->frames[i].last_used_time;
                lru_frame_idx = i;
            }
        }
        cache->frames[lru_frame_idx].page_number = page_num;
        cache->frames[lru_frame_idx].last_used_time = data->time_counter;
    }
    return 1; // 缺页
}

void lru_destroy(Cache* cache) {
    if (cache->alg_data) {
        free(cache->alg_data);
        cache->alg_data = NULL;
    }
}
```

#### `random_alg.c` (随机替换算法)：

```c
#include "algorithms.h"
#include "utils.h"

void random_alg_init(Cache* cache, int capacity) {
    cache->capacity = capacity > MAX_CACHE_SIZE ? MAX_CACHE_SIZE : capacity;
    cache->current_size = 0;
    for (int i = 0; i < cache->capacity; ++i) {
        cache->frames[i].page_number = -1;
    }
    cache->alg_data = NULL; // 随机算法不需要额外的数据结构
}

int random_alg_access(Cache* cache, int page_num, int current_time_step) {
    // 检查是否命中
    for (int i = 0; i < cache->current_size; ++i) {
        if (cache->frames[i].page_number == page_num) {
            return 0; // 命中
        }
    }

    // 缺页处理
    if (cache->current_size < cache->capacity) {
        // 找到第一个空帧
        int empty_frame_idx = -1;
        for(int i=0; i<cache->capacity; ++i) {
            if(cache->frames[i].page_number == -1) {
                empty_frame_idx = i;
                break;
            }
        }
        cache->frames[empty_frame_idx].page_number = page_num;
        cache->current_size++;
    } else { // 缓存已满，随机选择一个帧替换
        int victim_frame_idx = generate_random_int(0, cache->capacity - 1);
        cache->frames[victim_frame_idx].page_number = page_num;
    }
    return 1; // 缺页
}

void random_alg_destroy(Cache* cache) {
    // 随机算法不需要额外的清理工作
    cache->alg_data = NULL;
}
```

#### `clock_alg.c` (时钟算法)：

```c
#include "algorithms.h"
#include "utils.h"

void clock_alg_init(Cache* cache, int capacity) {
    cache->capacity = capacity > MAX_CACHE_SIZE ? MAX_CACHE_SIZE : capacity;
    cache->current_size = 0;
    for (int i = 0; i < cache->capacity; ++i) {
        cache->frames[i].page_number = -1;
        cache->frames[i].use_bit = false;
    }
    
    Clock_Data* data = (Clock_Data*)malloc(sizeof(Clock_Data));
    if (!data) die("分配Clock_Data失败");
    data->clock_hand = 0;
    cache->alg_data = data;
}

int clock_alg_access(Cache* cache, int page_num, int current_time_step) {
    Clock_Data* data = (Clock_Data*)cache->alg_data;
    
    // 检查是否命中
    for (int i = 0; i < cache->current_size; ++i) {
        if (cache->frames[i].page_number == page_num) {
            cache->frames[i].use_bit = true; // 设置使用位
            return 0; // 命中
        }
    }

    // 缺页处理
    if (cache->current_size < cache->capacity) {
        // 找到第一个空帧
        int empty_frame_idx = -1;
        for(int i=0; i<cache->capacity; ++i) {
            if(cache->frames[i].page_number == -1) {
                empty_frame_idx = i;
                break;
            }
        }
        cache->frames[empty_frame_idx].page_number = page_num;
        cache->frames[empty_frame_idx].use_bit = true;
        cache->current_size++;
    } else { // 缓存已满，使用时钟算法选择受害者
        while (true) {
            // 检查当前位置的使用位
            if (!cache->frames[data->clock_hand].use_bit) {
                // 找到受害者
                cache->frames[data->clock_hand].page_number = page_num;
                cache->frames[data->clock_hand].use_bit = true;
                data->clock_hand = (data->clock_hand + 1) % cache->capacity;
                break;
            } else {
                // 清除使用位并移动指针
                cache->frames[data->clock_hand].use_bit = false;
                data->clock_hand = (data->clock_hand + 1) % cache->capacity;
            }
        }
    }
    return 1; // 缺页
}

void clock_alg_destroy(Cache* cache) {
    if (cache->alg_data) {
        free(cache->alg_data);
        cache->alg_data = NULL;
    }
}
```

#### `lfu.c` (最不常用算法)：

```c
#include "algorithms.h"
#include "utils.h"
#include <limits.h>

void lfu_init(Cache* cache, int capacity) {
    cache->capacity = capacity > MAX_CACHE_SIZE ? MAX_CACHE_SIZE : capacity;
    cache->current_size = 0;
    for (int i = 0; i < cache->capacity; ++i) {
        cache->frames[i].page_number = -1;
        cache->frames[i].frequency = 0;
        cache->frames[i].last_used_time = 0; // 用于平局处理（使用FIFO或LRU）
    }
    
    LFU_Data* data = (LFU_Data*)malloc(sizeof(LFU_Data));
    if (!data) die("分配LFU_Data失败");
    cache->alg_data = data;
}

int lfu_access(Cache* cache, int page_num, int current_time_step) {
    // 检查是否命中
    for (int i = 0; i < cache->current_size; ++i) {
        if (cache->frames[i].page_number == page_num) {
            cache->frames[i].frequency++;
            cache->frames[i].last_used_time = current_time_step; // 用于平局处理
            return 0; // 命中
        }
    }

    // 缺页处理
    if (cache->current_size < cache->capacity) {
        // 找到第一个空帧
        int empty_frame_idx = -1;
        for(int i=0; i<cache->capacity; ++i) {
            if(cache->frames[i].page_number == -1) {
                empty_frame_idx = i;
                break;
            }
        }
        cache->frames[empty_frame_idx].page_number = page_num;
        cache->frames[empty_frame_idx].frequency = 1;
        cache->frames[empty_frame_idx].last_used_time = current_time_step;
        cache->current_size++;
    } else { // 缓存已满，找到频率最低的页面
        int lfu_frame_idx = -1;
        int min_frequency = INT_MAX;
        int oldest_time = INT_MAX;
        
        for (int i = 0; i < cache->capacity; ++i) {
            if (cache->frames[i].frequency < min_frequency ||
                (cache->frames[i].frequency == min_frequency && 
                 cache->frames[i].last_used_time < oldest_time)) {
                min_frequency = cache->frames[i].frequency;
                oldest_time = cache->frames[i].last_used_time;
                lfu_frame_idx = i;
            }
        }
        
        cache->frames[lfu_frame_idx].page_number = page_num;
        cache->frames[lfu_frame_idx].frequency = 1;
        cache->frames[lfu_frame_idx].last_used_time = current_time_step;
    }
    return 1; // 缺页
}

void lfu_destroy(Cache* cache) {
    if (cache->alg_data) {
        free(cache->alg_data);
        cache->alg_data = NULL;
    }
}
```

#### `wtinylfu.c` (简化实现)：

```c
#include "algorithms.h"
#include "utils.h"
#include <string.h>
#include <limits.h>

// W-TinyLFU的简化实现
// 注意：这是一个简化版本，实际的W-TinyLFU要复杂得多

void wtinylfu_init(Cache* cache, int capacity, int max_unique_pages) {
    cache->capacity = capacity > MAX_CACHE_SIZE ? MAX_CACHE_SIZE : capacity;
    cache->current_size = 0;
    
    WTinyLFU_Data* data = (WTinyLFU_Data*)malloc(sizeof(WTinyLFU_Data));
    if (!data) die("分配WTinyLFU_Data失败");
    
    // 分配全局频率数组
    data->global_page_frequencies = (int*)calloc(max_unique_pages, sizeof(int));
    if (!data->global_page_frequencies) die("分配全局频率数组失败");
    data->max_unique_pages_for_freq = max_unique_pages;
    
    // 初始化SLRU分段
    data->probation_capacity = (int)(WTINYLFU_PROBATION_RATIO * cache->capacity);
    if (data->probation_capacity < 1) data->probation_capacity = 1;
    data->protected_capacity = cache->capacity - data->probation_capacity;
    
    data->time_counter = 0;
    
    // 初始化所有帧
    for (int i = 0; i < cache->capacity; ++i) {
        cache->frames[i].page_number = -1;
        cache->frames[i].frequency = 0;
        cache->frames[i].last_used_time = 0;
    }
    
    cache->alg_data = data;
}

int wtinylfu_access(Cache* cache, int page_num, int current_time_step) {
    WTinyLFU_Data* data = (WTinyLFU_Data*)cache->alg_data;
    
    // 更新全局频率
    if (page_num < data->max_unique_pages_for_freq) {
        data->global_page_frequencies[page_num]++;
    }
    data->time_counter = current_time_step;
    
    // 检查是否命中
    for (int i = 0; i < cache->capacity; ++i) {
        if (cache->frames[i].page_number == page_num) {
            cache->frames[i].frequency++;
            cache->frames[i].last_used_time = data->time_counter;
            // 简化版：不实现试用段到保护段的晋升逻辑
            return 0; // 命中
        }
    }
    
    // 缺页处理
    if (cache->current_size < cache->capacity) {
        // 找到第一个空帧
        int empty_frame_idx = -1;
        for(int i=0; i<cache->capacity; ++i) {
            if(cache->frames[i].page_number == -1) {
                empty_frame_idx = i;
                break;
            }
        }
        cache->frames[empty_frame_idx].page_number = page_num;
        cache->frames[empty_frame_idx].frequency = 1;
        cache->frames[empty_frame_idx].last_used_time = data->time_counter;
        cache->current_size++;
    } else { // 缓存已满，需要选择受害者
        // 简化版：基于频率和LRU的组合来选择受害者
        int victim_frame_idx = -1;
        int min_frequency = INT_MAX;
        int oldest_time = INT_MAX;
        
        // 找到试用段范围内频率最低的页面（简化：前20%的帧）
        int probation_end = data->probation_capacity;
        for (int i = 0; i < probation_end && i < cache->capacity; ++i) {
            if (cache->frames[i].frequency < min_frequency ||
                (cache->frames[i].frequency == min_frequency && 
                 cache->frames[i].last_used_time < oldest_time)) {
                min_frequency = cache->frames[i].frequency;
                oldest_time = cache->frames[i].last_used_time;
                victim_frame_idx = i;
            }
        }
        
        // 如果试用段没有找到，从保护段找（简化：剩余的帧）
        if (victim_frame_idx == -1) {
            for (int i = probation_end; i < cache->capacity; ++i) {
                if (cache->frames[i].frequency < min_frequency ||
                    (cache->frames[i].frequency == min_frequency && 
                     cache->frames[i].last_used_time < oldest_time)) {
                    min_frequency = cache->frames[i].frequency;
                    oldest_time = cache->frames[i].last_used_time;
                    victim_frame_idx = i;
                }
            }
        }
        
        // 替换受害者
        if (victim_frame_idx != -1) {
            cache->frames[victim_frame_idx].page_number = page_num;
            cache->frames[victim_frame_idx].frequency = 1;
            cache->frames[victim_frame_idx].last_used_time = data->time_counter;
        }
    }
    
    return 1; // 缺页
}

void wtinylfu_destroy(Cache* cache) {
    if (cache->alg_data) {
        WTinyLFU_Data* data = (WTinyLFU_Data*)cache->alg_data;
        if (data->global_page_frequencies) {
            free(data->global_page_frequencies);
        }
        free(data);
        cache->alg_data = NULL;
    }
}
```

## III. 额外的可视化和分析工具

### 1. 自动化分析脚本 (`analyze_results.sh`)

```bash
#!/bin/bash
# analyze_results.sh - 自动化运行模拟和生成图表

# 检查必要的目录
if [ ! -d "results" ]; then
    mkdir results
fi

# 清理旧结果
rm -f results/*.csv results/*.png

# 运行模拟
echo "开始运行页面替换算法模拟..."
./page_sim

# 检查是否生成了结果文件
if [ ! -f "results/simulation_results.csv" ]; then
    echo "错误：未找到模拟结果文件"
    exit 1
fi

# 生成图表
echo "生成可视化图表..."
gnuplot plot_results.gp

# 或者使用Python脚本
# python3 visualize.py

echo "分析完成！结果保存在results目录中"
```

### 2. 性能比较表格生成器 (`generate_comparison_table.py`)

```python
#!/usr/bin/env python3
# generate_comparison_table.py - 生成算法性能比较表格

import pandas as pd
import sys

if len(sys.argv) < 2:
    print("用法: python3 generate_comparison_table.py <csv文件路径>")
    sys.exit(1)

csv_file = sys.argv[1]

try:
    df = pd.read_csv(csv_file)
    
    # 创建透视表，显示每种工作负载和缓存大小下不同算法的缺页率
    pivot_table = df.pivot_table(
        values='FaultRate',
        index=['WorkloadType', 'CacheSize'],
        columns='Algorithm'
    )
    
    print("\n页面替换算法性能比较表:")
    print("=" * 80)
    print(pivot_table)
    
    # 计算每种算法在不同工作负载下的平均缺页率
    avg_by_workload = df.groupby(['WorkloadType', 'Algorithm'])['FaultRate'].mean()
    print("\n\n不同工作负载下各算法的平均缺页率:")
    print("=" * 60)
    print(avg_by_workload)
    
    # 找出每种工作负载下表现最好的算法
    best_algorithms = df.loc[df.groupby(['WorkloadType', 'CacheSize'])['FaultRate'].idxmin()]
    print("\n\n每种工作负载和缓存大小下的最佳算法:")
    print("=" * 60)
    print(best_algorithms[['WorkloadType', 'CacheSize', 'Algorithm', 'FaultRate']])
    
except Exception as e:
    print(f"错误: {e}")
```

### 3. 实验报告生成器 (`generate_report.py`)

```python
#!/usr/bin/env python3
# generate_report.py - 自动生成实验报告

import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from datetime import datetime
import os

RESULTS_DIR = "results"
REPORT_FILE = os.path.join(RESULTS_DIR, "experiment_report.md")

def generate_report():
    # 读取数据
    df = pd.read_csv(os.path.join(RESULTS_DIR, "simulation_results.csv"))
    
    # 开始生成报告
    with open(REPORT_FILE, 'w', encoding='utf-8') as f:
        f.write("# 页面替换算法模拟实验报告\n\n")
        f.write(f"生成日期: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n")
        
        # 实验概述
        f.write("## 1. 实验概述\n\n")
        f.write("本实验模拟了6种页面替换算法在3种不同工作负载下的性能表现。\n\n")
        f.write("### 测试的算法:\n")
        algorithms = df['Algorithm'].unique()
        for alg in algorithms:
            f.write(f"- {alg}\n")
        
        f.write("\n### 工作负载类型:\n")
        workloads = df['WorkloadType'].unique()
        for wl in workloads:
            f.write(f"- {wl}\n")
        
        # 实验结果
        f.write("\n## 2. 实验结果\n\n")
        
        # 为每种工作负载生成分析
        for workload in workloads:
            f.write(f"### 2.{list(workloads).index(workload)+1} {workload} 工作负载\n\n")
            
            wl_data = df[df['WorkloadType'] == workload]
            
            # 计算统计信息
            stats = wl_data.groupby('Algorithm')['FaultRate'].agg(['mean', 'std', 'min', 'max'])
            
            f.write("#### 统计信息:\n\n")
            f.write("| 算法 | 平均缺页率 | 标准差 | 最小值 | 最大值 |\n")
            f.write("|------|------------|--------|--------|--------|\n")
            
            for alg in stats.index:
                row = stats.loc[alg]
                f.write(f"| {alg} | {row['mean']:.4f} | {row['std']:.4f} | {row['min']:.4f} | {row['max']:.4f} |\n")
            
            f.write(f"\n![{workload} 缺页率对比图]({workload}_FaultRate.png)\n\n")
        
        # 结论
        f.write("## 3. 结论\n\n")
        
        # 找出每种工作负载下的最佳算法
        best_algs = df.groupby(['WorkloadType'])['FaultRate'].mean().groupby('WorkloadType').idxmin()
        
        f.write("### 各工作负载下的最佳算法:\n\n")
        for workload in workloads:
            best_alg_data = df[(df['WorkloadType'] == workload)].groupby('Algorithm')['FaultRate'].mean()
            best_alg = best_alg_data.idxmin()
            best_rate = best_alg_data.min()
            f.write(f"- **{workload}**: {best_alg} (平均缺页率: {best_rate:.4f})\n")
        
        f.write("\n### 总体观察:\n\n")
        f.write("1. 不同的工作负载特征对算法性能有显著影响\n")
        f.write("2. 没有一种算法在所有情况下都是最优的\n")
        f.write("3. LRU和W-TinyLFU在具有局部性的工作负载下表现较好\n")
        f.write("4. 随机替换算法在所有情况下表现都较差\n")
        
        print(f"实验报告已生成: {REPORT_FILE}")

if __name__ == "__main__":
    generate_report()
```

## IV. 完整的实验流程

### 1. 编译和运行

```bash
# 1. 创建项目目录结构
mkdir -p page_replacement_sim/{include,src,obj,data,results}
cd page_replacement_sim

# 2. 创建所有源文件（根据上面的代码）

# 3. 编译项目
make

# 4. 运行模拟
./page_sim

# 5. 生成可视化图表
python3 visualize.py

# 6. 生成性能比较表
python3 generate_comparison_table.py results/simulation_results.csv

# 7. 生成实验报告
python3 generate_report.py
```

### 2. 调试建议

如果遇到问题，可以添加调试输出：

```c
// 在算法实现中添加调试信息
#ifdef DEBUG
    printf("Debug: Algorithm=%s, Page=%d, CacheSize=%d, CurrentSize=%d\n", 
           "FIFO", page_num, cache->capacity, cache->current_size);
#endif
```

编译时启用调试：

```bash
make CFLAGS="-DDEBUG -g"
```

## V. 进一步的改进方向

1. **添加更多度量指标：**

    - 平均访问时间
    - 命中率
    - 算法执行时间
2. **实现更复杂的工作负载：**

    - 多线程工作负载
    - 混合模式（如80-20和顺序循环的组合）
3. **优化数据结构：**

    - 使用哈希表加速查找
    - 实现更高效的LRU（双向链表+哈希表）
4. **参数化实验：**

    - 支持命令行参数
    - 配置文件支持
5. **并行化：**

    - 多个算法并行运行
    - 多个工作负载并行测试

这个完整的框架应该能够帮助你成功地在Ubuntu上实现页面替换算法的模拟实验，并进行可视化分析。

### 5. `include/simulator.h` 和 `src/simulator.c` (模拟器核心)

```c
// include/simulator.h
#ifndef SIMULATOR_H
#define SIMULATOR_H

#include "workload.h"
#include "algorithms.h"

typedef struct {
    const char* name;                     // 算法名称
    init_algorithm_func init_func;        // 初始化函数
    access_page_func access_func;         // 访问函数
    destroy_algorithm_func destroy_func;  // 销毁函数
    bool needs_max_unique_pages_on_init;  // 是否需要在初始化时提供最大唯一页面数（用于W-TinyLFU）
} AlgorithmEntry;

typedef struct {
    int page_faults;    // 缺页次数
    // 可以添加其他度量
} SimulationResult;

SimulationResult run_simulation(const Workload* workload,
                                AlgorithmEntry algorithm,
                                int cache_capacity,
                                int max_unique_pages_for_wtinylfu); // 如果不是W-TinyLFU则传0

#endif

// src/simulator.c
#include "simulator.h"
#include <stdio.h>

SimulationResult run_simulation(const Workload* workload,
                                AlgorithmEntry algorithm,
                                int cache_capacity,
                                int max_unique_pages_for_wtinylfu) {
    Cache cache;
    SimulationResult result = {0};
    int time_step = 0;

    if (algorithm.needs_max_unique_pages_on_init) {
        // W-TinyLFU或类似算法的特殊初始化
        ((void (*)(Cache*, int, int))algorithm.init_func)(&cache, cache_capacity, max_unique_pages_for_wtinylfu);
    } else {
        algorithm.init_func(&cache, cache_capacity);
    }

    for (int i = 0; i < workload->num_accesses; ++i) {
        time_step++;
        if (algorithm.access_func(&cache, workload->accesses[i], time_step)) {
            result.page_faults++;
        }
    }

    algorithm.destroy_func(&cache);
    return result;
}
```

### 6. `src/main.c` (主程序，实验编排)

```c
// src/main.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "utils.h"
#include "workload.h"
#include "algorithms.h"
#include "simulator.h"

#define NUM_ALGORITHMS 6 // FIFO、Random、LRU、Clock、LFU、W-TinyLFU

int main() {
    srand(time(NULL)); // 随机数生成器的种子

    AlgorithmEntry algorithms[NUM_ALGORITHMS] = {
        {"FIFO", fifo_init, fifo_access, fifo_destroy, false},
        {"Random", random_alg_init, random_alg_access, random_alg_destroy, false},
        {"LRU", lru_init, lru_access, lru_destroy, false},
        {"Clock", clock_alg_init, clock_alg_access, clock_alg_destroy, false},
        {"LFU", lfu_init, lfu_access, lfu_destroy, false},
        {"W-TinyLFU", (init_algorithm_func)wtinylfu_init, wtinylfu_access, wtinylfu_destroy, true}
    };

    // --- 实验参数 ---
    int total_accesses = 10000;             // 总访问次数
    int unique_pages = 100;                 // 唯一页面数
    int loop_length_for_seq = 50;           // 顺序循环工作负载的循环长度

    // 要测试的缓存大小（示例）
    int cache_sizes[] = {10, 20, 30, 40, 50, 60, 70, 80, 90, 100,
                         (loop_length_for_seq > 0 ? loop_length_for_seq - 1 : 0), // 例如，循环长度50时为49
                         loop_length_for_seq,                                     // 例如，50
                         (loop_length_for_seq < unique_pages ? loop_length_for_seq + 1: 0)}; // 例如，51
    int num_cache_sizes = sizeof(cache_sizes) / sizeof(cache_sizes[0]);
    // 如果loop_length_for_seq很小或未设置，过滤掉0或重复的缓存大小
    // ... （添加逻辑以清理cache_sizes数组）

    // 打开CSV文件以保存结果
    FILE* results_file = fopen("results/simulation_results.csv", "w");
    if (!results_file) {
        die("无法打开results.csv");
    }
    fprintf(results_file, "WorkloadType,Algorithm,CacheSize,TotalAccesses,PageFaults,FaultRate\n");

    // --- 定义工作负载 ---
    typedef struct {
        const char* name;
        Workload wl;
    } WorkloadEntry;

    WorkloadEntry workloads[] = {
        {"NoLocality", generate_no_locality_workload(total_accesses, unique_pages)},
        {"80-20", generate_80_20_workload(total_accesses, unique_pages)},
        {"SequentialLoop", generate_sequential_loop_workload(total_accesses, unique_pages, loop_length_for_seq)}
    };
    int num_workloads = sizeof(workloads) / sizeof(workloads[0]);

    // --- 运行实验 ---
    for (int wl_idx = 0; wl_idx < num_workloads; ++wl_idx) {
        printf("运行工作负载: %s\n", workloads[wl_idx].name);
        for (int alg_idx = 0; alg_idx < NUM_ALGORITHMS; ++alg_idx) {
            printf("  算法: %s\n", algorithms[alg_idx].name);
            for (int cs_idx = 0; cs_idx < num_cache_sizes; ++cs_idx) {
                int current_cache_size = cache_sizes[cs_idx];
                if (current_cache_size <= 0 || current_cache_size > MAX_CACHE_SIZE) continue; // 跳过无效的缓存大小

                printf("    缓存大小: %d ... ", current_cache_size);
                fflush(stdout);

                SimulationResult result = run_simulation(&workloads[wl_idx].wl,
                                                         algorithms[alg_idx],
                                                         current_cache_size,
                                                         unique_pages); // 为W-TinyLFU全局频率映射传递unique_pages

                float fault_rate = (float)result.page_faults / workloads[wl_idx].wl.num_accesses;
                printf("缺页: %d, 缺页率: %.4f\n", result.page_faults, fault_rate);

                fprintf(results_file, "%s,%s,%d,%d,%d,%.4f\n",
                        workloads[wl_idx].name,
                        algorithms[alg_idx].name,
                        current_cache_size,
                        workloads[wl_idx].wl.num_accesses,
                        result.page_faults,
                        fault_rate);
            }
        }
    }

    fclose(results_file);
    printf("模拟完成。结果已保存到 results/simulation_results.csv\n");

    return 0;
}
```

### 7. `Makefile` (编译脚本)

```makefile
CC = gcc
CFLAGS = -Wall -Wextra -g -Iinclude -std=c11 # -g用于调试
LDFLAGS =

# 查找src目录中的所有.c文件
SRCS = $(wildcard src/*.c)
# 从源文件名生成目标文件名
OBJS = $(patsubst src/%.c,obj/%.o,$(SRCS))

TARGET = page_sim

# 默认目标
all: $(TARGET)

# 链接目标可执行文件
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $^ -o $@ $(LDFLAGS)

# 将源文件编译为目标文件
obj/%.o: src/%.c | obj
	$(CC) $(CFLAGS) -c $< -o $@

# 目标文件目录
obj:
	mkdir -p obj

# 结果目录
results:
	mkdir -p results
	
# 数据目录（如果保存工作负载）
data:
	mkdir -p data

# 清理构建文件
clean:
	rm -rf $(TARGET) obj/* results/* data/*

.PHONY: all clean results data
```

## 编译与运行：

1. 创建目录结构：`mkdir include src obj data results`
2. 将上述代码片段保存到对应的文件中。
3. 在 `page_replacement_sim` 目录下运行 `make`。
4. 运行 `./page_sim`。
5. 结果会保存在 `results/simulation_results.csv`。

---

## II. 实验结果可视化方案

一旦 `simulation_results.csv` 文件生成，你可以使用多种工具进行可视化。

### 1. Gnuplot (命令行绘图工具，适合集成到脚本中)

- **安装：** `sudo apt update && sudo apt install gnuplot`
- **Gnuplot脚本示例 (`plot_results.gp`):**

```gnuplot
# plot_results.gp

set terminal pngcairo enhanced font "arial,10" size 800,600
set datafile separator "," # CSV格式

# --- "NoLocality"工作负载的图表 ---
set output "results/NoLocality_FaultRate.png"
set title "缺页率 vs. 缓存大小 (工作负载: 无局部性)"
set xlabel "缓存大小（页帧数）"
set ylabel "缺页率"
set key top right
set grid

# 使用awk进行数据过滤的更健壮方法
plot '< awk -F, ''$1 == "NoLocality" && $2 == "FIFO" {print $3, $6}'' results/simulation_results.csv' title "FIFO (无局部性)" with linespoints, \
     '< awk -F, ''$1 == "NoLocality" && $2 == "Random" {print $3, $6}'' results/simulation_results.csv' title "Random (无局部性)" with linespoints, \
     '< awk -F, ''$1 == "NoLocality" && $2 == "LRU" {print $3, $6}'' results/simulation_results.csv' title "LRU (无局部性)" with linespoints, \
     '< awk -F, ''$1 == "NoLocality" && $2 == "Clock" {print $3, $6}'' results/simulation_results.csv' title "Clock (无局部性)" with linespoints, \
     '< awk -F, ''$1 == "NoLocality" && $2 == "LFU" {print $3, $6}'' results/simulation_results.csv' title "LFU (无局部性)" with linespoints, \
     '< awk -F, ''$1 == "NoLocality" && $2 == "W-TinyLFU" {print $3, $6}'' results/simulation_results.csv' title "W-TinyLFU (无局部性)" with linespoints

# --- "80-20"工作负载的图表 ---
set output "results/80-20_FaultRate.png"
set title "缺页率 vs. 缓存大小 (工作负载: 80-20)"
# xlabel、ylabel、key、grid相同
plot '< awk -F, ''$1 == "80-20" && $2 == "FIFO" {print $3, $6}'' results/simulation_results.csv' title "FIFO (80-20)" with linespoints, \
     '< awk -F, ''$1 == "80-20" && $2 == "Random" {print $3, $6}'' results/simulation_results.csv' title "Random (80-20)" with linespoints, \
     '< awk -F, ''$1 == "80-20" && $2 == "LRU" {print $3, $6}'' results/simulation_results.csv' title "LRU (80-20)" with linespoints, \
     '< awk -F, ''$1 == "80-20" && $2 == "Clock" {print $3, $6}'' results/simulation_results.csv' title "Clock (80-20)" with linespoints, \
     '< awk -F, ''$1 == "80-20" && $2 == "LFU" {print $3, $6}'' results/simulation_results.csv' title "LFU (80-20)" with linespoints, \
     '< awk -F, ''$1 == "80-20" && $2 == "W-TinyLFU" {print $3, $6}'' results/simulation_results.csv' title "W-TinyLFU (80-20)" with linespoints

# --- "SequentialLoop"工作负载的图表 ---
set output "results/SequentialLoop_FaultRate.png"
set title "缺页率 vs. 缓存大小 (工作负载: 顺序循环)"
# xlabel、ylabel、key、grid相同
plot '< awk -F, ''$1 == "SequentialLoop" && $2 == "FIFO" {print $3, $6}'' results/simulation_results.csv' title "FIFO (顺序循环)" with linespoints, \
     '< awk -F, ''$1 == "SequentialLoop" && $2 == "Random" {print $3, $6}'' results/simulation_results.csv' title "Random (顺序循环)" with linespoints, \
     '< awk -F, ''$1 == "SequentialLoop" && $2 == "LRU" {print $3, $6}'' results/simulation_results.csv' title "LRU (顺序循环)" with linespoints, \
     '< awk -F, ''$1 == "SequentialLoop" && $2 == "Clock" {print $3, $6}'' results/simulation_results.csv' title "Clock (顺序循环)" with linespoints, \
     '< awk -F, ''$1 == "SequentialLoop" && $2 == "LFU" {print $3, $6}'' results/simulation_results.csv' title "LFU (顺序循环)" with linespoints, \
     '< awk -F, ''$1 == "SequentialLoop" && $2 == "W-TinyLFU" {print $3, $6}'' results/simulation_results.csv' title "W-TinyLFU (顺序循环)" with linespoints
```

- **运行Gnuplot脚本：** `gnuplot plot_results.gp` 这会生成PNG图片到 `results/` 目录下。

### 2. Python with Matplotlib/Seaborn (更灵活，交互性更好)

- **安装：** `sudo apt install python3-pip && pip3 install pandas matplotlib seaborn`
- **Python脚本示例 (`visualize.py`):**

```python
# visualize.py
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import os

RESULTS_DIR = "results"
CSV_FILE = os.path.join(RESULTS_DIR, "simulation_results.csv")

def plot_results(df):
    if not os.path.exists(RESULTS_DIR):
        os.makedirs(RESULTS_DIR)

    sns.set_theme(style="whitegrid")

    workload_types = df['WorkloadType'].unique()
    algorithms = df['Algorithm'].unique()

    for workload in workload_types:
        plt.figure(figsize=(12, 7))
        
        workload_df = df[df['WorkloadType'] == workload]
        
        for algorithm in algorithms:
            alg_df = workload_df[workload_df['Algorithm'] == algorithm]
            if not alg_df.empty:
                # 按CacheSize排序以正确绘制线条
                alg_df = alg_df.sort_values(by='CacheSize')
                plt.plot(alg_df['CacheSize'], alg_df['FaultRate'], marker='o', linestyle='-', label=algorithm)
        
        # 中文标题和标签
        workload_chinese = {
            'NoLocality': '无局部性',
            '80-20': '80-20局部性', 
            'SequentialLoop': '顺序循环'
        }.get(workload, workload)
        
        plt.title(f'缺页率 vs. 缓存大小 (工作负载: {workload_chinese})')
        plt.xlabel('缓存大小（页帧数）')
        plt.ylabel('缺页率')
        plt.legend(title='算法')
        plt.grid(True)
        
        plot_filename = os.path.join(RESULTS_DIR, f'{workload.replace(" ", "_")}_FaultRate.png')
        plt.savefig(plot_filename)
        plt.close()
        print(f"图表已保存: {plot_filename}")

if __name__ == "__main__":
    try:
        data = pd.read_csv(CSV_FILE)
        plot_results(data)
    except FileNotFoundError:
        print(f"错误：未找到结果文件 {CSV_FILE}")
    except Exception as e:
        print(f"发生错误: {e}")
```

- **运行Python脚本：** `python3 visualize.py` 这同样会生成PNG图片到 `results/` 目录下。Python提供了更强大的数据处理和绘图定制能力。

### 3. LibreOffice Calc / Microsoft Excel (手动导入CSV绘图)

你可以直接用电子表格软件打开 `simulation_results.csv`，然后使用其内置的图表功能来创建折线图。选择合适的列作为X轴（CacheSize）和Y轴（FaultRate），并按算法和工作负载类型进行分组。

---

## 重要提示和下一步：

- **错误处理和健壮性：** 上述C代码是框架性的，需要添加更完善的错误检查（例如，`malloc` 返回 `NULL` 的处理）。
- **算法实现的正确性：** 仔细测试每种页面替换算法的逻辑，确保其行为符合预期。特别是W-TinyLFU的简化版，要明确其工作方式。
- **效率：** 对于LRU和W-TinyLFU中的段管理，如果缓存非常大或访问序列非常长，基于数组扫描的查找会很慢。实际高性能实现会使用更复杂的数据结构（如双向链表配合哈希表来快速定位节点）。对于模拟实验，当前方法可能足够。
- **参数化：** 使 `main.c` 中的实验参数（如 `total_accesses`, `unique_pages`, `cache_sizes` 范围）更易于通过命令行参数配置。
- **模块化：** 确保 `algorithms.h` 中的接口清晰，各个算法的 `.c` 文件独立实现这些接口。

## 补充说明：

1. **算法实现复杂度说明：**

    - FIFO、Random：相对简单，容易实现
    - LRU、Clock：中等复杂度
    - LFU：需要处理频率计数和平局处理
    - W-TinyLFU：最复杂，需要实现SLRU结构和全局频率管理
2. **性能优化建议：**

    - 使用哈希表加速页面查找
    - 对于LRU，考虑使用双向链表
    - 对于W-TinyLFU，使用更高效的数据结构
3. **实验扩展方向：**

    - 添加更多工作负载模式
    - 测试不同的缓存大小范围
    - 添加性能统计（如执行时间）
    - 支持多级缓存模拟

