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
        - ConsumerConnectionEvent
        - ProducerConnectionEvent
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

Example of watcher usage:

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
message ConsumerConnectionMetadata {
    required uint64 consumer_id = 1;
    required uint64 request_id  = 2;
    optional string consumer_name = 3;
    optional int32 priority_level = 4;
    repeated KeyValue metadata = 5;
    optional string address = 6;
    optional string connectedSince = 7;
}

message ProducerConnectionMetadata {
    required uint64 producer_id = 1;
    required uint64 request_id  = 2;
    optional string producer_name = 3;
    repeated KeyValue metadata = 5;
    optional string address = 6;
    optional string connectedSince = 7;
}

message WatcherConnectionMetadata {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
    optional string watcher_name = 3;
    repeated KeyValue metadata = 5;
    optional bool watch_producers = 6 [default = false];
    optional string watch_producer_role = 7;
    optional bool watch_consumers = 8 [default = false];
    optional string watch_consumer_role = 9;
    optional bool watch_subscriptions = 10 [default = false];
    optional string watch_subscription_name = 11;
    optional bool watch_watchers = 12 [default = false];
    optional string watch_watchers_role = 13;
    optional string address = 14;
    optional string connectedSince = 15;
}

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
    repeated KeyValue metadata = 13;
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

message CommandUnwatch {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
}

message CommandUnwatchSuccess {
    required uint64 watcher_id = 1;
    required uint64 request_id  = 2;
    required uint64 disconnect_time = 3;
}

