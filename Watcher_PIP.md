* **Status**: Proposal
* **Author**: Will McCarley
* **Pull Request**: (Link to the main pull request to resolve this PIP)
* **Mailing List discussion**:
* **Release**:

### Motivation

_Some clients need the ability to programmatically monitor a given topic's conumers/subscriptions for certain events. This can be accomplished to a limited degree by using the available REST APIs and published stats (e.g. Prometheus.) However this solution requires the administrator of the system to provide access to these APIs and any monitoring solution will be based on polling rather than near-realtime alerting._

_The purpose of this PIP is to introduce a way for clients to programmatically attach 'Watchers' to topics to observe the behavior of the connected consumers & producers._

_One example is to create an out-of-band application that 'watches' a given subscription and scales the consumer application horizontally when the backlog becomes unacceptably high._

_A second example is for an application to monitor all subscriptions to a topic and programmatically determine if/when a given downstream subscription is behind and send a notification_

_A third example is for applications to watch a 

### Public Interfaces

```java
public interface TopicEventListener {

    public void onConsumerConnect(String topic, String subscription, ConsumerCnx cnx);
    
    public void onConsumerStats(String topic, String subscription, ServerConsumerStats[] stats);
    
    public void onConsumerDisconnect(String topic, String subscription, ConsumerCnx cnx);
    
    public void onProducerConnect(String topic, ProducerCnx cnx);
    
    public void onProducerStats(String topic, ServerProducerStats[] stats);
    
    public void onProducerDisconnect(String topic, ProducerCnx cnx);
    
    public void onSubscriptionCreate(String topic, String subscription, SubscriptionType type, SubscriptionInitialPosition position);
    
    public void onSubscriptionBacklog(String topic, String subscription, long backlogSize);
    
    public void onSubscriptionCatchUp(String topic, String subscription);
    
    public void onSubscriptionSeek(String topic, String subscription, long oldPosition, long newPosition);

    public void onSubscriptionIdle(String topic, String subscription);

    public void onSubscriptionDelete(String topic, String subscription);

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
    
    // implementation of the TopicEventListener interface
    .eventListener(myTopicEventListener)
    .watch()
    
...

watcher.close();
```
_Briefly list any new interfaces that will be introduced as part of this proposal or any existing interfaces that will be removed or changed. The purpose of this section is to concisely call out the public contract that will come along with this feature._

A public interface is any change to the following:

- Data format, Metadata format
- The wire protocol and API behavior
- Any class in the public packages
- Monitoring
- Command-line tools and arguments
- Configuration settings
- Anything else that will likely break existing users in some way when they upgrade

### Proposed Changes

_Describe the new thing you want to do in appropriate detail. This may be fairly extensive and have large subsections of its own. Or it may be a few sentences. Use judgment based on the scope of the change._

### Compatibility, Deprecation, and Migration Plan

- What impact (if any) will there be on existing users? 
- If we are changing behavior how will we phase out the older behavior? 
- If we need special migration tools, describe them here.
- When will we remove the existing behavior?

### Test Plan

_Describe in few sentences how the BP will be tested. We are mostly interested in system tests (since unit-tests are specific to implementation details). How will we know that the implementation works as expected? How will we know nothing broke?_

### Rejected Alternatives

_If there are alternative ways of accomplishing the same thing, what were they? The purpose of this section is to motivate why the design is the way it is and not some other way._
