## ant design vue Table根据数据合并单元格

之前在外包做项目的时候，甲方提出一个想要合并单元格的需求，表格里展示的内容是领导们的一周行程，因为不想出现重复内容的单元格，实际场景中领导可能连续几天参加某个会议或者某个其他行程，本来 系统中对会议时间冲突是做了限制，也就是不能创建时间冲突的会议，那么对重复行程的单元格直接进行合并是没有问题的；但是后来又放开了限制、又允许存在会议时间冲突的情况了，因为实际中可能存在连续几天的大会行程中，又安排了几个小会，所以在后续的沟通中确定的方案是：**单独的连续行程进行合并，如果中间出现多个行程就不合并，如果单独的长行程还没结束，后面连续的排期还是合并**。最终的效果参考下图中的“会议111”。

<img src="../imgs/table_collapse1.png" style="zoom:30%;" />

根据表格的数据合并单元格，因为用的是ant design vue这个UI库，所以我第一时间想的就是去翻文档，查到的用法如下：

<img src="../imgs/table_collapse2.png" style="zoom:30%;" />

可是把这段代码写到项目里并没有生效，才发现最新已经是`"ant-design-vue": "^4.2.6"`，而项目里用的版本是`"ant-design-vue": "^1.6.3"`，看懵了🤧🤧🤧，查了之后才发现这个版本的使用方法是这样的：

<img src="../imgs/table_collapse3.png" style="zoom:30%;" />

于是我就按着这么写：

<img src="../imgs/table_collapse4.png" style="zoom:30%;" />

结果发现rowSpan的设置不管用，在网上搜索了一番，又自己试了几次，发现加上style的设置才实现了合并单元格。

<img src="../imgs/table_collapse5.png" style="zoom:30%;" />

很烦接手这种项目，总是用一套模板开发新项目，永远不更新三方库，大量公司的“降本增效”以后这种情况会越来越多吧，反正当下能用就行，以后维护不了了再去考虑更新三方库不知道会爆出什么问题呢😅

具体的单元格是否合并就是按照业务逻辑来判断了。在这个项目里，每日行程的原始数据结构类似如下，就是把每个领导本周内的行程给查询出来。

```json
{
    staff1: [
      {
        event: '会议111',
        startTime: '2025-01-01 9:00',
        endTime: '2025-01-04 18: 00'
      },
      {
        event: '这是一个测试会议22',
        startTime: '2025-01-02 13:00',
        endTime: '2025-01-02 16: 00'
      },
      {
        event: '这是一个测试会议33',
        startTime: '2025-01-05 09:00',
        endTime: '2025-01-05 17: 00'
      }
    ],
    staff2: [
      {
        event: '会议q',
        startTime: '2025-01-01 9:00',
        endTime: '2025-01-01 18: 00'
      },
      {
        event: '这是一个测试会议ww',
        startTime: '2025-01-02 13:00',
        endTime: '2025-01-07 16: 00'
      },
    ],
    staff3: [
      {
        event: '待办事项x',
        startTime: '2025-01-01 9:00',
        endTime: '2025-01-01 18: 00'
      },
      {
        event: '这是一个待办事项ww',
        startTime: '2025-01-05 13:00',
        endTime: '2025-01-07 16: 00'
      },
    ]
 }
```

后端会做简单的处理，把日程按单日分组，返回给前端的数据结构类似如下（项目里原本是week0~week6，本文简单演示就直接使用日期了）：

```json
{
    staff1: {
      '2025-01-01': [
        {
          event: '会议111',
          startTime: '2025-01-01 9:00',
          endTime: '2025-01-04 18: 00'
        },
      ],
      '2025-01-02': [
        {
          event: '会议111',
          startTime: '2025-01-01 9:00',
          endTime: '2025-01-04 18: 00'
        },
        {
          event: '这是一个测试会议22',
          startTime: '2025-01-02 13:00',
          endTime: '2025-01-02 16: 00'
        },
      ],
      '2025-01-03': [
        {
          event: '会议111',
          startTime: '2025-01-01 9:00',
          endTime: '2025-01-04 18: 00'
        },
      ],
      '2025-01-04': [
        {
          event: '会议111',
          startTime: '2025-01-01 9:00',
          endTime: '2025-01-04 18: 00'
        },
      ],
      '2025-01-05': [
        {
          event: '这是一个测试会议33',
          startTime: '2025-01-05 09:00',
          endTime: '2025-01-05 17: 00'
        }
      ],
      '2025-01-06': [],
      '2025-01-07': [],
    },
    staff2: {
      '2025-01-01': [
        {
          event: '会议q',
          startTime: '2025-01-01 9:00',
          endTime: '2025-01-01 18: 00'
        },
      ],
      '2025-01-02': [
        {
          event: '这是一个测试会议ww',
          startTime: '2025-01-02 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
      '2025-01-03': [
        {
          event: '这是一个测试会议ww',
          startTime: '2025-01-02 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
      '2025-01-04': [
        {
          event: '这是一个测试会议ww',
          startTime: '2025-01-02 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
      '2025-01-05': [
        {
          event: '这是一个测试会议ww',
          startTime: '2025-01-02 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
      '2025-01-06': [
        {
          event: '这是一个测试会议ww',
          startTime: '2025-01-02 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
      '2025-01-07': [
        {
          event: '这是一个测试会议ww',
          startTime: '2025-01-02 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
    },
    staff3: {
      '2025-01-01': [
        {
          event: '待办事项x',
          startTime: '2025-01-01 9:00',
          endTime: '2025-01-01 18: 00'
        },
      ],
      '2025-01-02': [],
      '2025-01-03': [],
      '2025-01-04': [],
      '2025-01-05': [
        {
          event: '这是一个待办事项ww',
          startTime: '2025-01-05 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
      '2025-01-06': [
        {
          event: '这是一个待办事项ww',
          startTime: '2025-01-05 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
      '2025-01-07': [
        {
          event: '这是一个待办事项ww',
          startTime: '2025-01-05 13:00',
          endTime: '2025-01-07 16: 00'
        },
      ],
    },
}
```

