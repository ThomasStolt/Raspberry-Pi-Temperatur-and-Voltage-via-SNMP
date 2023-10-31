# Raspberry-Pi-Temperatur-and-Voltage-via-SNMP

## Quickstart

Run this:

```
sudo echo "pass  .1.3.6.1.4.1.8072.9999.9999.1  /usr/bin/sh /usr/local/bin/snmp-cpu-temp
pass  .1.3.6.1.4.1.8072.9999.9999.2  /usr/bin/sh /usr/local/bin/snmp-voltage-core" >> /etc/snmp/snmpd.conf
sudo systemctl restart snmpd
sudo usermod -aG video Debian-snmp
```
and then download and copy both, the ```snmp-cpu-temp``` and the ```snmp-voltage-core``` script to ```/usr/local/bin```.

Test with:

```console
pi@raspberry-pi:~ $ snmpget -v 2c -r 0 -c public <your IP address> .1.3.6.1.4.1.8072.9999.9999.1
iso.3.6.1.4.1.8072.9999.9999.1 = Gauge32: 55306
pi@raspberry-pi:~ $ snmpget -v 2c -r 0 -c public <your IP address> .1.3.6.1.4.1.8072.9999.9999.2
iso.3.6.1.4.1.8072.9999.9999.2 = Gauge32: 1200
```

The result should look like this:

```console
pi@raspberry-pi:~ $ snmpget -v 2c -r 0 -c public <your IP address> .1.3.6.1.4.1.8072.9999.9999.1
iso.3.6.1.4.1.8072.9999.9999.1 = Gauge32: 45764
pi@raspberry-pi:~ $ snmpget -v 2c -r 0 -c public <your IP address> .1.3.6.1.4.1.8072.9999.9999.2
iso.3.6.1.4.1.8072.9999.9999.2 = Gauge32: 926
```

# Guide

## Purpose of this Guide

This is a short guide to customise the net-snmp agent by adding a few custom OIDs that can then be polled by standard SNMP tools (such as telegraf or mrtg). The custom OIDs that I have picked are:

```.1.3.6.1.4.1.8072.9999.9999.1``` for the CPU temperature and

```.1.3.6.1.4.1.8072.9999.9999.2``` for the CPU voltage

## Temperature

Copy the script ```snmp-cpu-temperature``` from this respository to your ```/usr/local/bin``` directory.

The script ```snmp-cpu-temperatures``` is using this command to aqcuire the CPU temperature:

```console
pi@raspberry-pi:~ $ cat /sys/class/thermal/thermal_zone0/temp
45764
```

PS: There is another approach, which is this: ```vcgencmd measure_temp```. This apparently measures the GPU temperature, which sits on the same chip. Either way is good, I have decided here to go for the ```cat /sys...thermal_zone0/temp``` command.

This will return a number that represents the core temperature in "milli-Celsius". As we will publish that value in SNMP as a Gauge and Gauge can only do integer numbers, we will take this as it is and don't do any math to it. We will account for a factor at the receiver end, such as grafana, otherwise we would lose precision here.

## Core Voltage

Copy the script ```snmp-voltage-core``` from this respository to your ```/usr/local/bin``` directory.

The script ```snmp-voltage-core``` script measures the core voltage, it uses vcgencmd command like so:

```/usr/bin/vcgencmd measure_volts core``` This will return the core voltage in Volt like so:

```console
pi@raspi-053:~ $ /usr/bin/vcgencmd measure_volts core
volt=0.8600V
```

| ❗❗ | There is one important caveat with this command: the user that is running snmpd is Debian-snmp -> which in turn will call the script -> which in turn will call the vcgencmd command! In order for that to work that user needs to be part of the usergroup "video". | ❗❗ |
| :---: | :---: | :---: |

Add that user to the group video using this command:

```
sudo usermod -aG video Debian-snmp
```

We need to convert that into milli-Volts, which is again best for our usage, as we will use Gauge here as well. The ```snmp-voltage-core``` does that for us.

The vcgencmd utility is well documented here: ```https://www.raspberrypi.com/documentation//computers/os.html#vcgencmd```. There are 4 different voltages that you can measure:

vcgencmd measure_voltage core - Core voltage
vcgencmd measure_voltage sdram_c - SDRAM Core Voltage
vcgencmd measure_voltage sdram_i - SDRAM I/O Voltage
vcgencmd measure_voltage sdram_p - SDRAM Phy Voltage

I am assuming, these are different parts of the Broadcom chip.

The ```vcgencmd commands``` will btw show you, what parameter that command knows.

## Net-SNMP Configuration

Before you start, make sure your snmpd is working correctly and you can access the agent from wherever you want to access it from.

We will use the ```pass``` directive of the net-snmp agent (install with ```sudo apt install snmpd```). At the end of your ```/etc/snmp/snmpd.conf``` add the following lines:

```
pass  .1.3.6.1.4.1.8072.9999.9999.1  /usr/bin/sh /usr/local/bin/snmp-cpu-temp
pass  .1.3.6.1.4.1.8072.9999.9999.2  /usr/bin/sh /usr/local/bin/snmp-voltage-core
```

What that does is that whenever someone requests those OIDs vi SNMP from this agent, the agent will start the associated script with the argument ```-g``` (stands for "get"). That script will return the following data:

```console
pi@rasperry-pi:~ $ /usr/local/bin/snmp-cpu-temp -g
.1.3.6.1.4.1.8072.9999.9999.1
gauge
46738
```

This is what the net-snmp agent needs in order to know, what to send out for this request.

Analogue for the CPU voltage, the net-snmp agent calls ```/usr/local/bin/snmp-voltage-core``` when the second OID is queried. That script will return the following data:

```console
pi@raspi-053:~ $ /usr/local/bin/snmp-voltage-core -g
.1.3.6.1.4.1.8072.9999.9999.2
gauge
860
```

which is the proper data it needs to know what to send back to the requesting entity.


You can test this on your command line like so:

```console
snmpget -v 2c -r 0 -c public <your IP address> .1.3.6.1.4.1.8072.9999.9999.1
```

## Troubleshooting

When trying to request the OID with the snmpget command, you get an error message saying this:

```console
pi@raspberry-pi:~ $ snmpget -v 2c -r 0 -c public <your IP address> .1.3.6.1.4.1.8072.9999.9999.1
iso.3.6.1.4.1.8072.9999.9999.3 = No Such Object available on this agent at this OID
```

- Have you restarted your snmpd? (sudo systemctl restart snmpd)
- Double check your entries in your /etc/snmp/snmpd.conf file.
- Sometimes the net-snmp agent is lagging in its responses. If you change something and it doesn't immediately respond correctly, just restart the snmpd and wait 10 seconds or so.

When trying to request the voltage with the snmpget command, you get a 0 value:

```console
pi@raspberry-pi:~ $ snmpget -v 2c -r 0 -c public <your IP address> .1.3.6.1.4.1.8072.9999.9999.2
iso.3.6.1.4.1.8072.9999.9999.2 = Gauge32: 0
```

- this happens, if the user that is running the snmpd is not part of the group "video". Add that user to the group using this command:

```
sudo usermod -aG video Debian-snmp
```

---

Do leave a comment if this was helpful for you!!

:)




