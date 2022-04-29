Docker host example
==============

This example demonstrates a bug in operation when attempting to connect/reconnect to an mqtt broker operating from the host machine. The changes I have made to the original project are as follows:

- In`pub/main.go` and `sub/main.go`:
    - Allow SERVERADDRESS to be set by env var
    - Uncomment logging lines
- `docker-compose.yml`:
    - Set SERVERADDRESS on pub and sub from external SERVERADDRESS env var, with default pointing at docker mosquitto.
- Add new `docker-compose.mosquitto.yml` to allow starting mosquitto in separate network and exposing to host

## Standard operation (all working):

### Normal operation

```bash
docker-compose up
```

Result:
`pub` and `sub` connect to the broker and behave normally

### To demonstrate succesful reconnection when mqtt broker goes down then up

In terminal 1:

```bash
docker-compose up
```

In terminal 2:
```
docker-compose stop mosquitto
docker-compose start mosquitto
```

Result:
`pub` and `sub` lose connections then successfully reconnect


### Succesful connection when services start without broker, then broker starts

```bash
docker-compose up
```

In terminal 2:
```
docker-compose stop mosquitto
docker-compose restart sub pub
docker-compose start mosquitto
```

Result:
After restart, `pub` and `sub` services initially can't connect, then succeed when mosquitto is started


## Using External Host:

### Normal Operation

In terminal 2:

```bash
# Start the external broker
docker-compose -p mosquitto -f docker-compose.mosquitto.yml up -d
```

In terminal 1:

```bash
# Start pub and sub pointing at external broker
SERVERADDRESS=host.docker.internal:1883 docker-compose up
```

Result:
`pub` and `sub` connect to the broker and behave normally

### Unsuccesful reconnection when mqtt broker goes down then up

In terminal 2:

```bash
# Start the external broker
docker-compose -p mosquitto -f docker-compose.mosquitto.yml up -d
```

In terminal 1:

```bash
# Start pub and sub pointing at external broker
SERVERADDRESS=host.docker.internal:1883 docker-compose up
```

In terminal 2:

```bash
# Stop the external broker
docker-compose -p mosquitto -f docker-compose.mosquitto.yml down

# Restart the external broker
docker-compose -p mosquitto -f docker-compose.mosquitto.yml up -d

```

Result: 

Both services display single reconnection message, but never recover (pub keeps publishing)

```log
sub_1        | [ERROR] [client]   Connecting to tcp://host.docker.internal:1883 CONNACK was not CONN_ACCEPTED, but rather Connection Error
sub_1        | [DEBUG] [client]   Reconnect failed, sleeping for 1 seconds: network Error : EOF
pub_1        | [ERROR] [net]      connect got error EOF
pub_1        | [ERROR] [client]   Connecting to tcp://host.docker.internal:1883 CONNACK was not CONN_ACCEPTED, but rather Connection Error
pub_1        | [DEBUG] [client]   Reconnect failed, sleeping for 1 seconds: network Error : EOF
...
```

### Failure to connect when services start without broker, then broker starts

In terminal 2:

```bash
# Stop the external broker
docker-compose -p mosquitto -f docker-compose.mosquitto.yml down
```

In terminal 1:

```bash
# Start pub and sub pointing at external broker
SERVERADDRESS=host.docker.internal:1883 docker-compose up
```

In terminal 2:

```bash
# Start the external broker
docker-compose -p mosquitto -f docker-compose.mosquitto.yml up -d

```

Result: 

Both services start and hang at `connect started`, never recover

```log
sub_1        | SERVERADDRESS: host.docker.internal:1883
sub_1        | [DEBUG] [client]   Connect()
sub_1        | [DEBUG] [store]    memorystore initialized
pub_1        | SERVERADDRESS: host.docker.internal:1883
pub_1        | [DEBUG] [client]   Connect()
pub_1        | [DEBUG] [store]    memorystore initialized
pub_1        | [DEBUG] [client]   about to write new connect msg
pub_1        | [DEBUG] [client]   socket connected to broker
pub_1        | [DEBUG] [client]   Using MQTT 3.1.1 protocol
pub_1        | [DEBUG] [net]      connect started
sub_1        | [DEBUG] [client]   about to write new connect msg
sub_1        | [DEBUG] [client]   socket connected to broker
sub_1        | [DEBUG] [client]   Using MQTT 3.1.1 protocol
sub_1        | [DEBUG] [net]      connect started
```


# Old text

This folder contains all that is needed to build an environment with a publisher, broker (mosquitto) and subscriber
using docker (ideally `docker-compose`). While it provides an end-to-end example its primary purpose is to act as a
starting point for producing reproducible examples (when logging an issue with the library).

Because the publisher (`pub`), broker (`mosquitto`) and subscriber (`sub`) run in separate containers this setup closely
simulates a real deployment. One thing to bear in mind is that the network between the containers is very fast and
reliable (but there are some techniques that can be used to simulate failures etc).

# Usage

Ensure that you have [docker](https://docs.docker.com/get-docker/) and
[docker-compose](https://docs.docker.com/compose/install/) installed.

To start everything up change into the `cmd/docker` folder and run:

```
docker-compose up --build --detach
```

This will start everything up in the background. You can see what is happening by running:

```
docker-compose logs --follow
```

This will display a lot of information (mosquitto is running with debug level logging). To see the subscriber logs:

```
docker-compose logs --follow sub
```

Note: Messages received by the subscriber will be written to `shared/receivedMessages` (you may want to delete the
contents of this file from time to time!).

To stop everything run:

```
docker-compose down
```

Feel free to copy the folder and modify the publisher/subscriber to work as you want them to!

Note: The `pub` and `sub` containers connect to mosquitto via the internal network (`test-net`) but mosquitto should
also be available on the host port `8883` if you wish to connect to it. This will not work if you have mosquitto
installed locally (edit the `docker-compose.yml` and change the `published` port).

# Simulating Network Connection Loss

You can simulate the loss of network connectivity by disconnecting the network adapter within a container. e.g.

```
docker network disconnect lostpackets_test-net lostpackets_pub_1
docker network connect lostpackets_test-net lostpackets_pub_1
```
  