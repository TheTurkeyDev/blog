---
title: "TrueNAS Disappearing Files"
date: 2022-11-14T22:54:50-05:00
description: "I rebooted my TrueNAS server and all the files in one of my folders disappeared"
toc: true
tags: ["TrueNAS", "lost files", "Dataset", "folder", "files disappeared"]
---

## The Problem

I've started to rip movies from DVD's I have and save them to a Jellyfin server I've setup. While uploading files and organizing them on the TrueNas server, things locked up and after awhile I was forced to restart the server. After coming back online, I looked into the folder holding all my movies and discovered the folder was all but empty. The only thing inside of it was the very first movie I uploaded (in mkv format before I had concerted it to mp4 to save space). At first I thought I had corrupted my files somehow, but looking closer at TrueNas, the parent dataset still had the correct used space showing (just shy of 1 TiB), while the specific movies dataset within that, showed only 6 GiB used. Going into the TrueNas shell and navigating to the dataset also confirmed these findings as the parent folder sat just under 1 TiB in size while every folder inside of it was well under that mark, including the hidden xfs folder. All together they didn't even eclipse 100 GiB.

What gives? Well after a few hours of stressing (I did have a snapshot, but it was missing may movies and I REALLY did not want to have to re-rip them), I finally [came upon a post](https://www.truenas.com/community/threads/dataset-empty-after-reboot.43237/) mentioning the same symptoms and a comment spelling out what had happened for me 

```
You have a subdirectory called series, and you have a dataset called series. For some reason, your data got put into the subdirectory, not the dataset. Then when the dataset is mounted, the contents of the subdirectory can't be seen...
```

Before Jellyfin, I actually had setup Plex to try and originally had a parent dataset for plex and then within that dataset, I had a dataset for movies. After switching to Jellyfin I kinda abandon the concept of a sub dataset, but never fully removed it. At any rate it seems I must have made a folder called movies and also the dataset called movies. I'm not sure how it happened, but the movies I was adding to TrueNAS were being put into the movies folder and not the dataset, but after reloading the server, TrueNAS mounted the movies dataset overtop the movies folder that actually had all the movies in it, preventing me from accessing them

## My Solution

Luckily while all this sounds bad, the solution is not bad at all. I simply used the TrueNAS shell to move the folder away from the mounted dataset.
ex: `mv (path to dataset)/movies (path to dataset)/movies-copy` and then if you navigate into the new `movies-copy` folder all of the movies and files previously hidden under the mount are now visible! I don't fully know the logic why `cd` and everything else goes into the mounted folder, but `mv` moves the underlying folder and not the mount, but hey! I'll take it!

I did end up fixing all of these folders and datasets, so this shouldn't happen again, but on the off chance it does I now how this to help me fix it. Often complex and confusing issues have the simplest causes and resolutions.