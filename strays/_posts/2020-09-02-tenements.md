---
layout: post
title:  "Mineral Exploration Licence Visualisation"
date:   2020-09-02 18:00:00 +1000
category: strays
---

## Mineral Exploration
I've previously written that this blog won't probably have much to do with geology (or mining for that matter), but the other day I saw a [tweet](https://twitter.com/TasmanianLabor/status/1300308839875575810) on the #politas hashtag that got my interest piqued. It was a [media release](http://taslabor.com/exploration-levels-miles-below-ground/) from Labor in relation to declining levels of mineral exploration, which had been prompted by an [ABS release](https://www.abs.gov.au/AUSSTATS/abs@.nsf/DetailsPage/8412.0Jun%202020?OpenDocument) on exploration expenditure.

Now, I'm not interested in getting all political but I am somewhat interested in mineral exploration, having previously pegged a couple of tenements myself. I'm no longer significantly involved in the industry but I still pay attention when new tenements are advertised in the weekend papers, and I've noticed there doesn't *seem* to have been many new ones - really for a few years now.

## Expenditure vs Ground Held
Now there isn't necessarily a direct linear relationship between exploration expenditure and the amount of ground held under exploration licences, but I expect there is some correlation given that explorers are expected to do exploration, which is usually expensive. Of course the ABS already tracks expenditure and figures are available quarterly (hence the media release).

I was aware that I could get old [spatial tenement data](http://www.mrt.tas.gov.au/products/digital_data/historical_tenement_data) from Mineral Resources Tasmania and I had previously thought about looking at how it had changed over time. The tweet reminded me, so I figured it was time to actually do it. So here we are.

## Category 1
A quick note on exploration licences for those unfamiliar with them. There are six different categories of licence, as follows:

>Category 1:  metallic minerals and atomic substances;  
Category 2:  coal, peat, lignite, oil shale and coal seam gas;  
Category 3:  rock, stone, gravel, sand and clay used in construction, bricks and ceramics;  
Category 4:  petroleum products except oil shale;  
Category 5:  industrial minerals, precious stones, semi-precious stones;  
Category 6:  any geothermal substance.  

Licences can be held under multiple categories, and it's common to have both say categories 1 and 3. If you're a metals explorer and you want to develop, it's easier if you can also have the rights to 'quarry' type products on the area you're interested in. But generally the licence category that we think of as being associated with the hardrock 'mining industry' (as opposed to quarrying, or coal) is category 1 - i.e. metals. 

Consequently, the visualisation is for category 1 licences only and *not* the other categories, unless it is a multi-category licence that also includes category 1. Hopefully that's clear.

## Visualisation
So what I've put together is a short video showing quarter-by-quarter changes in the Category 1 exploration licences in Tasmania from 1970 to the present. Check it out:

<video width="630" height="543" controls>
  <source src="https://res.cloudinary.com/dqvgsqwop/video/upload/v1599047971/strays/aggregate_de1lzz.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
#### Visualisation

## How?
For those interested this was put together in [QGIS](https://qgis.org/en/site/) using the Time Manager plugin. Base imagery is ESRI World Imagery (Clarity) Beta. The individual frames were stitched with [FFmpeg](https://ffmpeg.org/).

## What Does it Mean?
Well, the video speaks for itself really. The total area of Tasmania under exploration licence is fairly low; it's somewhat lower than average over the last 50 years. But it's definitely been lower in the past than it is now; particularly in the early 1990s. I'm not passing judgement on whether that's good or bad; this is a presentation of the facts only. 
