---
title: "Broken TinyMCE functionality for Optimizely CMS: image cannot be added through Insert/edit image"
date: 2023-03-10
---

Recently, we got a bug report from the customer suggesting that when Insert/edit image is used from TinyMCE, the Source is not updated after the image is actually chosen.

![Broken insert image with TinyMCE](/assets/images/broken-editor.png)

**Steps to reproduce:**
1. In CMS open any page or block that has TinyMCE 
2. Try to add image via TinyMCE’s "Insert/Edit Image" popup (using drag & drop is working properly)
3. Click on the search icon and select an image from the Assets

**Observed**: when the image is chosen, source field is not populated and as a result image will not be added.

![TinyMCE insert image source empty](/assets/images/source-empty.png)

While we were trying to find which TinyMCE plugin exactly was causing the error, we upgraded EPiServer.CMS.TinyMce from 2.13.4 to 2.13.8 and that alone was enough to fix the issue.

I have not been able to find this on the Optimizely World, so decided to share.
