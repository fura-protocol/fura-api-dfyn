# Dfyn History

Check it out live: [https://demo.furadao.org/subgraph/polygon/dfyn](https://demo.furadao.org/subgraph/polygon/dfyn)


### Fura - Dfyn GraphQL Endpoint
https://polygon.furadao.org/subgraphs/name/dfyn


### Code Configuration
change [src/apollo/client.js](https://github.com/Uniswap/uniswap-info/blob/v2/src/apollo/client.js) as follows:

```
export const client = new ApolloClient({
  link: new HttpLink({
    uri: 'https://polygon.furadao.org/subgraphs/name/dfyn',
  }),
  cache: new InMemoryCache(),
  shouldBatch: true,
})

export const healthClient = new ApolloClient({
  link: new HttpLink({
    uri: 'https://polygon.furadao.org/subgraphs/name/dfyn',
  }),
  cache: new InMemoryCache(),
  shouldBatch: true,
})

export const blockClient = new ApolloClient({
  link: new HttpLink({
    uri: 'https://polygon.furadao.org/subgraphs/name/dfyn',
  }),
  cache: new InMemoryCache(),
})
```

### CORS Settings
https://polygon.furadao.org/subgraphs/name/dfyn 

now supports:
```
localhost:3000
localhost:9528
127.0.0.1:3000
127.0.0.1:9528
info.dfyn.network
```

### Use Playground
https://github.com/graphql/graphql-playground

# Dfyn Index Implementation

### Overall Topology

```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
Properties prop = new Properties();
prop.put("bootstrap.servers","*:9092");
prop.put("zookeeper.connect","*:2181");
prop.put("group.id",groupId);
prop.put("enable.auto.commit","true");
prop.put("auto.offset.reset","earliest");

FlinkKafkaConsumer010<String> consumer = new FlinkKafkaConsumer010<>("quick_tx_data", new SimpleStringSchema(), prop);
consumer.setStartFromGroupOffsets();

SingleOutputStreamOperator<Tuple1<TxEventData>> stream = env.addSource(consumer).setParallelism(16)
        .flatMap(new PairCreateProcessBolt()).name("paircreater").setParallelism(parallelism);

stream.addSink(new PairCreateSinkBolt()).name("sink-tokafka").setParallelism(parallelism);

env.execute(jobName);
```

### Paircreater Operator

```
public class PairCreateProcessBolt extends RichFlatMapFunction<String, Tuple1<TxEventData>> {
    private static final Logger LOG = LoggerFactory.getLogger(PairCreateHandler.class);

    private PairCreateHandler pairCreateHandler;
    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        pairCreateHandler = new PairCreateHandler();
    }

    @Override
    public void flatMap(String rawlog, Collector<Tuple1<TxEventData>> collector) throws Exception {
        try {
            long start = System.currentTimeMillis();
            CustomTransactionObject customTransactionObject = JSON.parseObject(rawlog, CustomTransactionObject.class);

            TxEventData txEventData = pairCreateHandler.handleCustomTransaction(customTransactionObject);
            if (txEventData != null) {
                collector.collect(new Tuple1<TxEventData>(txEventData));
            }
            long usetime = System.currentTimeMillis() - start;
            LOG.info("use time:{}",usetime);

        }catch (Exception ex) {
            LOG.error("PairCreateProcessBolt execute failed:{}", rawlog, ex);
        }
    }

}
```

### sink-tokafka

```
public class PairCreateSinkBolt extends RichSinkFunction<Tuple1<TxEventData>> {
    private static final Logger LOG = LoggerFactory.getLogger(PairCreateSinkBolt.class);

    private KafkaClientManager kafkaClientManager;
    @Override
    public void open(Configuration parameters) throws Exception {
        Config rootConfig = ConfigFactory.load("kafka.conf");
        Config kafkaConfig = rootConfig.getConfig("kafka-server");
        kafkaClientManager = KafkaClientManager.getInstance(kafkaConfig);
        super.open(parameters);
    }

    @Override
    public void invoke(Tuple1<TxEventData> tuple, Context context) throws Exception {
        TxEventData txEventData = tuple.f0;
        String txhash = txEventData.getHash();
        String json = JSON.toJSONString(txEventData);
        kafkaClientManager.send(json, txhash);
    }

}
```
