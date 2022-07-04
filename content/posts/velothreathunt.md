---
title: "Velociraptor Threathunting - Quick Introduction"
date: 2022-07-04T11:30:36+01:00
draft: false
toc: false
images:
tags:
  - DFIR
  - Threathunting
---

In my previous posts I've shown how to use Velociraptor to extract artefacts from dead hosts, but the real strength of Velociraptor is in its capability to do searches across multiple hosts for artefacts. This is really useful when responders are trying to determine the questions such as "Are there baddies in the network?", "How many machines have executable X on it?", "Did user X log on to system Y?" etc. In this blog post I will try to give a quick introduction to how to use Velociraptor to answer those kind of questions.
## Quick intro to VQL
All queries Velociraptor does are done using the Velociraptor Query language or VQL, which is very much like a simpler version of SQL, and therefore also something most people will be able to read and learn very easily. Take the example below - this query will take all columns from the `Info()` artefacts and show it to the responder.
```sql
SELECT * FROM info()
```
The output will be something like the following:

![](/img/velothreathunt/20220704115223.png)

If you did this over multiple machines using the 'Hunt Manager' in Velociraptor, you could expand the query to get an overview of how many different versions of OS' you have. To do this, We will create a hunt using the `Generic.Client.Info` artefacts, and query that across all machines. Once its done, the information received can we seen in the 'Notebook' tab of the hunt:

![](/img/velothreathunt/20220704120815.png)

Now, this will give a table with a row for each machine, which can be handy, but what if there is 1000 endpoints in the environment, the table will probably not be that easy to get an overview off. Therefore, lets edit the VQL statement to make it a bit easier to read. Do that by pressing the small edit button in the top of the notebook. The default query will probably look something like this:

```sql
/*
# Generic.Client.Info/BasicInformation
*/
SELECT * FROM source(artifact="Generic.Client.Info/BasicInformation")
LIMIT 50
```

If we look at the statement, we will see that the data source is not `info()` but instead `source()`, that's because it comes from a hunt collecting artefacts from multiple machines, and not a single run on a single machine.

Now, to get an overview, lets change the VQL statement to:
```sql
SELECT OS,Platform,PlatformVersion,count() AS Count FROM source()
GROUP BY OS
```
The output is that we now have a row per OS version an a count of how many systems that have this OS:

![](/img/velothreathunt/20220704122946.png)

