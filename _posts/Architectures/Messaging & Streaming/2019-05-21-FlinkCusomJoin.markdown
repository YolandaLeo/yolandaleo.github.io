---
layout: post
categories: 'development'
tag: flink
title: "Flink Custom Stream Join"
---

In this notes, I'll instroduce several Window experiments in Flink, then show my POC sample code on how to solve an two stream join with a dense input and a sparse one.
The following experiments are based on SocketWordCount job, with text as input stream.
<!--more-->

##SlidingWindow Experiment
SlidingWindow(Time.second(30), Time.second(5))
1. When event comes, Flink window assigns TimeWindows for each slide with normalized time range.
2. Trigger registers timer for each time window.
3. Timer triggers Trigger to clear timer for this window.
4. Trigger will not be processed when no events are in the window.

In the following experiment, E1 and E2 are events of the same slide and create pink TimeWindows. E3 comes in the second slide, then creates blue TimeWindows.

![avatar]({{ site.baseurl }}/img/FlinkSlidingWindow.png)


##TumblingWindow Experiment
E1 
assignWindows in CustomTumblingWindow: 1906076756
E2
assignWindows in CustomTumblingWindow: 1906076756
 => aaa : 2
E3
assignWindows in CustomTumblingWindow: 1906076756
E4
assignWindows in CustomTumblingWindow: 1906076756
E5
assignWindows in CustomTumblingWindow: 1906076756
 => aaa : 3


##GlobalWindow Experiment
GlobalWindow need custom trigger implementation, or the event stream will never be processed.
This is good scenario to show the difference between TriggerResult.FIRE and TriggerResult.FIRE_AND_PURGE.
```java
@Override
    public TriggerResult onElement(Object element, long timestamp, GlobalWindow window, TriggerContext ctx) throws Exception {
        System.out.println("CustomGlobalTrigger onElement, ctx = " + ctx);
        ctx.registerProcessingTimeTimer(System.currentTimeMillis() + 5000);
        return TriggerResult.FIRE;
    }

    @Override
    public TriggerResult onProcessingTime(long time, GlobalWindow window, TriggerContext ctx) throws Exception {
        return TriggerResult.FIRE;
    }
```
With input stream kept being ‘aaa’, the output shown in the following sequence indicaties PURGE happened and the value has been updated every 5 seconds.

*CustomGlobalTrigger onElement, ctx = Context{key=(aaa), window=GlobalWindow}*
*aaa : 1, lastModified = 1557821994712*
*aaa : 1, lastModified = 1557821994712*
*CustomGlobalTrigger onElement, ctx = Context{key=(aaa), window=GlobalWindow}*
*aaa : 1, lastModified = 1557822188496*
*CustomGlobalTrigger onElement, ctx = Context{key=(aaa), window=GlobalWindow}*
*aaa : 2, lastModified = 1557822189179*
*aaa : 2, lastModified = 1557822189179*
*CustomGlobalTrigger onElement, ctx = Context{key=(aaa), window=GlobalWindow}*
*aaa : 1, lastModified = 1557822198132*
*aaa : 1, lastModified = 1557822198132* 

After change onProcessingTime trigger to TriggerResult.FIRE, the output sequence just kept increasing without purging.

*CustomGlobalTrigger onElement, ctx = Context{key=(aaa), window=GlobalWindow}
aaa : 1, lastModified = 1557822344991
aaa : 1, lastModified = 1557822344991
CustomGlobalTrigger onElement, ctx = Context{key=(aaa), window=GlobalWindow}
aaa : 2, lastModified = 1557822350939
aaa : 2, lastModified = 1557822350939
CustomGlobalTrigger onElement, ctx = Context{key=(aaa), window=GlobalWindow}
aaa : 3, lastModified = 1557822357193
CustomGlobalTrigger onElement, ctx = Context{key=(aaa), window=GlobalWindow}
aaa : 4, lastModified = 1557822361817
aaa : 4, lastModified = 1557822361817*