前端就在以上的结构基础上进行遍历处理。

第一步准备工作，先简单判断当前处理的行程是否在一天内结束，并且判断是否跨时段（上下午），把这个两个判断结果存储起来用于后续操作。

```ts
const inOneDay =
    moment(schedule.endTime).format('YYYY-MM-DD') ===
    moment(schedule.startTime).format('YYYY-MM-DD') // 是否在一天内完成（开始日期和结束日期一致）
let inOneRange = false // 是否在同个时段（上下午），判断一天内的日程是否跨时段
if (inOneDay) {
  const startMorning = moment(schedule.startTime).isSameOrBefore(
      weekData[weekIndex].dateStr + ' ' + MORNING_END
  )
  const endAfternoon = moment(schedule.endTime).isSameOrAfter(
      weekData[weekIndex].dateStr + ' ' + AFTERNOON_START
  )
  if ((startMorning && !endAfternoon) || (!startMorning && endAfternoon)) inOneRange = true
}
```

第二步就在第一步的基础上先做第一轮简单的筛选，如果满足以下条件之一，则当前处理的行程不用跨行处理。

1. 当前行程所在时段存在多个行程
2. 当前行程本身不跨时段
3. 当前行程跨上下午时段，当前处理的是下午，但是上午存在多个行程

```ts
if (
      weekData[weekIndex][account].length > 1 || // 当前员工单个时段有多个行程
      (inOneDay && inOneRange) || // 某行程不跨时段
      (inOneDay &&
          !inOneRange &&
          weekIndex % 2 === 1 &&
          weekData[weekIndex - 1][account].length > 1) // 当前行程跨上下午时段，当前处理的是下午，但是上午存在多个行程
  ) {
    // 不做跨行处理
    result.isCross = false
    return result
}
```

第三步做第二轮筛选，首先做两个判断并保存判断结果。

1. 当前是否为跨行的开始行

   ```ts
   // 判断是否是跨行的开始（满足条件之一）：
   // 1. 行程的开始日期等于当前行的日期，行程的开始时间晚于等于当前行的startTime
   // 2. 行程的开始日期等于当前行的日期，行程的结束日期晚于当前行的日期
   // 3. 行程的开始日期早于当前行的日期，且前一行的行程数量大于1
   // 4. 当前行程在第一行
   const isStart =
       (scheduleStartDate === weekData[weekIndex].dateStr &&
           scheduleStartTime >= weekData[weekIndex].startTime) ||
       (weekIndex % 2 === 1 &&
           weekData[weekIndex - 1][account].length > 1 &&
           scheduleStartDate === weekData[weekIndex].dateStr &&
           scheduleEndDate >= weekData[weekIndex].dateStr) ||
       (scheduleStartDate < weekData[weekIndex].dateStr &&
           weekIndex > 0 &&
           weekData[weekIndex - 1][account].length > 1) ||
       weekIndex === 0
   ```

2. 当前是否为跨行的结束行

   ```ts
   // 判断是否是跨行的结束（满足条件之一）:
   // 1. 当前行程在最后一行
   // 2. 行程的结束日期等于当前行的日期，行程的结束时间晚于当前行的startTime且早于等于当前行的endTime
   // 3. 下一行的日程数量大于1
   const isEnd =
       weekIndex === 13 ||
       (scheduleEndDate === weekData[weekIndex].dateStr &&
           scheduleEndTime >= weekData[weekIndex].startTime &&
           scheduleEndTime <= weekData[weekIndex].endTime) ||
       weekData[weekIndex + 1][account].length > 1
   ```

如果两个判断结果都为true，则说明既是开始行，同是又是结束行，那就不用做跨行处理。

最后筛出来的就是要跨行的单元格了，就要计算跨的行数了，也就是起始行的rowSpan值，非起始行的rowSpan就是0了。

起始行的rowSpan就是计算具体这个行程在表格里跨的行数。

首先计算单个行程自身原本跨了几个时段。

```ts
const diffScheduleEnd = moment(scheduleEndDate).diff(
    moment(weekData[weekIndex].dateStr),
    'days'
) // 与行程结束日期的天数差值
const diffWeekEnd = moment(weekData[13].dateStr).diff(
    moment(weekData[weekIndex].dateStr),
    'days'
) // 与周最后一天的天数差值
const dayOff = Math.min(diffScheduleEnd, diffWeekEnd) // 跨的天数
const timeOff = scheduleEndTime <= MORNING_END ? 1 : 2 // 跨的时段
let offRows = 0
// 行位移 = (天数-1)*2 + 跨的时段
if (dayOff > 0) offRows = (dayOff - 1) * 2 + timeOff
// 如果当前行程是上午开始的，再加一个行跨
if (weekIndex % 2 === 0) offRows++
```

再向后遍历碰到存在多个行程的单元格就表示跨行结束，得到了rowSpan的值。

```ts
const len = weekIndex + 1 + offRows
let rowSpan = 1
for (let i = weekIndex + 1; i < len; i++) {
  if (weekData[i][account].length > 1) {
    break
  } else {
    rowSpan++
  }
}
```

最后我们就可以得到合并的单元格。

