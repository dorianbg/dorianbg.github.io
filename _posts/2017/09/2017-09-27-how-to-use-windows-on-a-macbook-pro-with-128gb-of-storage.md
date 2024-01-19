---
title: "How to use Windows on a Macbook Pro with 128GB of storage"
date: "2017-09-27"
categories: 
  - "misc"
---

In this post I will explain my laptop setup which I consider quite practical.

This is a quick highlight of my setup:

- 128GB 13" Retina Macbook Pro
- Windows 10 on a special Bootcamp partition
- Parallels Desktop on MacOs to run Windows 10 Bootcamp partition in a VM

 

### What exactly is your setup?

My laptop is Macbook Pro Retina 13" 2015 model with 8GB of RAM and 128GB SSD.

I also have a expansion SD card that brings in extra 64GB of very slow memory. This storage is mainly used for videos, music, pictures, documents. Almost all of the software I use is stored on the super fast SSD of Macbook.

The partitioning of the 120GB SSD is following:

- Have ~70GB of storage for MacOS
    - used for general programming - big data projects, machine learning, java programming, shell scripts...
- Have ~50GB of storage for Windows
    - used for mainly for SQL Server and Microsoft business intelligence software

As you can see this is the exact divide:

![Screen Shot 2017-09-27 at 21.12.48.png](assets/img/old_blog_post_images/screen-shot-2017-09-27-at-21-12-48.png)

 

### Why do you have a Bootcamp partition instead of just running a VM?

I use the Bootcamp feature of MacOS for 2 main reasons:

1. Using Windows in native mode is 2x faster than using in within a VM like Parallels. Some tools like SSDT (Visual Studio 2015) constantly crash on me when I run them within Parallels desktop
2. You can point Parallels to create a VM container from your Bootcamp partition! This is a genius feature that Parallels has and really gives you the most options. Eg. if you want to run a quick script in SSMS -> fire up the parallels VM, if you want to make a SSIS package -> reboot into Windows.

 

### How can I fit all my Windows stuff to only 50GB?

I use only 40GB of storage and have a SQL Server 2016 engine with all major services (SSAS, SSRS, SSIS), SSDT, SSMS and PowerBI.

But you have to cut some corners to achieve that. For instance choosing to install Polybase would add extra 4-5GB of storage to your my limited environment so I decided not to do that. You also can't have a large local database on your machine. But you can still install something like WorldWideImporters or AdventureWorks.
