# Smart Agriculture — IoT Course Project

University of Pisa — IoT course project. A small smart-agriculture system: constrained nodes sample temperature and humidity in a field, a backend collector ingests samples and drives an irrigation actuator above/below configured thresholds.

## Architecture

```
 [Temp sensor]  --MQTT--\
                         \                                   +-> MySQL
 [Humi sensor]  --MQTT--->  [RPL Border Router] <--6LoWPAN--+
                         /        |                          +-> CLI
 [Water actuator] <-CoAP-/        |
                                  +-- [Java Collector (MQTT broker client + CoAP client)]
```

- 6LoWPAN/RPL mesh terminated by an `rpl-border-router` exposing IPv6 prefix `fd00::/64`.
- Sensors publish JSON samples over MQTT (broker on the host, port `1883`).
- Actuator is a CoAP server (`PUT /WaterSpurt?mode=on|off`) that registers itself to the collector via `POST coap://[fd00::1]:5683/registration`.
- Collector keeps moving averages, persists samples to MySQL, and drives the actuator either automatically (threshold crossing) or via CLI.

## Components

| Path | Role | Stack |
| --- | --- | --- |
| [rpl-border-router/](rpl-border-router/) | 6LoWPAN ↔ IPv6 gateway | Contiki-NG, nRF52840 / Cooja |
| [Project_SensorTempMqtt/](Project_SensorTempMqtt/) | Temperature sensor node, publishes `temperature`, subscribes `waterspurt` | Contiki-NG, MQTT |
| [Project_SensorHumiMqtt/](Project_SensorHumiMqtt/) | Humidity sensor node, publishes `humidity`, subscribes `waterspurt` | Contiki-NG, MQTT |
| [Project_ActuatorCoap/](Project_ActuatorCoap/) | Irrigation actuator, CoAP resource `WaterSpurt` | Contiki-NG, Erbium CoAP |
| [iot.unipi.it.ProjectSmartAgriculture/](iot.unipi.it.ProjectSmartAgriculture/) | Collector app, CLI, persistence | Java 8, Paho MQTT, Californium CoAP, MySQL |

## Protocols & Topics

- MQTT broker: `fd00::1:1883` (configured in each node's `project-conf.h` and in [mqttSensorTemp.c:24](Project_SensorTempMqtt/mqttSensorTemp.c#L24)).
  - `temperature` — `{"Node": <id>, "temperature": <int>}`
  - `humidity` — `{"Node": <id>, "humidity": <int>}`
  - `waterspurt` — `on` | `off` (commands)
- CoAP: `coap://[fd00::1]:5683`
  - `POST /registration` — actuator → collector, payload `registration`, expects `Success`.
  - `PUT /WaterSpurt?mode=on|off` — collector → actuator.

## Build & Run

Prerequisites: Contiki-NG toolchain (`gcc-arm-none-eabi` or Cooja), Java 8, Maven, MySQL 8, a local MQTT broker (e.g. Mosquitto) bound on `fd00::1`.

### Border router

```sh
cd rpl-border-router
make TARGET=nrf52840 BOARD=dongle border-router.dfu-upload PORT=<tty>
# or for Cooja: make TARGET=cooja
```

Then run tunslip on the host: `sudo make TARGET=nrf52840 connect-router`.

### Sensor / actuator nodes

```sh
cd Project_SensorTempMqtt  && make TARGET=nrf52840
cd Project_SensorHumiMqtt  && make TARGET=nrf52840
cd Project_ActuatorCoap    && make TARGET=nrf52840
```

### Collector

```sh
cd iot.unipi.it.ProjectSmartAgriculture
mvn clean package
java -jar target/iot.unipi.it.ProjectSmartAgriculture-0.0.1-SNAPSHOT-jar-with-dependencies.jar
```

DB connection parameters are hard-coded in [DBDriver.java:18-24](iot.unipi.it.ProjectSmartAgriculture/src/main/java/iot/unipi/it/SmartAgriculture/persistence/DBDriver.java#L18-L24) — schema `SmartAgricultureDB` with tables `temperature(node, degree)` and `humidity(node, percentuage)` must exist on `localhost:3306`.

## CLI

Commands accepted by the collector ([SmartAgricultureCollector.java](iot.unipi.it.ProjectSmartAgriculture/src/main/java/iot/unipi/it/SmartAgriculture/App/SmartAgricultureCollector.java)):

```
!help <cmd>             show command detail
!get_humidity           last humidity moving average
!get_temperature        last temperature moving average
!spurt_the_water_on     activate irrigation (MQTT broadcast + CoAP PUT)
!spurt_the_water_off    deactivate irrigation
!set_humidity_th <v>    set humidity activation threshold
!set_temperature_th <v> set temperature activation threshold
!exit
```

## Documentation

Full report and slides: [Documentation.pdf](Documentation.pdf), [Presentation.pdf](Presentation.pdf).
