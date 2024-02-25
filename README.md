# weewx-sdr-ng

Fork of Matthew Wall's great project to get PRs merged faster and modernize the code.

This is a driver for weewx that captures data from software-defined radio.
It works with open source rtl sdr software that in turn works with
inexpensive, broad spectrum radio receivers such as the Realtek RTL2838UHIDIR.
These devices cost about 20$US and are capable of receiving radio signals from
weather stations, energy monitors, doorbells, and many other devices that use
unlicensed spectrum such as 433MHz, 838MHz, and 900MHz frequencies.

## Hardware

Tested with the Realtek RTL2838UHIDIR. Should work with any software-defined
radio that is compatible with the rtl-sdr software. Uses the modules in
rtl_433 to recognize packets.

Output from many different sensors is supported. To see the list of supported
sensors, run the driver directly with the list-supported action.

If a sensor is supported by rtl_433 but not by weewx-sdr, it is a fairly simple
matter of writing a parser for that sensor within weewx-sdr. Things are a bit
more complicated if a sensor is not supported by rtl_433.


## Prerequisites

Running installations of 

- weewx
- rtl-sdr
- rtl_433

## Installation

1. Install the driver with
```shell
weectl extension install https://github.com/an0nfunc/weewx-sdr-ng/archive/refs/heads/master.zip
```

2. Run the driver directly to identify the packets you want to capture. You need to be in the weewx install dir for this to work.

```shell
PYTHONPATH=. python bin/user/sdr.py --cmd="rtl_433 -M time:unix -F json"
```

3. modify the `[SDR]` section of weewx.conf using a text editor
   - create a `[[sensor_map]]` for the data you want to capture
   - possibly modify the `cmd` parameter

4. set `[Station]` -> `station_type` to `SDR`
5. (re)start weewx

## How to run the driver directly

Run the driver directly for testing and diagnostics. For example, if weewx
was installed using setup.py:

```shell
PYTHONPATH=/home/weewx python /home/weewx/bin/user/sdr.py --help
```

## Configuration

Use the `[SDR]` section of the weewx configuration file (nominally weewx.conf) to
adjust the driver configuration.

The default configuration uses this command:

```shell
rtl_433 -M time:unix -F json
```

Specify different options using the cmd parameter. For example:

```
[SDR]
    driver = user.sdr
    cmd = rtl_433 -M time:unix -F json -R 17 -R 44 -R 50
```

The rtl_433 executable emits data for many different types of sensors, some of
which have similar output. Use the sensor_map to distinguish between sensors
and map the output from rtl_433 to the database fields in weewx.

### Examples

##### Acurite 5n1 sensor 0BFA and t/h sensor 24A4
```
[SDR]
    driver = user.sdr
    [[sensor_map]]
        windDir = wind_dir.0BFA.Acurite5n1Packet
        windSpeed = wind_speed.0BFA.Acurite5n1Packet
        outTemp = temperature.0BFA.Acurite5n1Packet
        outHumidity = humidity.0BFA.Acurite5n1Packet
        rain_total = rain_total.0BFA.Acurite5n1Packet
        inTemp = temperature.24A4.AcuriteTowerPacket
        inHumidity = humidity.24A4.AcuriteTowerPacket
```

##### Acurite 986 fridge/freezer sensor set 1R and 2F
```
[SDR]
    driver = user.sdr
    [[sensor_map]]
        extraTemp1 = temperature.1R.Acurite986Packet
        extraTemp2 = temperature.2F.Acurite986Packet
```

##### Acurite 06002RM t/h sensor 3067
```
[SDR]
    driver = user.sdr
    [[sensor_map]]
        inTemp = temperature.3067.AcuriteTowerPacket
        inHumidity = humidity.3067.AcuriteTowerPacket
```

##### Hideki TS04 sensors with channel=1 and channel=2
```
[SDR]
    driver = user.sdr
    [[sensor_map]]
        outBatteryStatus = battery.1:9.HidekiTS04Packet
        outHumidity = humidity.1:9.HidekiTS04Packet
        outTemp = temperature.1:9.HidekiTS04Packet
        inBatteryStatus = battery.2:9.HidekiTS04Packet
        inHumidity = humidity.2:9.HidekiTS04Packet
        inTemp = temperature.2:9.HidekiTS04Packet
```

