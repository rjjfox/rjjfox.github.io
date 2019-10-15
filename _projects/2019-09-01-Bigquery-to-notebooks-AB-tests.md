---
layout: post
title:  "End-to-end AB test calculator in Jupityer Notebooks"
languages: Python and BigQuery 
date: 2019-09-01 15:00:00 +0700
---

Whilst I was at Eurostar, our success metric tracking for AB tests came via Google Analytics. Often we needed to use BigQuery to pull out greater granularity in the data. Once we had the data, we'd parse that into Jupyter Labs/Notebooks and run the appropriate hypothesis tests there.

<!--description-->

Why we didn't use a testing platform for our analysis is because the tool used changed a few times while I was there as our technical needs changed i.e. moving to react, client-side to server side. Therefore this meant that we wanted to ensure that our implementation was testing tool agnostic.

<!-- TODO Explain why we used BigQuery and our own significance calculator instead of a testing platform -->
<!-- TODO Explain how this is the first real project -->
<!-- TODO Photo of the BigQuery implementation and a short explanation of how to do this -->
<!-- TODO Small snippet of the code and photo of the graphs -->
<!-- TODO Small explanation of the tests used -->
