# psutil-mqtt
Publish hardware monitoring data from psutil to a MQTT broker

`psutil-mqtt` is a lightweight wrapper around [psutil](https://pypi.org/project/psutil/) that publishes CPU utilization, free memory, and other system-level stats to a MQTT broker. The primary use case is to collect system stats for ingestion into Home Assistant (HA) for alerting, reporting, and firing off any number of automations. However, given the minimal nature of this code, it could be used other purposes as well.

This project is intended to be a lightweight alternative to the (very good) [Glances](https://github.com/nicolargo/glances) project. The primary design difference is that the Glances integration into Home Assistant relies on periodically polling a RESTful API. However, the pub/sub model of MQTT is a better fit for real-time reporting of this type of data. Additionally, MQTT is already widely used in the home automation community. 

## Getting Started
First, the dependencies will need to be installed via `pip`. (`psutil-mqtt` is a Python3 application so depending on your system, you may have to explicitly call the Python3 version of pip using `pip3` or `pip-3` instead of just the plain `pip`.)
```bash
pip install psutil paho-mqtt  
```
Start sending metrics!
```bash
./psutil-mqtt.py --name NUC --cpu=60 --vm -vvvvv
```

This will create the necessary MQTT topics and start sending virtual memory and CPU utilization metrics. 
 - The `--name` parameter is used for the friendly name of the sensor in Home Assistant and for the MQTT topic names. It is the only required parameter. Here, I'm using "NUC" since my primary server is an Intel NUC.
 - The `--cpu=60` parameter is the collection interval for the CPU metrics. Here CPU metrics are gathered for 60 seconds and then the average value is published to MQTT state topic for the sensor. A good value for this option is anywhere between 60 and 1800 seconds (1 to 15 minutes).
 - The `--vm` flag indicates that virtual memory (RAM) metrics should also be published.
 - `-vvvvv` (five v's) specifies debug-level logging to the console. Reduce the quantity of v's to reduce the logging verbosity.
 
## Disk Usage
`psutil-mqtt` can publish disk usage metrics using the `du` option. Multiple `du` options can be specified to monitor different volumes. Each volume will present as a separate sensor in Home Assistant. The sensor state reports the percentage of total volume space consumed. Additional data (total volume size in bytes, free bytes, and used bytes) are accessable as state attributes on each sensor.

#### Example

`./psutil-mqtt.py --name Server1 -vvvvv --cpu=60 --vm --du='/var/spool' --du='/'`

## Using with Home Assistant (HA)
Once `psutil-mqtt` is collecting data and publishing it to MQTT, it's rather trival to use the metrics in Home Assistant.

A few assumptions:
- **Home Assistant is already configured to use a MQTT broker.** Setting up MQTT and HA is beyond the scope of this documentation. However, there are a lot of great tutorials on YouTube. Either the (recently deprecated) internal broker, or preferably an external broker like [Mosquitto](https://mosquitto.org/) will need to be installed and the HA MQTT intergration configured. I run both HA and Mosquitto in separate Docker containers on the same host and the config works well.
- **The HA MQTT integration is configured to use `homeassistant` as the MQTT autodiscovery prefix.** This is the default for the integration and also the default for `psutil-mqtt`. If you have changed this from the default, use the `--prefix` parameter to specify the correct one.
- **The MQTT broker is running on the same host you want to collect metrics from.** If not, specify either the hostname or IP address of your MQTT broker by using the `--broker` parameter.
- **You're not using any authentication or TLS to connect to the broker.** Currently `psutil-mqtt` only works with anonymous connections. User name / password authentication is fairly trivial to implement, but TLS encryption is less-so. If this is a feature you need, please post a feature request (or submit a pull request if you're the ambitious type).

Using the default prefix and a system name of `NUC` (the name of my server), the following state can be found in the "States" section of Developer Tools in HA:

![Home Assistant Developer Tools screenshot](https://github.com/jamiebegin/psutil-mqtt/blob/master/docs/dev_tools_example.png?raw=true)

The state value is the overall CPU utilization as a percentage. A detailed breakdown of kernal vs userland time, I/O wait time, etc. appear as attributes. This can be tested in the template sandbox in the Developer Tools:
```
The NUC server recently had a CPU utlization of {{ states('sensor.nuc_cpu') }}%. This included executing user-space programs for {{state_attr('sensor.nuc_cpu', 'user')}}% of cycles.
```
Which displays something similar to: `The NUC server recently had a CPU utlization of 3.1%. This included executing user-space programs for 1.9% of cycles.`

### Lovelace Dashboards

I mostly use the excellent [mini-graph-card](https://github.com/kalkih/mini-graph-card) custom card for my Lovelace dashboards. It's highly-customizable and fairly easy to make great looking charts in HA. Here is a very basic config example of using the metrics produced by `psutil-mqtt` to display the past 12 hours of CPU and memory utilization on my Intel NUC server:

```yaml
entities:
  - entity: sensor.nuc_cpu
    name: CPU Utilization
    show_legend: true
    show_line: true
    show_points: false
  - entity: sensor.nuc_virtual_memory
    name: Memory Utilization
    show_legend: true
    show_line: true
    show_points: false
hours_to_show: 12
line_width: 2
lower_bound: 0
name: NUC System Metrics
points_per_hour: 6
show:
  labels: false
  labels_secondary: false
type: 'custom:mini-graph-card'
upper_bound: 100

```
![Example card in Home Assistant](https://github.com/jamiebegin/psutil-mqtt/blob/master/docs/example_card.png?raw=true)