As you can see, the VQL language is quite powerful, and will give the responder the ability to quickly collect data or to get an overview of data collected from hosts. In fact, all hunts in Velociraptor is actually VQL statements made to query systems for information.
## Installing artefacts from 'Artifact Exchange'
There are a lot of artefacts already build in to Velociraptor, but a really neat feature of the whole Velociraptor universe is the [Artifact Exchange](https://docs.velociraptor.app/exchange/). The exchange is a collection of artefacts contributed by the community. This means that if you are using Velociraptor and make a really awesome query, that are not already part of the normal system you can submit it to the exchange and share with everyone else so they can benefit from it, and find even more bad stuff in environments.

The 'Artifact Exchange' needs to be imported in to the Velociraptor server. That can be done by running the artefact `Server.Import.ArtifactExchange` on the Velociraptor server itself - Go to the 'Server Artifact' menu and create a new collection. From the drop down choose the `Server.Import.ArtifactExchange` artefact and press launch:

![](/img/velothreathunt/20220704125408.png)
![](/img/velothreathunt/20220704125501.png)

All artefacts from the exchange are now imported in to the server and can now be used in hunts.
## Hunting
Now that we have scratched the surface of VQL and how to use it in hunts, lets dig in to some of the more power usages of hunts and VQL in Velociraptor. This section is a collection of different hunts that I have found useful, and would like to share.

### Finding renamed executables
One thing we often see in cases is the usage of psexec or use of [LOLBINs](https://lolbas-project.github.io/), example: use of `rundll32.exe` to run a dll on the host. These methods are sometimes block by security programs to prevent threat actors from using them to run arbitrary code on the systems, and the threat actors therefore often find clever ways to circumvent detection. One of the ways is to copy the executable to another place.

Example, lets say a threat actor is avoiding detection by renaming `rundll32.exe`  to `x.exe`, and moving the renamed file to `C:\tmp`. To find that using Velociraptor, we can use the artefacts called `Windows.Detection.BinaryRename`, which will search for executable and compare the binary metadata up against a predefined list (That you can add and remove from):

![](/img/velothreathunt/20220704131544.png)

Results:

![](/img/velothreathunt/20220704131930.png)

### Finding lateral movement
A key question in almost all investigations and hunts is: 'where did the threat actor go?', and to answer this (or get indication off) we can use the artefact `Windows.EventLogs-RDPAuth`. 

Even though the name indicates that Velociraptor will only search for RDP authentications, if we look a bit closer at the artefact, we will see in the description that it actually also looks at common logons such as 4624 type 3,7 and 10, which is not exclusively used for RDP authentication.

As you can see, the output can be very noisy and hard to get an overview over:

![](/img/velothreathunt/20220704133227.png)

To narrow down the output, lets edit the VQL statement:
```sql
SELECT EventTime,Computer,Channel,EventID,UserName,LogonType,SourceIP,Description,Message,Fqdn FROM source()
WHERE ( // excluded logons of the user on their own system
(UserName =~ "suspect1" AND NOT Computer =~ "suspect1workstation") 
)
AND NOT EventID = 4634 // less interested in logoff events
// Remove DCs, exchange and fileserver, since this is expected to have alot of logons
AND NOT (Computer =~ "dc" OR Computer =~ "exchange" OR Computer =~ "fs1") 
ORDER BY EventTime
```

The output should be a more compact list of logons to other hosts from the patient zero user account, thereby giving the responder easier overview of where to investigate next.

### Getting a list of running processes
An important thing to always have in mind when doing threathunts and investigations is 'How does normal look like?'. Its always tough to answer that question as a consultant coming in to an environment, but Velociraptor can help in getting an overview of what's running on the systems.

To get a list of running processes we can use the artefact `Windows.System.Pslist`.  The output will be quite big, and it will therefore make sense to do some sorting and stacking on the output.

The following VQL query takes the output from the hunt, and only shows the unsigned binaries running on the systems:
```
SELECT Name,Exe,Commandline,Hash.SHA256 AS SHA256,Authenticode.Trusted,Username,Fqdn,count() AS Count FROM source()
WHERE Authenticode.Trusted = "untrusted" //show only unsigned binaries
AND NOT Exe = "C:\\Path\\to\\whitelisted\\binary"
//Stack for prevalence
GROUP BY Exe
ORDER BY Count
```
### Finding files across the network
The last artefact I would like to show is how to search for files. This can be useful if we already have a list of known IOCs with filenames that we would like to search for, or it could be we are just hunting for 'weird' files. To search for files we can use the `Windows.Search.FileFinder`  artefact.

As example lets do a search for DLL files in weird locations, that might help us find bad stuff. For the SearchFileGlobTable, we will add locations to look for:

```
C:\Windows\Tasks\*.dll
C:\Windows\Temp\**\*.dll
C:\*\*.dll
C:\*.dll
```

![](/img/velothreathunt/20220704140415.png)

The output can be very noisy, and we will therefore need to do some sorting in the VQL statement, which should make it a little bit easier to read:
```sql
SELECT Fqdn,FullPath,MTime as Modified,BTime as Created,IsDir, count() as Count FROM source() WHERE IsDir = 'false'
GROUP BY FullPath
ORDER BY Count
```
## Summary
As you can see, the Velociraptor agent can be used to do very power full and quick search the whole host landscape, and thereby making it easier for threat hunters and the like to find things that are out of the ordinary.

I for more information about how to do hunts and creating queries in VQL, please see the Velociraptor [documentation](https://docs.velociraptor.app/).

## Credit
Sources for this blog post comes from:

- Stephan Berger [twitter](https://twitter.com/malmoeb)
- Matthew Green [twitter](https://twitter.com/mgreen27)
- Eric Capuano's awesome Velociraptor [video](https://twitter.com/Recon_InfoSec/status/1538216506483478528?s=20&t=v0fV-l1L9bH8PWKhQXdnKg) and the [Velociraptor notebook](https://gist.github.com/ecapuano/daee6f3704273c2c8b527f522c1725db)


