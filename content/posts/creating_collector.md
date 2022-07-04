---
title: "Creating Standalone Artifact Collector"
date: 2022-06-07T16:30:11+01:00
draft: false
comments: false
images:
tags:
  - DFIR
  - forensics
---

I got quite a few positive reactions to my last post about how to do deadhost collections, so I thought I wanted to follow it up with another post about how to create a standalone collector that can collect artefacts from a running host using Velociraptor.

[Velociraptor](https://docs.velociraptor.app/) is an awesome DFIR tool made to efficiently get visibility into endpoints. It can be run in a server/agent setup, essentially working as an EDR with thousands of hosts, and it can also be used as a standalone artefact collector.

A standalone collector is an executable that is handy in situations where Velociraptor or other kinds of collection tools are not deployed in the environment. It could be that a consultant responding to an incident wants to give the client sysadmin a quick way to collect artefacts from a suspicious host without tool deployment.

## Running Velociraptor in GUI mode

Velociraptor made it quite easy to create a collector without having to setup a lot beforehand. After downloading the Velociraptor executable from their [github page](https://github.com/Velocidex/velociraptor) we can run the executable using the `gui` parameter. This will give us the Velociraptor web page, as if we were running it in server mode.

```powershell
.\velociraptor-v0.6.4-2-windows-amd64.exe gui
```

![](/img/offlinecollect/20220603171710.png)

This will also automatically open up the default browser to the Velociraptor web page.

![](/img/offlinecollect/20220603172257.png)

We are now ready to create the collector.

## Creating standalone collector
Once the Velociraptor executable is running in gui mode, we will be able to use to use its capabilities to create a standalone collector.

To do this we will go to the *Server Artifacts* menu tab and press the small paper plain icon called *Create offline Collector*. This will open op a menu with the different default artefacts Velociraptor is capable of collecting from hosts.

![](/img/offlinecollect/20220603172949.png)

For this exercise I'll choose *Windows.KapeFiles.Targets* and *Windows.Memory.Acquisition*. KAPE is created after Eric Zimmermans KAPE (Kroll Artefact Parser and Extractor), and can collect a wide range of artefacts. Windows.Memory.Acquisition will use WinPmem to collect the memory of the machine.

After selecting  *Windows.KapeFiles.Targets* and *Windows.Memory.Acquisition* go to the Configure Parameters tab at the bottom. This will show the two selected artefact modules. Select thereafter the *Windows.KapeFiles.Targets* module, which will give a wide range of artefacts to collect. I will recommend to choose the *\_SANS\_Triage* artefact list because this gives a wide range of artefacts, and will cover most use cases.

![](/img/offlinecollect/20220603174026.png)


At the next tab *Configure Collection* there's multiple settings that can be set. Some of the more interesting once are the *Collection* Type and *Password*. Collection Type gives the possibility to:
- Create a zip file locally on the system 
- Upload the zip file directly to a Google Cloud Bucket
- Upload to an AWS bucket
- Upload to a SFTP server

This is really smart if we have ie. a timesketch setup, that automatically handles the collected artefacts being uploaded and creates a timeline in timesketch.

![](/img/offlinecollect/20220603174333.png)

In the *Specify resources* tab we'll be able to set a limit on the resources used by the Velociraptor collector. As a standard the CPU limit is set to 100% and max execution is 600 seconds. Since we are collecting memory also, this might take a while, so I would recommend setting the max execution time a bit higher eg. 900  or 1200 seconds

![](/img/offlinecollect/20220603174843.png)

Once done with the different configurations, we can go to the *Review* tab, check the configuration in JSON, and from there go to the *Launch* tab. Launch tab will check all prerequisites including downloading the WinPmem executable from the Velociraptor repository. The neat thing with this, is that Velociraptor will embed the executable in to the Collector executable, so we wont have to deal with multiple executable.

Once all is green on the *Launch* tab, we can click the newly create collection and go to the *Uploaded Files* tab as shown below. Here we'll be able to download the newly created collector executable.

![](/img/offlinecollect/20220603175002.png)

We can now copy this executable to the host we want to collect from and run it as administrator.

![](/img/offlinecollect/20220603175217.png)


The collector will open up a new window showing all the different files being collected, and close it self as soon as it is done collecting. After collecting we can then copy out the zip file to our investigator host or go to the uploaded file in our Google/AWS/SFTP bucket

![](/img/offlinecollect/20220603175305.png)

## Re-configuring the collector executable
Now, if we already created a offline collection executable, and we want to reconfigure the settings, we can dump out the configuration from the collector to yaml, reconfigure the settings in the yaml and then pack it into a new executable.

Extract configuration from standalone collector executable:

```powershell
.\Collector_velociraptor-v0.6.4-2-windows-amd64.exe config show > config.yaml
```

Once the yaml file is re-configured, we can repack the collector using the following command:

```powershell
.\Collector_velociraptor-v0.6.4-windows-amd64.exe config repack config.yaml collector_repacked.exe
```

We can also verify the configuration using the following command:

```powershell
.\collector_repacked.exe config show
```

