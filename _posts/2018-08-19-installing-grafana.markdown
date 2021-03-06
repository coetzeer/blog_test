---
layout: post
title: Installing Influxdb and Grafana on FreeNAS
date: 2018-08-19
description: Installing Influxdb and Grafana on FreeNAS
img: grafana.png # Add image post (optional)
tags: [grafana, influxdb] # add tag
toc: true
---

# The many and various graphs of FreeNAS

{% include figure image_path="/assets/img/freenas_graphs.png" alt="The original freenas graphs." class="image-small image-right" caption="Ye Olde Freenas Graphs." %}

Out of the box, freeNAS comes with some system graphing tools. Theres the system graphs provided by the old freenas ui, the graphs in the monitoring section of the new UI, and the graphs provided by Netdata.

## The original graphs
The graphs that are provided in the original FreeNAS GUI that offer the familiar rrd-tool look and feel. My guess is that they are generated by collectd using the rrd-tool plugin. They're functional and do what they say on the tin: you can get great performance information on your CPU, memory, ZFS metrics, etc. On the negative side they are NOT sexy in the vain on Kibana or chart.js and they have a limited retention i.e they only go back a few hours.

## The new sexiness

{% include figure image_path="/assets/img/freenas_graphs2.png" alt="The spanky new freenas graphs." class="image-small image-right" caption="Ye Newy Fancy Freenas Graphs." %}

The new FreeNAS GUI ups the attractiveness stakes by replacing the RRD tool generated graphs with some slinky svg graphs, with nice period selection sliders and pastel colour pallet that would bring a tear to Steve Job's eye. They do however suffer from the same problem as the original graphs in that the graphing period is limted. There seems to be a maximum of about 10 minutes that you can go back, which, if you're a noob freeNAS administrator like me is not enough if you're trying to figure out what happens when your scheduled scrub runs.  

## Netdata

One of the best kept secrets in FreeNAS, however, is the NetData gui that, once enabled in the service menu, gives you an amazing set of graphs out of the box. Almost every conceivable metric is on offer, and it's a super tool to learn about how everything fits together. I've spent hours peering in the graphs seeing what happens to RAM if I make a pool with Dedup enabled, or watching the network traffic ebb and flow as 'The Great Move' happens for the third time... sigh.

BUT, as it comes, netdata has a bit of a drawback, and that's the retention again. It seems to only store the last 24 hours worth of metrics. This is by design although I'm not 100% sure of the reasons behind it. I think the reason for this is that the 24 hours of data seems to take about 600 Megabytes, so that would very quickly grow beyond a small SSD or flashdrive if you increased it to a month or a year.

Anyhow, messing around with the config on the freeNAS appliance is also discouraged as all bets are off when it comes to configuration files. You never know when you're going to meticulously craft the perfect configuration for your needs but have it break something in FreeNAS that you weren't aware of or, less disasterously, simply be rewritten at the next reboot. Perhaps tuning the retention time for NetData is something that folks at in the FreeNAS community would consider something worth of elevating to a GUI level parameter?

{% include figure image_path="/assets/img/freenas_netdata2.png" alt="The hot snot that is netdata." class="image-large image-centre" caption="Ye Uber Freenas Graphs: Netdata" %}

# Sending data to Graphite


{% include figure image_path="/assets/img/freenas_export_to_graphite.png" alt="Graphite writing sneakiness" class="image-small image-right" caption="Send your metrics to a safe home." %}

One last bit of FreeNAS configuration sugar that is easy to overlook is that it's VERY easy to send all the metrics collected by the collectd daemon built into the applicance to graphite writer. Tucked away in the System/Advanced menu is a field called 'Remote Graphite Server Name'.


As the name suggests, if this field is populated, metrics will be sent to the host specified using the a Graphite style http post. So an attractive solution to my above problems of metric retention is to set up a Graphite server somewhere. And this is exactly what I did. Almost.

# The wonder of Influxdb

I spent an evening attempting to set up graphite. I won't bore you with the details but the summary is that I gave up. Graphite is a couple of moving pieces: Carbon, the thing that collects the metrics, the Graphite web application implmented in Django that privides http access to your various time series data stores and whisperDB which is the implementation of the time series database that comes out of the box.

This collection of bits is, in itself, not a problem. In fact it gives you quite a bit of flexibility in a large deployment: you can cluster your Carbon collectors to scale horizontally, you can move the Django app off to a separate machine and you slot in different implementations of time series databases if whisperDB doesn't float your boat. Different storage options include: [ceresdb](https://graphite.readthedocs.io/en/latest/ceres.html) (built in), [kairosdb](https://kairosdb.github.io/) and [cyanite](https://github.com/pyr/cyanite) both based on cassandra, and some new fancy ones like [Kenshin](https://github.com/douban/Kenshin)

So while this ability to chop and change makes my OCD twitch like nervous tick accross a worried sheep (I MUST TRY ALL THE OPTIONS), it makes setting up Graphite a bit of a pain in the neck. That is most likely just me being a noob, but I gave up after 3 hours of fiddling with a multitude of property files.

