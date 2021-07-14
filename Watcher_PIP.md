* **Status**: Proposal
* **Author**: Will McCarley
* **Pull Request**: (Link to the main pull request to resolve this PIP)
* **Mailing List discussion**:
* **Release**:

### Motivation

_Some clients need the ability to programmatically monitor a given topic's conumers/subscriptions for certain events. This can be accomplished to a limited degree by using the available REST APIs and published stats (e.g. Prometheus.) However this solution requires the administrator of the system to provide access to these APIs and any monitoring solution will be based on polling rather than near-realtime alerting._

_The purpose of this PIP is to introduce a way for clients to programmatically attach 'Watchers' to topics to observe the behavior of the connected consumers & producers._

_One example is to create an out-of-band application that 'watches' a given subscription and scales the consumer application horizontally when the backlog becomes unacceptably high._


### Public Interfaces

_Briefly list any new interfaces that will be introduced as part of this proposal or any existing interfaces that will be removed or changed. The purpose of this section is to concisely call out the public contract that will come along with this feature._

Proposing the following changes:

- Additional wire protocol commands:
    - CommandWatch
    - CommandWatchSuccess
    - CommandPauseWatch
    - CommandPauseWatchSuccess
    - CommandResumeWatch
    - CommandResumeWatchSuccess
    - CommandUnWatch
    - CommandUnWatchSuccess
    - CommandWatchEvent
        - ConsumerConnect
        - ConsumerDisconnect
        - ConsumerStuck
        - ConsumerUnstuck
        - ProducerConnect
        - ProducerDisconnect
        - ProducerIdle
        - SubscriptionCreate
        - SubscriptionBacklog
        - SubscriptionCatchUp
        - SubscriptionIdle
        - SubscriptionUnsubscribe
        - TopicBacklogEviction
        - TopicIdle
        - TopicPartitionCountChange
        - TopicSchemaAdd
        - TopicSchemaDelete
        - TopicSchemaUpdate
        - TopicTermination
        - WatcherConnected
        - WatcherPaused
        - WatcherUnPaused
        - WatcherDisconnected
- Introduce new interfaces to the Pulsar API:
    - WatcherBuilder
    - Watcher
    - WatchEventListener
- New configuration options in broker.conf:
    - enableWatchers // Whether the broker will allow watchers to connect
    - defaultWatcherConsumerStuckPeriodMillis // If a consumer stays at zero permits for more than this period of time the ConsumerStuck event will be fired
    - defaultWatcherSubscriptionIdleGracePeriodMillis // default grace period before subscription is considered 'idle' to prevent premature invocation of onSubscriptionIdle(...) callback when all consumer instances disconnect and re-connect
    - defaultWatcherSubscriptionBacklogGracePeriodMillis // default grace period to prevent excessive SubscriptionBacklog and SubscriptionCatchUp events
    - defaultWatcherSubscriptionBacklogGraceMessageCount // same as above but quantity of messages allowed in backlog
    - watcherSubscriptionCheckIntervalMillis // how often to check for Subscription_xx events
- Create several new 'watch' permission separate from 'produce' & 'consume' permissions
    - watch_subscriptions(_sub name regex_)
    - watch_consumers(_role regex_)
    - watch_producers(_role regex_)
    - watch_watchers(_role regex_)
    - watch_topic
- Modify PulsarAdmin commands:
    - stats command  -> show information about attached Watchers
    - grant/revoke_permission -> allow granting/revoking the new 'watch_xx' permissions
    
- New PulsarAdmin commands

Example of usage:

```java
PulsarClient client = PulsarClient.builder()
    .serviceUrl("pulsar+ssl://broker:6651")
    .build();
                              
Watcher watcher = client.newWatcher()
    .topic("persistent://tenant/ns/topic")
    
    .watchProducers("role-1")
    
    .watchSubscriptions("mysub-.*")
    
    .watchConsumers("role-1")
    
    // this is to prevent excessive invocation of onSubscriptionBacklog(...) callback
    .subscriptionBacklogGracePeriodMillis(30000)
    .subscriptionBacklogGraceMessageCount(5000)
    
    // grace period before subscription is considered 'idle' for this watcher
    // this is to prevent premature invocation of onSubscriptionIdle(...) callback
    // when all consumer instances disconnect and re-connect
    .subscriptionIdleGracePeriodMillis(30000)
    
    // grace period before topic is considered 'idle' for this watcher
    // this is to prevent unneccessary invocation of onTopicIdle(...) callback
    .topicIdleGracePeriodMillis(30000)
    
    // implementation of the WatchEventListener interface
    .eventListener(myWatchEventListener)
    .watch()
    
...

watcher.close();
```


