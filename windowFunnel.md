[png]:

``` sql 
with t as (
    select
        [ randomPrintableASCII(1),
        randomPrintableASCII(1) ] a,
        [ randBinomial(100, 0.45),
        randBinomial(100, 0.8) ] b
    from
        numbers(10)
)
select
    groupArrayResampleArray(1, 100, 50)(a, b) t  
from
    t;

with tmp as (
    select
        number a,
        [ randBinomial(10, 0.35),
        randBinomial(10, 0.45) ] b
    from
        numbers(1000)
)
select
    sequenceCount('(?1)(?t<2)(?2)')(a, b = [ 2, 3 ], b = [ 3, 4 ]) t,
    length(arrayFilter(x - > x > 5 ? 1: 0, groupArrayArray(b))) l
from
    tmp;

```


windowFunnel
window 是当前的事件下标 <= 第一个事件的下标+ window

strict_order
不属于事件链的下一个事件会中断事件链，有序包含了2层含义
1 时间戳必须连续，不能缺失，2 事件log中不能包含无效事件(不在事件链中)

strict_deduplication
当前事件是已经存在，则返回上一个记录的事件级别，事件级别1在考虑范围内
事件链事件按照时间序列   

strict_increase 
基于数据已经通过时间戳进行排序，同时也保证了事件链的顺序插入，暂时没想到解决的场景是什么

```  sql 
with [1,5,3,3,4,2] as b ,
     [1,2,3,4,6,5] as a,
t as ( SELECT a va , b vb  )
select
    sequenceCount('(?1)(?t<3)(?2)')(va, vb = 1, vb=4) ta,
    groupArray(vb) tb,
    windowFunnel(10,'strict_order')(va, vb =1, vb=3, vb=4, vb =2) orders ,
    windowFunnel(10,'strict_deduplication')(va, vb =1, vb=3, vb=4, vb =2) dups, 
    windowFunnel(10,'strict_increase')(va, vb =1, vb=3, vb=4, vb =2) increase, 
    windowFunnel(10)(va, vb =1, vb=3, vb=4, vb =2)  default -- increase
from
    t 
    array join 
        va  ,
        vb  ;
```

windowFunnel 函数中计算level的函数实现

``` c++
UInt8 getEventLevel(Data & data) const
{
    // 如果事件列表为空，则返回 0
    if (data.size() == 0)
        return 0;
    // 如果不允许改变事件顺序且事件级别为 1 的数量为 1，则直接返回 1
    if (!strict_order && events_size == 1)
        return 1;

    // 对事件列表按时间戳进行排序
    data.sort();

    // events_timestamp 存储时间窗口内每个事件级别的第一个和上一个事件的时间戳
    // 使用 std::optional 来标识是否存在值
    std::vector<std::optional<std::pair<UInt64, UInt64>>> events_timestamp(events_size);

    // 记录是否存在第一个事件
    bool first_event = false;

    // 遍历事件列表
    for (size_t i = 0; i < data.events_list.size(); ++i)
    {   
        // 当前事件的时间戳
        const T & timestamp = data.events_list[i].first;
        // 当前事件的事件级别
        const auto & event_idx = data.events_list[i].second - 1;

        // 如果严格顺序且事件级别为 -1（无效），则判断是否存在第一个事件
        if (strict_order && event_idx == -1)
        {
            if (first_event)
                break;
            else
                continue;
        }
        // 如果事件级别为 1
        else if (event_idx == 0)
        {
            // 更新事件级别为 1 的事件的时间戳信息，并标记存在第一个事件
            events_timestamp[0] = std::make_pair(timestamp, timestamp);
            first_event = true;
        }
        // 如果开启了严格去重且事件级别对应的时间戳已存在，则返回前一个事件的级别
        // 事件级别1 是特殊的，不需要考虑strict_deduplication
        // 这里其实是有问题的，如果出现1 3 1 3 这种事件序列，那么返回的是1 ，但是实际上应该是2 才对
        else if (strict_deduplication && events_timestamp[event_idx].has_value())
        {
            return data.events_list[i - 1].second; // 这段代码有bug，会出现完全不同的情况 如下图
        }
        // 如果开启了严格顺序且存在第一个事件，但前一个事件级别对应的时间戳不存在，则返回第一个空缺的事件级别
        // 时间戳为null时的情况
        else if (strict_order && first_event && !events_timestamp[event_idx - 1].has_value())
        {
            for (size_t event = 0; event < events_timestamp.size(); ++event)
            {
                if (!events_timestamp[event].has_value())
                    return event;
            }
        }
        // 如果前一个事件级别对应的时间戳存在,保证了处理log时是按照事件链的顺序执行
        else if (events_timestamp[event_idx - 1].has_value())
        {
            auto first_timestamp = events_timestamp[event_idx - 1]->first;

            // 检查时间戳是否在时间窗口内，并根据 strict_increase 条件判断是否严格递增
            bool time_matched = timestamp <= first_timestamp + window;
            if (strict_increase)
                time_matched = time_matched && events_timestamp[event_idx - 1]->second < timestamp;

            // 如果满足条件，则更新事件级别为 event_idx 的时间戳信息
            if (time_matched)
            {
                events_timestamp[event_idx] = std::make_pair(first_timestamp, timestamp);
                if (event_idx + 1 == events_size)
                    return events_size;
            }
        }
    }

    // 遍历 events_timestamp，返回最大的事件级别
    for (size_t event = events_timestamp.size(); event > 0; --event)
    {
        if (events_timestamp[event - 1].has_value())
            return event;
    }

    // 如果所有事件级别对应的时间戳都为空，则返回 0
    return 0;
}
```

![][png]

``` c++

// 列数据首先整理成AggregateFunctionWindowFunnelData结构体，add方法用于将数据添加到事件列表中，后续共计算level调用

void add(AggregateDataPtr __restrict place, const IColumn ** columns, const size_t row_num, Arena *) const override
{
    bool has_event = false;
    const auto timestamp = assert_cast<const ColumnVector<T> *>(columns[0])->getData()[row_num];
    /// reverse iteration and stable sorting are needed for events that are qualified by more than one condition.
    for (auto i = events_size; i > 0; --i)
    {
        auto event = assert_cast<const ColumnVector<UInt8> *>(columns[i])->getData()[row_num];
        if (event)
        {
            this->data(place).add(timestamp, i);
            has_event = true;
        }
    }

    if (strict_order && !has_event)
        this->data(place).add(timestamp, 0); // 0用来标识非事件链中事件
}

static constexpr size_t max_events = 32;

template <typename T>
struct AggregateFunctionWindowFunnelData
{
    using TimestampEvent = std::pair<T, UInt8>;
    using TimestampEvents = PODArrayWithStackMemory<TimestampEvent, 64>;

    bool sorted = true;
    TimestampEvents events_list;
    
    void add(T timestamp, UInt8 event)
    {
        /// 由于大多数事件应该已经按照时间戳排序。
        /// 通过将当前时间戳与最后一个时间戳进行比较，检查 events_list 是否仍然按时间戳排序。
        /// 如果当前时间戳大于或等于最后一个时间戳，则保持排序标志为真。
        /// 否则，将排序标志设置为假。
        if (sorted && events_list.size() > 0)
        {
            if (events_list.back().first == timestamp)
                sorted = events_list.back().second <= event;
            else
                sorted = events_list.back().first <= timestamp;
        }
    
        /// 将新的事件（时间戳，事件）添加到 events_list 中。
        events_list.emplace_back(timestamp, event);
    }
    
    ......
}

```
 