The next night I installed Influxdb and I was done in 20 minutes (with the help of [this](https://www.rudimartinsen.com/2018/04/12/monitoring-freenas-with-influxdb-and-grafana/) awesome blog post).

__Step 1: Go to jail__

```bash
iocage create -n influxdb dhcp=on allow_sysvipc=1 bpf=yes vnet=on -r11.1-RELEASE
iocage start influxdb
```

__Step 2: Install InfluxDB__

```bash
iocage console influxdb
pkg install influxdb
sysrc influxd_enable=YES
service influxd start
```

__Viola!__

Test to see that it's listening to commands:

```bash
# you may have to install curl and nano and some other tools into the jail
pkg install curl wget nano
curl localhost:8086/query --data-urlencode "q=show databases"
```

__Step 3: create the databases__

We'll need 2 databases: one that will receive collectd data from the various rasperry pi's and other things that support collectd, and one that will receive data using the graphite receiver.

```bash
curl localhost:8086/query --data-urlencode "q=CREATE DATABASE collectd"
curl localhost:8086/query --data-urlencode "q=CREATE DATABASE graphite"
```

In order to make it a bit more useful, we now have to enable some of the fancy collectors sockets that influx offers.

Crack open the influx config at __/usr/local/etc/influxd.conf__

```ini
[[collectd]]
  enabled = true
  # bind-address = ":25826"
  database = "collectd"
  typesdb = "/usr/local/share/collectd"
  security-level = "none"

[[graphite]]
  # Determines whether the graphite endpoint is enabled.
  enabled = true
  database = "graphite"
  # retention-policy = ""
  bind-address = ":2003"
  protocol = "tcp"
  consistency-level = "one"

  templates = [
    "*.app env.service.resource.measurement",
    "servers.* .host.resource.measurement*",
    # Default template
    #"server.*",
  ]

```
__Step 4: Add the collectd typesdb__

In order to use the collectd collector in the above configuration, the collectd typesdb needs file needs to be installed. The easiest way to get this file is to install collectd.

```bash
pkg install collectd5
service restart influxd
```

Now, if you've made the changes mentioned in the section above, you should start seeing metrics being delivered to your shiney new influxdb.

```bash
root@influxdb:~ # influx

Connected to http://localhost:8086 version 1.5.0
InfluxDB shell version: 1.5.0
> use graphite
Using database graphite
> show series
key
---
arcstat_ratio_arc-hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_arc-l2_hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_arc-l2_misses,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_arc-misses,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_data-demand_data_hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_data-demand_data_misses,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_data-prefetch_data_hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_data-prefetch_data_misses,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_metadata-demand_metadata_hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_metadata-demand_metadata_misses,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_metadata-prefetch_metadata_hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_metadata-prefetch_metadata_misses,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_mu-mfu_ghost_hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_mu-mfu_hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_mu-mru_ghost_hits,host=freenas_home,resource=zfs_arc_v2
arcstat_ratio_mu-mru_hits,host=freenas_home,resource=zfs_arc_v2
cache_eviction-cached,host=freenas_home,resource=zfs_arc
cache_eviction-eligible,host=freenas_home,resource=zfs_arc
cache_eviction-ineligible,host=freenas_home,resource=zfs_arc
cache_operation-allocated,host=freenas_home,resource=zfs_arc
cache_operation-deleted,host=freenas_home,resource=zfs_arc
cache_ratio-arc,host=freenas_home,resource=zfs_arc
cache_result-demand_data-hit,host=freenas_home,resource=zfs_arc
cache_result-demand_data-miss,host=freenas_home,resource=zfs_arc
cache_result-demand_metadata-hit,host=freenas_home,resource=zfs_arc
cache_result-demand_metadata-miss,host=freenas_home,resource=zfs_arc
etc...


```

# The missing metric - hard drive temperatures

One of the things that I'm a bit paranoid about is the temperature of the hard drives in the Fractal Design case. They are all lined up next to each other, with two big fans blowing fresh air over them all the time, but I'm a noob when it comes to paying attention to hard drives, so I have no idea what's appropriate for operating temperatures, and I also want to be able to retrospectively look into what might have caused the drive to fail, should it ever actually fail.

And temperature is important, and completely missing from all of the 'off the shelf' collectd and freenas metrics. So I decided to try and roll my own.

Smartctl can give you a temparature readout of each drive. Grabbing this and parsing it with grepping and awking, allows me to harvest the temparature for each drive in degrees C.

This then has to be sent to the right place in influxdb.

This script below accomplishes those steps with some success:

```bash
#!/bin/sh

# /usr/local/sbin/send_hdd_temp_to_inflush.sh

smartctl -a /dev/ada0 | grep Temp | awk '{print "DiskTemp,component=ada0 value="$10}' \
  | curl -i -XPOST 'http://influxdb.home:8086/write?db=graphite' --data-binary @-
smartctl -a /dev/ada1 | grep Temp | awk '{print "DiskTemp,component=ada1 value="$10}' \
  | curl -i -XPOST 'http://influxdb.home:8086/write?db=graphite' --data-binary @-
smartctl -a /dev/ada2 | grep Temp | awk '{print "DiskTemp,component=ada2 value="$10}' \
  | curl -i -XPOST 'http://influxdb.home:8086/write?db=graphite' --data-binary @-
smartctl -a /dev/ada3 | grep Temp | awk '{print "DiskTemp,component=ada3 value="$10}' \
  | curl -i -XPOST 'http://influxdb.home:8086/write?db=graphite' --data-binary @-
smartctl -a /dev/ada4 | grep Temp | awk '{print "DiskTemp,component=ada4 value="$10}' \
  | curl -i -XPOST 'http://influxdb.home:8086/write?db=graphite' --data-binary @-
smartctl -a /dev/ada5 | grep Temp | awk '{print "DiskTemp,component=ada5 value="$10}' \
  | curl -i -XPOST 'http://influxdb.home:8086/write?db=graphite' --data-binary @-

```

The final step is to run that script at regular intervals. I decided to use the FreeNAS schedule GUI to set this up, rather than a crontab in a jail.  This keeps things nice and visible, and the config is in the right place (so it won't be lost next time the system updates). The only draw back is that the highest frequency that GUI allows is once a minute. I can live with that for time being, but it does make the graphs look a bit weird.