### Proposed Changes

_Describe the new thing you want to do in appropriate detail. This may be fairly extensive and have large subsections of its own. Or it may be a few sentences. Use judgment based on the scope of the change._

#### Protobuf Changes ####

```protobuf
message CommandWatch {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
    required string topic = 3;
    optional bool watch_producers = 4 [default = false];
    optional string watch_producer_role = 5;
    optional bool watch_consumers = 6 [default = false];
    optional string watch_consumer_role = 7;
    optional bool watch_subscriptions = 8 [default = false];
    optional string watch_subscription_name = 9;
    optional bool watch_watchers = 10 [default = false];
    optional string watch_watchers_role = 11;
    optional string watcher_name = 12;
    repeated KeyValue metadata = 13
}

message CommandWatchSuccess {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
    required string watcher_name = 3;
}

message CommandPauseWatch {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
}

message CommandPauseWatchSuccess {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
    required uint64 pause_time = 3;
}

message CommandResumeWatch {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
}

message CommandResumeWatchSuccess {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
    required uint64 resume_time = 3;
}

message CommandUnWatch {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
}

message CommandWatchEventConsumerConnect {
}

message CommandWatchEventConsumerDisconnect {
}

message CommandWatchEventConsumerStuck {
}

message CommandWatchEventConsumerUnstuck {
}

message BaseCommand {
    enum Type {
        ...
        WATCH_EVENT_CONSUMER_CONNECT = 90;
    }
    
    ...
    
    optional CommandWatchEventConsumerConnect watchEventConsumerConnect = 90;
```

#### Pulsar API Changes ####
```java
public interface WatchEventListener {

    public void onConsumerConnect(String topic, String subscription, ConsumerCnx cnx);
    
    public void onConsumerStuck(String topic, String subscription, ConsumerCnx cnx);
    
    public void onConsumerUnstuck(String topic, String subscription, ConsumerCnx cnx);
    
    public void onConsumerStats(String topic, String subscription, ServerConsumerStats[] stats);
    
    public void onConsumerDisconnect(String topic, String subscription, ConsumerCnx cnx);
    
    public void onProducerConnect(String topic, ProducerCnx cnx);
    
    public void onProducerStats(String topic, ServerProducerStats[] stats);
    
    public void onProducerDisconnect(String topic, ProducerCnx cnx);
    
    public void onProducerIdle(String topic, ProducerCnx cnx, long lastActiveTime);
    
    public void onSubscriptionCreate(String topic, String subscription, SubscriptionType type, SubscriptionInitialPosition position);
    
    public void onSubscriptionBacklog(String topic, String subscription, long backlogSize);
    
    public void onSubscriptionCatchUp(String topic, String subscription);
    
    public void onSubscriptionSeek(String topic, String subscription, long oldPosition, long newPosition);

    public void onSubscriptionIdle(String topic, String subscription);

    public void onUnsubscribe(String topic, String subscription);

    public void onTopicBacklogEviction(String topic, String[] affectedSubscriptions);
    
    public void onTopicIdle(String topic);
    
    public void onTopicPartitionCountChange(String topic, int oldPartitionCount, int newPartitionCount);
    
    public void onTopicSchemaAdd();
    
    public void onTopicSchemaDelete();
    
    public void onTopicSchemaUpdate();
    
    public void onTopicTermination(String topic);
    
    public void onWatcherConnected(String topic, WatcherInfo watcherInfo);
    
    public void onWatcherPaused(String topic, WatcherInfo watcherInfo);
    
    public void onWatcherDisconnected(String topic, WatcherInfo watcherInfo);
}
```

### Compatibility, Deprecation, and Migration Plan

- What impact (if any) will there be on existing users? 
- If we are changing behavior how will we phase out the older behavior? 
- If we need special migration tools, describe them here.
- When will we remove the existing behavior?

### Test Plan

_Describe in few sentences how the BP will be tested. We are mostly interested in system tests (since unit-tests are specific to implementation details). How will we know that the implementation works as expected? How will we know nothing broke?_

### Rejected Alternatives

_If there are alternative ways of accomplishing the same thing, what were they? The purpose of this section is to motivate why the design is the way it is and not some other way._