message CommandWatchEventConsumerLifecycle {
    enum ConsumerLifecycleType {
        Connect = 0;
        Disconnect = 1;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required ConsumerLifecycleType type = 4;
    required ConsumerConnectionMetadata conumerMetadata = 5;
}

message CommandWatchEventConsumerActivity {
    enum ConsumerActivityType {
        Stuck = 1;
        Unstuck = 2;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required string subscription = 5;
    required ConsumerActivityType type = 6;
    required ConsumerConnectionMetadata conumerMetadata = 7;
    required uint64 permit_count = 8;
    required uint64 last_ack_timestamp = 9;
}

message CommandWatchEventProducerLifecycle {
    enum ProducerLifecycleType {
        Connect = 0;
        Disconnect = 1;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required ProducerLifecycleType type = 5;
    required ProducerConnectionMetadata producerMetadata = 6;
}

message CommandWatchEventProducerActivity {
    enum ProducerActivityType {
        Idle = 1;
        Active = 2;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required ProducerActivityType type = 5;
    required ProducerConnectionMetadata producerMetadata = 6;
    optional uint64 last_send_timestamp = 7;
}

message CommandWatchEventSubscriptionLifecycle {
    enum SubscriptionLifecycleType {
        Create = 0;
        Delete = 1;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required string subscription = 5;
    required SubscriptionLifecycleType type = 6;
    optional InitialPosition initial_position = 7;
}

message CommandWatchEventSubscriptionActivity {
    enum SubscriptionActivityType {
        Backlog = 0;
        CatchUp = 1;
        Idle = 2;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required string subscription = 5;
}

message CommandWatchEventSubscriptionSeek {
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required string subscription = 5;
    optional MessageIdData message_id = 3;
    optional uint64 message_publish_time = 4;
}

message CommandWatchEventTopicLifecycle {
    enum TopicLifecycleEventType {
        Create = 0;
        Terminate = 1;
        Delete = 2;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required TopicLifecycleType type = 5;
}

message CommandWatchEventTopicActivity {
    enum TopicActivityType {
        Active = 0;
        Idle = 1;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required TopicActivityType type = 5;
}

message CommandWatchEventTopicPartitionCountChange {
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    optional uint32 old_partition_count = 5;
    required uint32 new_partition_count = 6;
}

message CommandWatchEventTopicPolicyEnforcement {
    enum TopicPolicyEnforcementType {
        BacklogEviction = 0;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required TopicLifecycleType type = 5;
}

message CommandWatchEventTopicSchemaModify {
    enum TopicSchemaModifyEventType {
        Update = 0;
        Delete = 1;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required TopicSchemaModifyEventType type = 5;
    optional Schema old_schema = 6;
    optional Schema new_schema = 7;
}

message CommandWatchEventWatcherLifecycle {
    enum WatcherLifecycleType {
        Connect = 0;
        Disconnect = 1;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required WatcherLifecycleType type = 5;
    required WatcherConnectionMetadata watcherMetadata = 6;
}

message CommandWatchEventWatcherActivity {
    enum WatcherActivityType {
        Pause = 1;
        Resume = 2;
    }
    required uint64 watcher_id = 1;
    required uint64 event_id  = 2;
    required uint64 event_time = 3;
    required string topic = 4;
    required WatcherActivityType type = 5;
    required WatcherConnectionMetadata watcherMetadata = 6;
    optional uint64 last_watch_event_timestamp = 7;
}

message BaseCommand {
    enum Type {
        ...
        WATCH = 90;
        WATCH_SUCCESS = 91;
        PAUSE_WATCH = 92;
        PAUSE_WATCH_SUCCESS = 93;
        RESUME_WATCH = 94;
        RESUME_WATCH_SUCCESS = 95;
        UNWATCH = 96;
        UNWATCH_SUCCESS = 97;
        WATCH_EVENT_CONSUMER_LIFECYCLE = 98;
        WATCH_EVENT_CONSUMER_ACTIVITY = 99;

        WATCH_EVENT_PRODUCER_LIFECYCLE = 100;
        WATCH_EVENT_PRODUCER_ACTIVITY = 101;

        WATCH_EVENT_SUBSCRIPTION_LIFECYCLE = 102;
        WATCH_EVENT_SUBSCRIPTION_ACTIVITY = 103;
        WATCH_EVENT_SUBSCRIPTION_SEEK = 104;

        WATCH_EVENT_TOPIC_LIFECYCLE = 105;
        WATCH_EVENT_TOPIC_ACTIVITY = 106;
        WATCH_EVENT_TOPIC_PARTITION_COUNT_CHANGE = 107;
        WATCH_EVENT_TOPIC_POLICY_ENFORCEMENT = 108;
        
        WATCH_EVENT_TOPIC_SCHEMA_MODIFY = 109;

        WATCH_EVENT_WATCHER_LIFECYCLE = 110;
        WATCH_EVENT_WATCHER_ACTIVITY = 111;
    }
    
    ...
    optional CommandWatch watch = 90;
    optional CommandWatchSuccess watchSuccess = 91;
    optional CommandPauseWatch pauseWatch = 92;
    optional CommandPauseWatchSuccess pauseWatchSuccess = 93;
    optional CommandResumeWatch resumeWatch = 94;
    optional CommandResumeWatchSuccess resumeWatchSuccess = 95;
    optional CommandUnwatch unwatch = 96;
    optional CommandUnwatchSuccess unwatchSuccess = 97;
    optional CommandWatchEventConsumerLifecycle watchEventConsumerLifecycle = 98;
    optional CommandWatchEventConsumerActivity watchEventConsumerActivity = 99;
    optional CommandWatchEventProducerLifecycle watchEventProducerLifecycle = 100;
    optional CommandWatchEventProducerActivity watchEventProducerActivity = 101;
    optional CommandWatchEventSubscriptionLifecycle watchEventSubscriptionLifecycle = 102;
    optional CommandWatchEventSubscriptionActivity watchEventSubscriptionActivity = 103;
    optional CommandWatchEventSubscriptionSeek watchEventSubscriptionSeek = 104;
    optional CommandWatchEventTopicLifecycle watchEventTopicLifecycle = 105;
    optional CommandWatchEventTopicActivity watchEventTopicActivity = 106;
    optional CommandWatchEventTopicPartitionCountChange watchEventTopicPartitionCountChange = 107;
    optional CommandWatchEventTopicPolicyEnforcement watchEventTopicPolicyEnforcement = 108;
    optional CommandWatchEventTopicSchemaModify watchEventTopicSchemaModify = 109;
    optional CommandWatchEventWatcherLifecycle watchEventWatcherLifecycle = 110;
    optional CommandWatchEventWatcherActivity watchEventWatcherActivity = 111;
```

#### Pulsar API Changes ####
```java
/**
  * <p>The primary interface that must be implemented to use the Watcher functionality.
  * 
  */
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
    
    public void onTopicSchemaUpdate(Schema oldSchema, Schema newSchema);
    
    public void onTopicTermination(String topic);
    
    public void onWatcherConnected(String topic, WatcherInfo watcherInfo);
    
    public void onWatcherPaused(String topic, WatcherInfo watcherInfo);
    
    public void onWatcherDisconnected(String topic, WatcherInfo watcherInfo);
}
```

#### broker.conf Settings ####

```ini
# Whether the broker will allow watchers to connect
enableWatchers = false

# If a consumer stays at zero permits for more than this period of time the
# ConsumerStuck event will be fired
defaultWatcherConsumerStuckPeriodMillis = 5000

# How frequently the broker should resend ConsumerStuck events, -1 disables
# resending the events
defaultWatcherConsumerStuckResendPeriod = 3000

# default grace period before subscription is considered 'idle' Set this to a
# value large enough to prevent premature invocation of onSubscriptionIdle(...)
# callback when all consumer instances disconnect and re-connect (for instance
# when the topic moves to another broker)
defaultWatcherSubscriptionIdleGracePeriodMillis = 10000

# Default grace period to prevent excessive SubscriptionBacklog and
# SubscriptionCatchUp events
defaultWatcherSubscriptionBacklogGracePeriodMillis = 10000

# Default backlog message count that is considered 'normal' we should not
# fire SubscriptionBacklog events
defaultWatcherSubscriptionBacklogGraceMessageCount = 2000

# How often to check for Subscription_xx events
watcherSubscriptionCheckIntervalMillis = 3000
```

### Compatibility, Deprecation, and Migration Plan

- What impact (if any) will there be on existing users? 
- If we are changing behavior how will we phase out the older behavior? 
- If we need special migration tools, describe them here.
- When will we remove the existing behavior?

### Test Plan

_Describe in few sentences how the PIP will be tested. We are mostly interested in system tests (since unit-tests are specific to implementation details). How will we know that the implementation works as expected? How will we know nothing broke?_

This PIP consists entirely of new functionality in the form of a the proposed Watcher API.

### Rejected Alternatives

_If there are alternative ways of accomplishing the same thing, what were they? The purpose of this section is to motivate why the design is the way it is and not some other way._
