 
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
就目前代码看，要求的其实是当前事件的上一个事件有时间戳则不中断
1 时间戳必须连续，不能缺失，2 事件log中不能包含无效事件(不在事件链中)
目前来看其实有点疑惑，虽然能保证非事件链中事件出现然后中断，但是在事件链中的会出现不一致的情况
当时间序列 [1,3,2,3,4,2] 时，返回为2， 当时间序列[1,3,4,3,4,2] 返回为4 ，函数执行windowFunnel(10,'strict_order')(va, vb =1, vb=3, vb=4,vb=2) 

strict_deduplication
当前事件是已经存在，则返回上一个记录的事件级别，事件级别1在考虑范围内
这里其实是有问题的，如果出现1 3 1 3 这种事件序列，那么返回的是1 ，但是实际上应该是2 才对， 也就是直接返回 当前事件级别 就行吧

strict_increase 
不知道使用场景

```  sql 
with [1,5,3,3,4,2] as b ,
     [1,2,3,4,6,5] as a,
t as ( SELECT a va , b vb  )
select
    windowFunnel(10,'strict_order')(va, vb =1, vb=3, vb=4, vb =2) orders ,
    windowFunnel(10,'strict_deduplication')(va, vb =1, vb=3, vb=4, vb =2) dups, 
    windowFunnel(10,'strict_increase')(va, vb =1, vb=3, vb=4, vb =2) increase, 
from
    t 
    array join 
        va  ,
        vb  ;
```

windowFunnel 函数中计算level的函数实现,加了点注释，学习学习，方便更好的使用
  https://github.com/ClickHouse/ClickHouse/blob/master/src/AggregateFunctions/AggregateFunctionWindowFunnel.h#L153
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
        else if (strict_deduplication && events_timestamp[event_idx].has_value())
        {
            return data.events_list[i - 1].second; 
        }
        // 如果开启了严格顺序且存在第一个事件，但前一个事件级别对应的时间戳不存在，则返回第一个空缺的事件级别
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
 

 
