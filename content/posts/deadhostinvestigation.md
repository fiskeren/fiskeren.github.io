---
title: "Deadhost Investigation and Super Timeline"
date: 2022-05-22T09:45:02+01:00
draft: false
toc: false
images:
tags:
  - DFIR
  - Deadhost investigation
  - forensics
---

I recently attended a [SANS 508](https://www.sans.org/cyber-security-courses/advanced-incident-response-threat-hunting-training/) course and got time to play around with [Velociraptor](https://docs.velociraptor.app/), which is an awesome DFIR tool made to efficiently get visibility into endpoints. It can be run in a server/agent setup, essentially working as an EDR with thousands of hosts, and it can also be used as a standalone artifact collector.

In this post, I'll try to explain how to use the Velociraptor executable as a artefact collector to quickly collect Windows artefacts from a dead host image. After collecting the artefacts, I will show how to create a super timeline using [Plaso log2timeline](https://plaso.readthedocs.io/en/latest/sources/user/Using-log2timeline.html) from the evidence collected.

For this blog post I've used an image from NIST - [Hacking Case](https://cfreds.nist.gov/all/NIST/HackingCase), and mounting the E01 image is done using the *Image Mounter* tool made by [Arsenal Recon](https://arsenalrecon.com/downloads/).

![](/img/deadhost/20220521204910.png)

The Velociraptor executable used is the standard exe downloaded from Velociraptors [Github](https://github.com/Velocidex/velociraptor).

## Collecting artifacts
The Velociraptor executable has multiple built-in collection modules that can be used to collect various artefacts from hosts. In this example I've chosen to use the Windows.KapeFiles.Targets artefacts collection method. This is created after Eric Zimmermans KAPE (Kroll Artefact Parser and Extractor), and collects a wide range of artefacts. I will be collecting artefacts defined by the built in \_SANS_Triage, Notepad and MemoryFiles artefacts lists because this collects *most* artefacts needed in an investigation. These lists are essentially collections of directory and file names that Velociraptor will search for on the specified host.

The command used us the following:

```powershell
.\velociraptor-v0.6.4-2-windows-amd64.exe -v artifacts collect Windows.KapeFiles.Targets --output TriageFile.zip --args Device="E:" --args KapeTriage=Y --args _SANS_Triage=Y --args Notepad=Y --args MemoryFiles=Y
```

Arguments used:
- `artifacts collect Windows.KapeFiles.Targets` - Select the collection method *Windows.KapeFiles.Targets*
- `--output TriageFile.zip` - Save the files collected to a zip file called TriageFile.zip
- ``--args Device="E:"`` - The drive letter the E01 is mounted to
- `--args KapeTriage=Y` - Collect KapeTriage files
- `--args _SANS_Triage=Y` - Collect files defined by the \_SANS_Triage list
- `--args Notepad=Y` - Collect files defined by the Notepad list
- `--args MemoryFiles=Y` - Collect files defined by the MemoryFiles list

Executing the commands creates the following output:

![](/img/deadhost/20220521210410.png)

After collecting the Veloricaptor executable has now created a file called TriageFile.zip

![](/img/deadhost/20220521211442.png)

## Super Timeline
Once the artefacts are collected, Plaso's log2timeline can be used to create a timeline from them. Giving an easy way to triage an image, identifying suspicious activity.

I will be using the docker version of log2timeline, where I mount my local folder to the docker instance as `/data`:

```bash
sudo docker run -v /path/to/evidence/:/data log2timeline/plaso log2timeline --storage-file /data/host.plaso /data/E
```

The command should give the output as shown below and create a file called *host.plaso*.

![](/img/deadhost/20220521214159.png)

The *host.plaso* file is the database file created by log2timeline, which can be exported to a CSV file, which can be read using ie. Eric Zimmermans [timeline explorer](https://ericzimmerman.github.io/#!index.md). 

Converting the host.plaso file to a CSV file is done using Plaso's psort.py tool.

```bash
sudo docker run -v /path/to/evidence/:/data log2timeline/plaso psort -o l2tcsv -w /data/timeline.csv /data/host.plaso
```

The command gives the output as shown below and creates the file *timeline.csv*.

![](/img/deadhost/20220521215255.png)


Once the CSV is created, it can be loaded in to a tool such as Eric Zimmermans Timeline explorer, which has a very handy preset of colour mapping to easily get an overview of execution, web history and much more. A screenshot of the CSV just created is seen below.

![](/img/deadhost/20220521215624.png)

## Conclusion

This small blog shows that triaging of a disk can be optimised using tools such as Velociraptor and log2timeline. Instead of running a timeline analysis of a whole disk image, Velociraptor can be used to collect only what is important to create a picture of what happened when and how.

Instead of running log2timeline across all files on a disk, which in some cases can take days, timeline analysis is only being run on the important files collected by the Velociraptor executable and thereby cut down the crunch time (wait time) drasticaly. 