
![Blueprints](blueprints.svg)


TODO:
 - how to deal with backloging when the sender is out of range?
    backloging in connect might be  possible by providing a date with the alert api call.
    backlogging in Prometheus is possible, but tricky
    sending a single update with the lowset data rate setting takes 1-2 minutes so the speed for sending backlogs is not enough. need add an option to increase the speed based on the signal strength.
 - when the gps hasn't changed too much just send 1 to indicate the same position.
 - make  the demo.


# Setup Pager duty account for the alerting(optional).
 - Sign up for an account.
 - Add a phone number under the profile Notification Rules.
 - Create a service.
 - Choose the integration as API V2.
 - copy the `Integration Key` and add it as an env vairable called `PAGERDUTY_ROUTING_KEY` with the balena setup(see below).

# Setup Slack for alerting(optional).
- Create a slack channel for receiving the alerts.
- Install the [Slack Webhooks App](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks)
- Copy the Webhook URL and add it as an env vairable called `SLACK_API_URL` with the balena setup(see below).

# Initial Setup on Balena cloud

Create an application for the sender and the receiver: `LoraGpsSender`, `SMARTLoraReciever`.

## SMARTLoraReciever Setup

- Fleet configuration
```
RESIN_HOST_CONFIG_enable_uart
RESIN_HOST_CONFIG_dtparam "i2c_arm=on","spi=on","audio=on"
RESIN_HOST_CONFIG_dtoverlay pi3-disable-bt
RESIN_HOST_CONFIG_core_freq 250 // Seems that uart is more stable with this.
RESIN_HOST_CONFIG_gpu_mem 16mb
```

 - Env vars
```
APPLICATION__SERVER_EXTERNAL__API_JWT__SECRET=....
POSTGRES_USER=postgres
POSTGRES_PASSWORD=...
PAGERDUTY_ROUTING_KEY=... # The "Integration Key" from Pager Duty for sending  alerts with the alert manager.
SLACK_API_URL=... # The "Webhook URL" from the Slack Webhooks App.
CONCENTRATOR_CONFIG= // The semtech gateway setting. See https://github.com/arribada/packet-forwarder
NETWORK_SERVER__BAND__NAME = // The chirpstack network server band settings. The default is EU_863_870. For all possible options see https://www.chirpstack.io/network-server
```



- Add a device and follow the UI steps.

- Install the [balena cli](https://github.com/balena-io/balena-cli) and apply the application compose file.

```
cd ./receiver
balena push SMARTLoraReciever
```
- Service Variables for the `chirpstack-appserver` service.
> replace the `...` with the value from the POSTGRES_PASSWORD env variable.

```
POSTGRESQL_DSN=postgres://chirpstack_as:...@chirpstack-postgresql/chirpstack_as?sslmode=disable
```
- Service Variables for the `chirpstack-networkserver` service.
```
POSTGRESQL_DSN=postgres://chirpstack_ns:...@chirpstack-postgresql/chirpstack_ns?sslmode=disable
```

### Access the applications:
Chirpstack App Server: http://deviceIPorDomain:8080<br/>
Login: admin admin

SMART connect: https://deviceIPorDomain:8443/server<br/>
Login: smart smart

### Setup chirpstack app server

- Network-servers/Add
```
name: gpsTracker
server: chirpstack-networkserver:8000
```
- Service-profiles/Create
```
name: gpsTracker
server: gpsTracker
Add gateway meta-data: selected
```
- Device-profiles/Create
```
name: gpsTracker
server: gpsTracker
LoRaWAN MAC version: 1.0.3
LoRaWAN Regional Parameters revision: A
Join (OTAA / ABP): Device supports OTAA
```
- Gateways/Create
```
name: gpsTracker
description: gpsTracker
id:... #look for the gateway_ID in the sender's compose file or in the corresponding env variable  if overridden by one. 
server: gpsTracker
location: #drag the pin to the current gateway location. This determens when the gps tracker is outside a parimeter and when to send alerts.
```
- Applications/Create
```
name: gpsTracker
description: gpsTracker
profile: gpsTracker
codec:none
```
- Applications/gpsTracker/Devices/Create
```
name: gpsSender
description: gpsSender
EUI: generate random # write it down as it will be used when setting up the sender
profile: gpsTracker
```
Applications/gpsTracker/Devices/<br/>
tab: KEYS
```
Application key: generate random # write it down as it will be used when setting up the sender
```
- Applications/gpsTracker/Integrations/Create
```
kind: HTTP
headers:
    SMARTserver: https://smart-connect:8443
    SMARTcarea: get it from SMART connect
    SMARTuser: smart
    SMARTpass: smart
    SMARTDesktopFile: # Optional header if you want to create an upload to Smart Desktop. See the section for Smart Desktop setup.
Uplink data URL: http://lora-connect:8070
```
## LoraGpsSender setup
 - Env vars
```
APP_KEY= // the one set in Chirpstack app server
DEV_EUI= // the one set in Chirpstack app server
BAND= // by default is is set to EU868 , other possible values are: AS923, EU868, AU915, US915, IN865, KR920
```
 - Fleet configuration
```
RESIN_HOST_CONFIG_enable_uart
RESIN_HOST_CONFIG_dtoverlay pi3-miniuart-bt
```
- Now add a device and follow the UI steps.

- Apply the application compose file.

```
cd ./sender
balena push ApplicationName
```

## Smart Desktop setup

If you want to create upload the data to SMART desktop need to setup SMART desktop to be connected to SMART connect and also set the content of the data to be uploaded as a header in the chirpstack HTTP integration setup.
 - Install the Smart connect plugins.
 - Setup the connection to SMART connect. It requires HTTPS and for this can use the default certificate in https://github.com/arribada/SMARTConnect
 - Create an example Patrol and export it. This will be used as a template.
 - Take the content of the Patrol file and set it as chirpstack HTTP integration header.

## Adding an additional device

From the balena UI just select `Add a new device` and follow the on screen instructions.

- Select the latest OS
- Production image
