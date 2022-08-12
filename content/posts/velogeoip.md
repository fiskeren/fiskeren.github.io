---
title: "Geolocating IP addresses in Velociraptor"
date: 2022-08-12T17:28:43+01:00
draft: false
toc: false
images:
tags:
  - DFIR
  - Threathunting
  - geoip
---

I read the [blog post](http://www.mashthatkey.com/2022/08/velociraptor-playground-2022-08-02.html?m=1) by Carlos Cajigas and saw his [YouTube](https://www.youtube.com/watch?v=DMj0pU6kYvg) video demonstrating some really neat Velociraptor tricks including usage of the build in `geoip()` function, which can be used to enrich artefacts in Velociraptor hunts. This inspired me to write this small blog post where I'll quickly run over how to use `geoip()`.

This could be useful if you know hosts normally only connects to IPs located in the US, Ireland and UK and want to spot that specific process that connects to something else, or if you are on the lookout for processes connecting to hosts in a specific country etc - insert your own use case :-)
## Collecting netstat information from hosts
In Velociraptor there's an artefact called `Windows.Network.Netstat`, which pulls an overview of connections done by Windows hosts  to any IPs, just as if we had opened a command line prompt and typed in `netstat` on a host. That way we can see what the hosts actually connects to at the moment. Below is a picture of the artefact in Velociraptor.

![](/img/velogeoip/Pasted_image_20220812193744.png)

Once the hunt has run on the host(s). The output should look like the image below, which is multiple pages long. 

![](/img/velogeoip/Screenshot_2022-08-12_19-40-25.png])
## Geo locating remote IPs
from the picture above we can see that the `netstat` output shows `Raddr.IP` which is the remote IP or the IP of the server that the host is connecting to.

Looking at an IP and telling where that IP is located in the world can be hard, so to make that easier we can use the GeoLite2 Free database from [Maxmind](https://www.maxmind.com/en/accounts/current/geoip/downloads). This is a database that can match an IP up against ie. country, city or ASN number. In this blog I am going to use the `GeoLite2-City.mmdb` database.

With the database downloaded from GeoLite2, we can go ahead an edit the `Windows.Network.Netstat` artefact output.

![](/img/velogeoip/Screenshot_2022-08-12_19-50-52.png])

```sql
SELECT count() AS Count,Timestamp,Pid,Name,Status,`Laddr.IP`,`Laddr.Port`,
geoip(ip=`Raddr.IP`,db='C:\\GeoLite2-City.mmdb').country.names.en AS Country,`Raddr.IP`,`Raddr.Port`,Fqdn
FROM source()
WHERE Status =~ "ESTAB"
AND NOT Country =~ "United States"
GROUP BY Country
ORDER BY Count
```

The VQL statement above grabs the `Timestamp`, Process ID (`Pid`), Name of the process(`Name`), `Status` (example: established, connecting), Local IP(`Laddr.IP`), Local port (`Laddr.Port`, Remote IP(`Raddr.IP`), Remote port(`Raddr.Port`) and the host name of the computer (`Fqdn`, and then enriches it using the function caled `geoip()` that fetches the mmdb GeoLite database we downloaded. It then queries the remote IPs up against it to get a country name, looks for only established connections, filters out countries that are not the US and then orders it by how many connections there are.

The query gives the following output.

![](/img/velogeoip/Screenshot_2022-08-12_19-57-20.png)
![](/img/velogeoip/Screenshot_2022-08-12_20-07-46.png)
As seen above, there are a couple of connections to Ireland, some to Denmark and then some to the UK. This could give a really easy overview off the different countries and IPs that the hosts in the environment are connecting to.
