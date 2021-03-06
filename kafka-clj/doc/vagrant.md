Kafka Vagrant Development
==========================

Home: https://github.com/gerritjvv/kafka-fast



# Overview

Using vagrant allows you to develop kafka application as if your running a full production deployed cluster.

Note: you need vagrant installed before running it :) (https://www.vagrantup.com/)

## Changes in Vagrant boxes:

Vagrant boxes use precise64 (instead of other previous custom boxes like gerritjvv/centos-6-java etc ) which is a default hosted and available box for vagrant.
This does mean extra time is required to download java etc and run update on the boxes when provisioned. Please be patient when they build the first time
,this is true for most vagrant builds, and report any issues as tickets in the "issues" tab.

## Requirements


```vagrant plugin install vagrant-vbguest```
```vagrant plugin install vagrant-hostmanager```


### Shared folders

The Vagrantfile uses nfs for shared folders, this is faster and more sane than the current alternatives.
You might be asked for you're password when starting the machines.

## Kerberos

The Kafka brokers are configured to use Kerberos authentication.
This allows the kafka client to be easily tested in a Kerberos setup, but also means some more
setup from the client side before using the vagrant setup.

Clients must sign in via keytabs generated in ```vagrant/keytabs```, these keytabs are generated
via the ```vagrant/scripts/kerberos.sh``` script as part of the kerberos vagrant box.

## Machines/Boxes

The boxes launched are:

*Kerberos*
  * kerberos kerberos.kafkafast 192.168.4.60

*Brokers*
  * broker1 broker1.kafkafast 192.168.4.40:9092
  * broker2 broker2.kafkafast 192.168.4.41:9092
  * broker3 broker3.kafkafast 192.168.4.42:9092
  
*Zookeeper*
  * zookeeper1 zookeeper1.kafkafast 192.168.4.2:2181


*Services* -- Redis

services1 services1.kafkafast 192.168.4.10

  * redis 192.168.4.10:6379
  * redis 192.168.4.10:6380
  * redis 192.168.4.10:6381
  * redis 192.168.4.10:6382
  * redis 192.168.4.10:6383

The services box is there to not only run the redis instance but any other instances such as a mysql db etc that  
is required for a particular usecase.

## Startup

Startup order is important:

To run type:

```vagrant up --provision kerberos```

```vagrant up --provision services1```

```vagrant up --provision zookeeper1```


```vagrant up --provision broker1```
```vagrant up --provision broker2```
```vagrant up --provision broker3```

Note: ssh into all the machines checking that each instance is running,
check for errors in their logs, use ps uax etc.


To destroy all boxes run:

```vagrant destroy```

## What next?

Once the boxes are up and running you can refer to them and use them as any other kafka cluster.

But first test that the cluster is up and running by running ping on each of the ips above.


### Send data to the cluster:

Remember to create the topic first using vagrant/scripts/create_topic_remote.sh "my-topic"

*Do first*

Check that the kerberos options are enabled in the ```project.clj```

```
"-Dsun.security.krb5.debug=true"
"-Djava.security.debug=gssloginconfig,configfile,configparser,logincontext"
"-Djava.security.auth.login.config=/vagrant/vagrant/config/kafka_client_jaas.conf"
"-Djava.security.krb5.conf=/vagrant/vagrant/config/krb5.conf"
```

Ssh into the broker1 box and cd to ```/vagrant```, this is important, running it outside of broker1 will not work
with the defined jaas and krb5 config.

Note: when you run the repl on broker1 it might be necessary to configure the ram used on the box.  The easiest root
is first to set the ram for the repl profile to a lower value see ```project.clj``` ```:profiles {:repl {:jvm-opts [ ```


```
vagrant ssh broker1
cd /vagrant

lein repl
```


*Clojure*

```clojure
(use 'kafka-clj.client :reload)

(def msg1kb (.getBytes (clojure.string/join "," (range 10))))

;;use flush-on-write true for testing, this will flush the message on write to kafka
;;set to false for performance in production
;;:jaas "KafkaClient" refers to the section in the jaas file passed in using the environment
;;variable -Djava.security.auth.login.config, see project.clj
(def c (create-connector [{:host "192.168.4.41" :port 9092}]  {:flush-on-write true :jaas "KafkaClient" :kafka-version ""0.9"}))

(send-msg c "my-topic" msg1kb)
```

*Java*

```java
import kakfa_clj.core.*;

KafkaConf conf = new KafkaConf();
conf.setJaas("KafkaClient");
conf.setKafkaVersion("0.9"); //do for kafka 0.9 version only

Producer producer = Producer.connect(conf, new BrokerConf("192.168.4.40", 9092));
producer.sendMsg("my-topic", "Hi".getBytes("UTF-8"));
producer.close();
```

### Consume data from the cluster

Remember to create the topic first using vagrant/scripts/create_topic_remote.sh "my-topic"

Please Note: By default on a clean setup the consumer will see no offsets in redis the latest offsets
will be used, which means you will not see any data written till after the consumer has started for the first time.
This behaviour can be changed by reading https://github.com/gerritjvv/kafka-fast#offsets-and-consuming-earliest.

For this reason we add sending messages just before calling read-msg in this test.



*Clojure*

```clojure
(use 'kafka-clj.consumer.node :reload)
(use 'kafka-clj.client :reload)

(def consumer-conf {:bootstrap-brokers [{:host "192.168.4.41" :port 9092}]
                    :jaas "KafkaClient"
                    :kafka-version "0.9"
                    :redis-conf {:host ["192.168.4.10:6379" "192.168.4.10:6380" "192.168.4.10:6381"
                                        "192.168.4.10:6382" "192.168.4.10:6383" ]
                                        :slave-connection-pool-size 500
                                        :master-connection-pool-size 500
                                        :max-active 5 :timeout 1000 :group-name "test"} :conf {}})
                                        
(def node (create-node! consumer-conf ["my-topic"]))

;;; important, if any redis errors etc, please see the "Errors and fixes" section
;;;            sometimes the redis cluster via vagrant setup doesn't go 100% :(.

;; important, wait till you see something like  work-organiser:288 - Set initial offsets [ my-topic / 0 ]:  42336

(def c (create-connector [{:host "192.168.4.41" :port 9092}]  {:flush-on-write true :jaas "KafkaClient" :kafka-version "0.9"}))

(dotimes [i 1000] (send-msg c "my-topic" (.getBytes "MyTestMessage-12121212121212121212")))

(read-msg! node)
;;for a single message
(def m (msg-seq! node))

```

*Java*

```java
import kakfa_clj.core.*;


KafkaConf conf = new KafkaConf();
conf.setJaas("KafkaClient");
conf.setKafkaVersion("0.9"); //do for kafka 0.9 version only

Consumer consumer = Consumer.connect(conf, new BrokerConf[]{new BrokerConf("192.168.4.41", 9092)}, new RedisConf("192.168.4.10", 6379, "test-group"), "my-topic");
Message msg = consumer.readMsg();

String topic = msg.getTopic();
long partition = msg.getPartition();
long offset = msg.getOffset();
byte[] bts = msg.getBytes();

//Add topics
consumer.addTopics("topic1", "topic2");

//Remove topics
consumer.removeTopics("topic1", "topic2");

//Iterator: Consumer is Iterable and consumer.iterator() returns a threadsafe iterator
//          that will return true unless the consumer is closed.
for(Message message : consumer){
  System.out.println(message);
}

//close
consumer.close();
```

### Errors and fixes

#### CompilerException org.redisson.client.RedisConnectionException: Can't connect to servers!

This means that the redis cluster is not probably installed. Kafka-clj can run with a redis cluster or
just a single redis node, but for vagrant a redis cluster is chosen to show how it can be done.

Test the redis instances by logging into the services1 instance:

```vagrant ssh services1```

Then check that all instances are running

```ps aux |grep "redis"```
```
vagrant@services1:~$ ps aux |grep "redis"
redis     9285  0.1  0.5  38220  2240 ?        Ssl  20:58   0:02 /opt/redis-3.0.0-rc4/src/redis-server 192.168.4.10:6379 [cluster]
redis     9298  0.1  0.6  38220  2564 ?        Ssl  20:58   0:02 /opt/redis-3.0.0-rc4/src/redis-server 192.168.4.10:6380 [cluster]
redis     9311  0.1  0.6  38220  2596 ?        Ssl  20:58   0:02 /opt/redis-3.0.0-rc4/src/redis-server 192.168.4.10:6381 [cluster]
redis     9323  0.1  0.6  38220  2572 ?        Ssl  20:58   0:02 /opt/redis-3.0.0-rc4/src/redis-server 192.168.4.10:6382 [cluster]
redis     9334  0.1  0.6  38220  2600 ?        Ssl  20:58   0:03 /opt/redis-3.0.0-rc4/src/redis-server 192.168.4.10:6383 [cluster]
```

Try to set some values via the redis cli

```
redis-cli -h 192.168.4.10 set foo bar
```

If you see output like "Cluster is down" then run the join cluster script

``` echo 'yes' | /opt/redis-3.0.0-rc4/src/redis-trib.rb create 192.168.4.10:6379 192.168.4.10:6380 192.168.4.10:6381 192.168.4.10:6382 192.168.4.10:6383```