#### Fine Offset sensor cluster with serial number 0026
```
[SDR]
    driver = user.sdr
    [[sensor_map]]
        windGust = wind_gust.0026.FOWH1080Packet
        outBatteryStatus = battery.0026.FOWH1080Packet
        rain_total = rain_total.0026.FOWH1080Packet
        windSpeed = wind_speed.0026.FOWH1080Packet
        windDir = wind_dir.0026.FOWH1080Packet
        outHumidity = humidity.0026.FOWH1080Packet
        outTemp = temperature.0026.FOWH1080Packet
```

To figure out the sensor identifiers, run the driver directly, possibly with
the `--debug` option. Another option is to run weewx with the logging options
for `[SDR]` enabled to display the sensors found by rtl_433, the sensor
identifiers used by weewx, and the sensors actually recognized by weewx.

```
[SDR]
    driver = user.sdr
    log_unknown_sensors = True
    log_unmapped_sensors = True
```

By default, the logging options are False.

## How to diagnose problems

First try running the rtl_433 application to be sure that it works properly:

```shell
rtl_433
```

If you know exactly which sensors you want to monitor, try the -R option to
reduce the clutter. For example:

```shell
rtl_433 -M time:unix -F json -R 9 -R 31
```

Once that is working, run the driver directly to be sure that it is collecting
data from the rtl_433 application:

```shell
PYTHONPATH=/home/weewx python /home/weewx/bin/user/sdr.py
```


## Environment

The driver invokes the rtl_433 executable, so the path to that executable and
any shared library linkage must be defined in the environment in which weewx
runs.

For example, with rtl_433 and rtl-sdr installed like this:

/opt/rtl-433/
/opt/rtl-sdr/

one would set the path like this:

```bash
export PATH=/opt/rtl-433/bin:${PATH}
export LD_LIBRARY_PATH=/opt/rtl-sdr/lib
```

Typically, this would be done in the rc script that starts weewx. If rtl_433
and rtl-sdr are install to /usr/local or /usr, then there should be no need
to set the PATH or LD_LIBRARY_PATH before invoking weewx.

If you cannot control the environment in which weewx runs, then you can specify
the LD_LIBRARY_PATH and PATH in the weewx-sdr driver itself. For example:

```
[SDR]
    driver = user.sdr
    cmd = rlt_433 -M time:unix -F json
    path = /opt/rtl-433/bin
    ld_library_path = /opt/libusb-1.0.20/lib:/opt/rtl-sdr/lib
    [[sensor_map]]
        ...
```

## libusb

I have had problems running rtl-sdr on systems with libusb 1.0.11. The rtl_433
command craps out with a segmentation fault, and the rtl_test command sometimes
leaves the dongle in a weird state that can be cleared only by unplugging then
replugging the dongle.

Using a more recent version of libusb (e.g., 1.0.20) seems to clear things up.

## Detecting new sensors

After you have run for a while, you might want to add new sensors to your
system. If you have more than two or three sensors, it can be quite a
challenge to pick through all the output when you run the driver directly.
This shows how to display only sensors that are detected but not yet part of
your weewx configuration.

First, shut down weewx so that you can talk to the SDR directly.

Then run the SDR driver directly, but tell it to print out information only
about sensors that you have not yet added to your weewx configuration:

```shell
PYTHONPATH=/home/weewx python /home/weewx/bin/user/sdr.py --config /home/weewx/weewx.conf --hide=out,parsed,mapped
```

As always, unless the sensor identifier is marked on the sensor itself, you
should turn on sensors one at a time, marking the outside of the sensor with
its identifier. Then you can turn on all the sensors and place them, using
the identifier on the sensor to distinguish which sensor is which when you
map them to database fields in your weewx configuration.

## Support for new sensor types

To add support for new sensors, capture the output from rtl_433. To capture
output, run the driver directly and hide known packets:

```shell
PYTHONPATH=/home/weewx python /home/weewx/bin/user/sdr.py --cmd "rtl_433 -M time:unix -F json" --hide parsed,out,empty
```

This should emit a line for each unparsed type. For example:

> unparsed: ['{"time" : "2017-01-16 15:44:51", "temperature" : 54.140, "humidity" : 34, "id" : 221, "model" : "LaCrosse TX141TH-Bv2 sensor", "battery" : "OK", "test" : "Yes"}']

If you are not comfortable writing your own parser, open a new issue with the output and some helpful person might write
the parser for you.