##Custom Two Stream Join with Queryable State
Queryable State is an experimental feature in Flink_1.8.0 ([reference doc](https://ci.apache.org/projects/flink/flink-docs-stable/dev/stream/state/queryable_state.html))

I built a server for producing dense input stream, and a client for producing the sparse stream and join with dense stream to trigger data sink.

Producer FoldingFunction
```java
public class AddInstrumentFoldingFunction implements FoldFunction<AddInstrumentEvent, FraudUnifiedState> {
    //private MapStateDescriptor<String, List<AddInstrumentEvent>> stateDescriptor;
    //private transient MapState<String, List<AddInstrumentEvent>> state;
    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd hh:mm:ss");
    /*@Override
    public void open(Configuration parameters) {
        stateDescriptor = new MapStateDescriptor<>("add-state", BasicTypeInfo.STRING_TYPE_INFO, TypeInformation.of(new TypeHint<List<AddInstrumentEvent>>() {
        }));
        state = getRuntimeContext().getMapState(stateDescriptor);
    }*/

    @Override
    public FraudUnifiedState fold(FraudUnifiedState accumulator, AddInstrumentEvent event) throws Exception {
        //List<AddInstrumentEvent> addList = accumulator.get(value.getInstrumentId());
        //accumulator = new FraudUnifiedState();
        /**
         * <userId, <StartTime in String, Tuple2<addCnt, suspCnt>>>
         */
        Map<String, Map<String, Tuple2<Integer, Integer>>> userMap
                = accumulator.getInnerState() == null ? new HashMap<>() : accumulator.getInnerState();

        String start = normalizeWindowToHour(event.getEventTime());
        Map<String, Tuple2<Integer, Integer>> windowMap = userMap.get(event.getUserId());
        if(windowMap == null) {
            windowMap = new HashMap<>();
            windowMap.put(start, Tuple2.of(1, 0));
        } else {
            Integer addCnt = windowMap.get(start) == null ? 0 : windowMap.get(start).f0;
            windowMap.put(start, Tuple2.of(++addCnt, 0));
        }
        userMap.put(event.getUserId(), windowMap);
        accumulator = new FraudUnifiedState();
        accumulator.setInstrumentId(event.getInstrumentId());
        accumulator.setInnerState(userMap);
        return accumulator;
    }
    private String normalizeWindowToHour(Date date) {
        LocalDateTime ldt = LocalDateTime.ofInstant(date.toInstant(), ZoneId.of("America/Los_Angeles"));
        ldt = ldt.truncatedTo(ChronoUnit.HOURS);
        return ldt.format(formatter);
    }
}
```

Client ProcessFunction
```java
public class SuspendMapFunction extends ProcessFunction<UserSuspendEvent, FraudUnifiedState> {
    private static final String queryableStateName = "dense-input";
    private static final Integer TIME_WINDOW_DAYS = 3;
    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd hh:mm:ss");

    private QueryableStateClient client;
    private FoldingStateDescriptor<AddInstrumentEvent, FraudUnifiedState> descriptor;
    private JobID jobId = JobID.fromHexString("13633222c0eb247f37aa1e95c3a8d896");
    private MapStateDescriptor<String, Tuple2<Integer, Integer>> stateDescriptor =
            new MapStateDescriptor<>("value-map",
            BasicTypeInfo.STRING_TYPE_INFO,
            TypeInformation.of(new TypeHint<Tuple2<Integer, Integer>>(){}));
    private MapState<String, Tuple2<Integer, Integer>> state;

    @Override
    public void open(Configuration parameters) throws UnknownHostException {
        client = new QueryableStateClient("10.249.74.134", 9069);
        client.setExecutionConfig(getRuntimeContext().getExecutionConfig().enableClosureCleaner());

        FraudUnifiedState initialState = new FraudUnifiedState();
        descriptor =
                new FoldingStateDescriptor<>(
                        "dense-input",
                        initialState,
                        new AddInstrumentFoldingFunction(),
                        TypeInformation.of(new TypeHint<FraudUnifiedState>() {
                        }));
        state = getRuntimeContext().getMapState(stateDescriptor);
    }

    @Override
    public void processElement(UserSuspendEvent value, Context ctx, Collector<FraudUnifiedState> out) throws Exception {
        CompletableFuture<FoldingState<AddInstrumentEvent, FraudUnifiedState>> resultFuture =
                client.getKvState(jobId, queryableStateName, value.getInstrumentId(), BasicTypeInfo.STRING_TYPE_INFO, descriptor);
        try {
            FoldingState<AddInstrumentEvent, FraudUnifiedState> foldingState = resultFuture.get();
            FraudUnifiedState unifiedState = foldingState.get();
            int addCnt= 0;
            Map<String, Tuple2<Integer, Integer>> windowMap = unifiedState.getInnerState().get(value.getUserId());
            if(windowMap != null) {
                List<String> windowKeys = buildWindowKeys(value.getEventTime());
                for(String key : windowKeys) {
                    Tuple2<Integer, Integer> cntTuple = windowMap.get(key);
                    if(cntTuple != null) {
                        cntTuple.f1++;
                    }
                }
            } else {
                System.out.println("UnifiedState with unmatched userMap, query result: " + unifiedState.toString() + ", \n\tUserSuspendEvent: " + value.toString());
                return;
            }


            Tuple2<Integer, Integer> stateValue = state.get(value.getInstrumentId()) == null ? new Tuple2<>(0, 0) : state.get(value.getInstrumentId());
            stateValue.f0 = addCnt;
            stateValue.f1++;
            state.put(value.getInstrumentId(), stateValue);
            out.collect(unifiedState);
        } catch (UnknownKeyOrNamespaceException |ExecutionException ex) {
            System.out.println("No state for the specified key/namespace: " + value.toString());
        }

    }
    private List<String> buildWindowKeys(Date date) {
        LocalDateTime ldt = LocalDateTime.ofInstant(date.toInstant(), ZoneId.of("America/Los_Angeles"));
        ldt = ldt.truncatedTo(ChronoUnit.HOURS);
        LocalDateTime begin = ldt;
        begin = begin.minusDays(TIME_WINDOW_DAYS);
        List<String> result = new ArrayList<>();
        do {
            result.add(begin.format(formatter));
            begin = begin.plusHours(1);
        }
        while (begin.isBefore(ldt));
        return result;
    }
}
```