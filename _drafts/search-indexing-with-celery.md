---
layout: post
title: "Search Indexing with Celery and Solr"
author: "Jason Johns"
categories: development
tags: [celery,text-search,solr]
image: arctic-2.jpg
---

_ The content here is adapted from a presentation I gave at the Maine Server Side and Engineering meetup group's Jan 2018 session _

If you or your workplace has a great amount of content available for user access, having it easily accessible via search is a major component of the overall user experience, because if a user can't find what they're looking for, what reason do they have to stick around or come back?  DegreeData found itself in this situation not long ago, and the experiences and lessons learned are recounted here.

## Background

DegreeData is a fully curated content archive for US college catalogs going back to 2007.  We have over 27,000 fully mirrored web catalogs and PDF documents from almost 4000 higher education institutions on our Amazon S3 buckets.  Rather than relying on web crawlers to willy-nilly archive content, we use trained data entry personnel and an internal web application to specifically target catalogs at individual universities across the US.  

Each catalog is ingested via 
