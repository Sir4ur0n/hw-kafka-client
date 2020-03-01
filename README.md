# hw-kafka-client
[![CircleCI](https://circleci.com/gh/haskell-works/hw-kafka-client.svg?style=svg&circle-token=5f3ada2650dd600bc0fd4787143024867b2afc4e)](https://circleci.com/gh/haskell-works/hw-kafka-client)

Kafka bindings for Haskell backed by the
[librdkafka C module](https://github.com/edenhill/librdkafka).

## Credits
This project is inspired by [Haskakafka](https://github.com/cosbynator/haskakafka)
which unfortunately doesn't seem to be actively maintained.

## Ecosystem
HaskellWorks Kafka ecosystem is described here: https://github.com/haskell-works/hw-kafka

# Consumer
High level consumers are supported by `librdkafka` starting from version 0.9.
High-level consumers provide an abstraction for consuming messages from multiple
partitions and topics. They are also address scalability (up to a number of partitions)
by providing automatic rebalancing functionality. When a new consumer joins a consumer
group the set of consumers attempt to "rebalance" the load to assign partitions to each consumer.

### Example:

```bash
$ stack build --flag hw-kafka-client:examples
```

or

```bash
$ stack build --exec kafka-client-example --flag hw-kafka-client:examples
```

A working consumer example can be found here: [ConsumerExample.hs](example/ConsumerExample.hs)</br>
To run an example please compile with the `examples` flag.

```haskell
import Control.Exception (bracket)
import Data.Monoid ((<>))
import Kafka
import Kafka.Consumer

-- Global consumer properties
consumerProps :: ConsumerProperties
consumerProps = brokersList [BrokerAddress "localhost:9092"]
             <> groupId (ConsumerGroupId "consumer_example_group")
             <> noAutoCommit
             <> logLevel KafkaLogInfo

-- Subscription to topics
consumerSub :: Subscription
consumerSub = topics [TopicName "kafka-client-example-topic"]
           <> offsetReset Earliest

-- Running an example
runConsumerExample :: IO ()
runConsumerExample = do
    res <- bracket mkConsumer clConsumer runHandler
    print res
    where
      mkConsumer = newConsumer consumerProps consumerSub
      clConsumer (Left err) = return (Left err)
      clConsumer (Right kc) = (maybe (Right ()) Left) <$> closeConsumer kc
      runHandler (Left err) = return (Left err)
      runHandler (Right kc) = processMessages kc

-------------------------------------------------------------------
processMessages :: KafkaConsumer -> IO (Either KafkaError ())
processMessages kafka = do
    mapM_ (\_ -> do
                   msg1 <- pollMessage kafka (Timeout 1000)
                   putStrLn $ "Message: " <> show msg1
                   err <- commitAllOffsets OffsetCommit kafka
                   putStrLn $ "Offsets: " <> maybe "Committed." show err
          ) [0 .. 10]
    return $ Right ()
```

# Producer

`kafka-client` producer supports sending messages to multiple topics.
Target topic name is a part of each message that is to be sent by `produceMessage`.

A working producer example can be found here: [ProducerExample.hs](example/ProducerExample.hs)

### Delivery reports

Kafka Producer maintains its own internal queue for outgoing messages. Calling `produceMessage`
does not mean that the message is actually written to Kafka, it only means that the message is put
to that outgoing queue and that the producer will (eventually) push it to Kafka.

However, it is not always possible for the producer to send messages to Kafka. Network problems
or Kafka cluster being offline can prevent the producer from doing it.

When a message cannot be sent to Kafka for some time (see `message.timeout.ms` [configuration](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md) option),
the message is *dropped from the outgoing queue* and the *delivery report* indicating an error is raised.

It is possible to configure `hw-kafka-client` to set an infinite message timeout so the message is
never dropped from the queue:

```haskell
producerProps :: ProducerProperties
producerProps = brokersList [BrokerAddress "localhost:9092"]
             <> sendTimeout (Timeout 0)           -- for librdkafka "0" means "infinite" (see https://github.com/edenhill/librdkafka/issues/2015)
```

*Delivery reports* provide the way to detect when producer experiences problems sending messages
to Kafka.

Currently `hw-kafka-client` only supports delivery error callbacks:

```haskell
producerProps :: ProducerProperties
producerProps = brokersList [BrokerAddress "localhost:9092"]
             <> setCallback (deliveryCallback print)
```

In the example above when the producer cannot deliver the message to Kafka,
the error will be printed (and the message will be dropped).

### Example

```Haskell
{-# LANGUAGE OverloadedStrings #-}
import Control.Exception (bracket)
import Control.Monad (forM_)
import Data.ByteString (ByteString)
import Kafka.Producer

-- Global producer properties
producerProps :: ProducerProperties
producerProps = brokersList [BrokerAddress "localhost:9092"]
             <> logLevel KafkaLogDebug

-- Topic to send messages to
targetTopic :: TopicName
targetTopic = TopicName "kafka-client-example-topic"

-- Run an example
runProducerExample :: IO ()
runProducerExample =
    bracket mkProducer clProducer runHandler >>= print
    where
      mkProducer = newProducer producerProps
      clProducer (Left _)     = return ()
      clProducer (Right prod) = closeProducer prod
      runHandler (Left err)   = return $ Left err
      runHandler (Right prod) = sendMessages prod

sendMessages :: KafkaProducer -> IO (Either KafkaError ())
sendMessages prod = do
  err1 <- produceMessage prod (mkMessage Nothing (Just "test from producer") )
  forM_ err1 print

  err2 <- produceMessage prod (mkMessage (Just "key") (Just "test from producer (with key)"))
  forM_ err2 print

  return $ Right ()

mkMessage :: Maybe ByteString -> Maybe ByteString -> ProducerRecord
mkMessage k v = ProducerRecord
                  { prTopic = targetTopic
                  , prPartition = UnassignedPartition
                  , prKey = k
                  , prValue = v
                  }
```

### Synchronous sending of messages
Because of the asynchronous nature of librdkafka, there is no API to provide
synchronous production of messages. It is, however, possible to combine the
delivery reports feature with that of callbacks. This can be done using the
`Kafka.Producer.produceMessage'` function.

```haskell
produceMessage' :: MonadIO m
                => KafkaProducer
                -> ProducerRecord
                -> (DeliveryReport -> IO ())
                -> m (Either ImmediateError ())
```

Using this function, you can provide a callback which will be invoked upon the
produced message's delivery report. With a little help of `MVar`s or similar,
you can in fact, create a synchronous-like interface.

```haskell
sendMessageSync :: MonadIO m
                => KafkaProducer
                -> ProducerRecord
                -> m (Either KafkaError Offset)
sendMessageSync producer record = liftIO $ do
  -- Create an empty MVar:
  var <- newEmptyMVar

  -- Produce the message and use the callback to put the delivery report in the
  -- MVar:
  res <- produceMessage' producer record (putMVar var)

  case res of
    Left (ImmediateError err) ->
      pure (Left err)
    Right () -> do
      -- Flush producer queue to make sure you don't get stuck waiting for the
      -- message to send:
      flushProducer producer

      -- Wait for the message's delivery report and map accordingly:
      takeMVar var >>= return . \case
        DeliverySuccess _ offset -> Right offset
        DeliveryFailure _ err    -> Left err
        NoMessageError err       -> Left err
```

_Note:_ this is a semi-naive solution as this waits forever (or until
librdkafka times out). You should make sure that your configuration reflects
the behavior you want out of this functionality.

# Installation

## Installing librdkafka

Although `librdkafka` is available on many platforms, most of
the distribution packages are too old to support `kafka-client`.
As such, we suggest you install from the source:

```bash
    git clone https://github.com/edenhill/librdkafka
    cd librdkafka
    ./configure
    make && make install
```

Sometimes it is helpful to specify openssl includes explicitly:

```
    LDFLAGS=-L/usr/local/opt/openssl/lib CPPFLAGS=-I/usr/local/opt/openssl/include ./configure
```

If you are using Stack with Nix, don't forget to declare `rdkafka` as extra package:

```yaml
# stack.yaml
nix:
  enable: true
  packages:
    - rdkafka
```

## Installing Kafka

The full Kafka guide is at http://kafka.apache.org/documentation.html#quickstart

Alternatively `docker-compose` can be used to run Kafka locally inside a Docker container.
To run Kafka inside Docker:

```bash
$ docker-compose up
```
