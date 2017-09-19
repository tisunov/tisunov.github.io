---
title: How I built awesome package tracking app with React Native and Node.js
layout: single
header:
  image: /assets/how-i-built-parcels/cover.png
  teaser: /assets/how-i-built-parcels/cover.png
  caption: "[**Parcels**](http://parcelsapp.com/en)"
---

I've struggled with tracking my packages from Aliexpress. I've tried multiple tracking apps over the years, Aliexpress own tracking site, Cainiao and 17Track and numerous others. 

None worked the way I wanted, didn't track many packages that were coming my way or were just overloaded with needless features, belts and whistles.

## Solution

I thought how hard could it be to build my own app and in the mean time learn React Native.

After all, I just need to parse courier or postal company web site, nicely format it as JSON and pass it to mobile app.

## Backend

I chose Node.js as I've done some parsing recently with [cheerio](https://github.com/cheeriojs/cheerio) and like how easy and straightforward it was.

For Node.js process mananger I used [PM2](https://github.com/Unitech/pm2) and still in awe how well it works, zero complaints!

## Initial courier and postal services

I started with Russian market in mind, and added support initially only for Russian Post, 4PX, Singapore Post, China Post, Hong Kong Post, SF-Express, Yanwen, Cainiao.

These are the most frequently used carriers for shipments to Russia and it proved enough to test a demand for my app. 