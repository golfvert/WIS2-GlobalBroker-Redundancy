# WIS2-GlobalBroker-NodeRed

The code in this repository is to provide basic reference of a Global Broker as defined in the (draft) technical specification of WIS2. 
See https://community.wmo.int/en/WIS2_Technical_Specification_Guidance
The repo has been cloned from the official NodeRed repository.

## What is here ?

1. This is the source code of the container `golfvert/wis2globalbrokernodered` and the required files to run the deduplication software in front of a Global Broker
2. configuration files to run the container available at `golfvert/wis2globalbrokernodered`

## What does it do ?

1. Listen to subscribed topics from WIS2Node and other Global Brokers (one subscription per container)
2. Optionally verify/discard/ignore the message for its validity compared to approved message format
3. Look at the `id` in the message. 
4. Through a redis request check if that `id` has already been seen in the last 15 minutes. If yes, simply discard the message
5. If not, publish the message to the attached Global Broker
6. It also provides prometheus metrics available at http://@IP:1880/metrics

## How to use it ?

Download 
- docker-compose.yaml

and edit:

Copy & Paste the subscriber_mqtt_1 to get one container per WIS2Node or other Global Brokers / Global Cache to subscribe to. Do NOT subscribe to the local Global Broker.
- Change the name of the container (make sure it is unique!)
- Change all MQTT_SUB_* to connect to the remote broker and to the topic from that source. In the example below `origin/a/wis2/fra/#` will subscribe to all topic from France according to WIS2 agreed topic hierarchy. It can be a list of topics separated by ",". Such as `origin/a/wis2/fra/#,cache/a/wis2/fra/#`. This is, typically, useful when subscribing to another Global Broker.
- Change ports 1st 1880:1880, 2nd 1881:1880,... or run behind a traefik proxy (this is the preferred method).

```
  subscriber_mqtt_1:
    container_name: subscriber_mqtt_1
    image: golfvert/wis2globalbrokernodered
    environment:
      - TZ=Europe/Paris
      - MQTT_SUB_BROKER=Broker_URL   # WIS2Node URL broker such as mqtts://broker.example.com:8883 or wss://broker.example.com:443
      - MQTT_SUB_USERNAME=
      - MQTT_SUB_PASSWORD=
      - MQTT_SUB_TOPIC=Topic_to_sub   # e.g. origin/a/wis2/fra/#. 
      - MQTT_SUB_VERIFYCERT= true   # if using SSL should the certificate by checked (prevent slef-signed certificates to work. Or not)
      - MQTT_PUB_BROKER=GlobalBroker_URL   # Global Broker URL such as mqtts://globalbroker.site.com:8883 or wss://globalbroker.site.com:443
      - MQTT_PUB_USERNAME=
      - MQTT_PUB_PASSWORD=
      - MQTT_MONIT_TOPIC=Topic_to_publish_on_Global_Broker
      - MSG_CHECK_OPTION=verify      # Should messages be "verify" (just add _comment in the notification message), "discard" (bin the message if not correct), "ignore" (don't check the messages)
      - PROM_CENTRE_ID=Name_of_Center used as label
      - PROM_COUNTRY=3-letter code used as label
    ports:
      - "1880:1880"
    networks:
      - wis2relay
    depends_on:
      - redis
 ```
 
There must be a redis container running on the same wis2relay bridged docker network. So a compose like that can be used:

```
services:
  redis:
    container_name: redis
    image: redis:latest
    command: redis-server --save 20 1
    ports:
      - "6379:6379"
    networks:
      - wis2relay
    environment:
      - REDIS_URL=redis://redis:6379/0
    volumes:
      - /localdir/redis:/data
networks:
  wis2relay:
    driver: bridge
    
```

MQTT_MONIT_TOPIC is optional. If defined, statistics on the status of the subsciption to the remote broker will be published to the Global Broker on the topic MQTT_MONIT_TOPIC/status. And every minute, the time difference (in seconds) between the current time and the time when the last message has been received from the remote broker. This will will be published on MQTT_MONIT_TOPIC/pubsub. If MQTT_MONIT_TOPIC is empty or does not exist no statistics will be published.
PROM_CENTRE_ID is used as a variable as label in Prometheus.
PROM_COUNTRY is used as a variable as label in Prometheus.

When done, save the docker-compose.yaml and start it with `docker compose up -d`

It will  subscribe to all "remote" destinations (WIS2node(s), Global Caches, other Global Brokers) and will publish to the local Global Broker.

Initially this was mainly a proof of concept implementation. However, it is in use for a few months now and has proven to be rock solid. So using it for 24/7 production can be considered with confidence.

## Check Global Broker Anti-Loop Node-Red container behaviour
Node-red interface is available at http://@IP:1880/admin/ (unless behing a proxy or other changes - eg. port). Accessing this interface will show the flows in action. Adding debug nodes may help in understaning the function of the anti-loop software. This interface is NOT password protected. Typically unprotected access should be prevented through port filtering and proxying the access via traefik.

## How to modify it ?

This is a fork from the official Node-Red repo. Follow official documentation to tweak it to your needs.
