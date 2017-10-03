---
title: How I built awesome package tracking app with React Native and Node.js in record time
layout: single
header:
  image: /assets/how-i-built-parcels/cover.png
  teaser: /assets/how-i-built-parcels/cover.png
  caption: "[**Parcels**](http://parcelsapp.com/en)"
---

I did not get timely updates or any tracking at all with all the hyped up apps like 17Track while tracking my packages from Aliexpress. 

I've tried multiple tracking apps over the years. None worked the way I wanted, didn't track many packages that were coming my way or were just overloaded with needless features.

## Solution

How hard could it be to build simplest possible [package tracking app](https://itunes.apple.com/us/app/apple-store/id1229071393?pt=1235501&mt=8)?

After all, I just need to parse postal company web site, and show a list of statuses. All that is needed is just a tracking number and app will figure out who delivers it and where to track it.

Below is brief high-level overview how I built app and backend, cracked captchas and used [CodePush](https://microsoft.github.io/code-push/). Write in the comments or tweet at me what you want me to cover in details.

## App

{:.center}
[![Parcels app](/assets/how-i-built-parcels/cover.png)](http://parcelsapp.com/en)
*Parcels iOS and Android app*

I decided to use React Native as I thought it will be faster to get beta version into [App Store](https://itunes.apple.com/us/app/apple-store/id1229071393?pt=1235501&mt=8). And idea to eventually release same app to [Google Play](https://play.google.com/store/apps/details?id=com.brightstripe.parcels) played role as well.

I released 1st version to App Store in 2 weeks and after that it took only 2 days to adapt app to [Android](https://play.google.com/store/apps/details?id=com.brightstripe.parcels). 

Amazing how easy it was, and I as a single developer can easily support app for 2 platforms and ship updates daily.

## Backend

I chose to build tracking API with Node.js as I've done some parsing recently with [cheerio](https://github.com/cheeriojs/cheerio) and like how easy and straightforward it was.

To keep Node.js API running smoothly, and support multiple processors I used [PM2](https://github.com/Unitech/pm2) process manager and still in awe how well it works!

## Initial courier and postal services

I started with Russian market in mind, and added support initially for Russian Post, 4PX, Singapore Post, China Post, Hong Kong Post, SF-Express, Yanwen, Cainiao. 

These are the most frequently used carriers for shipments from China to Russia and it proved enough to test a demand for my app.

{:.center}
![Small part of supported carriers](/assets/how-i-built-parcels/carriers.png)
*Part of supported carriers*

Now [Parcels](http://parcelsapp.com) supports 130+ postal and courier companies around the world and most importantly it knows a lot of inter-carrier connections and can track packages when they are handed over between carriers.

## Tracking API Architecture

API is a simple [express.js](https://expressjs.com/) server. Then are A LOT of regular expression patterns for various tracking numbers. 

{:.center}
![Tracking ID Patterns](/assets/how-i-built-parcels/regexp.png)
*RegExp Patterns*

Some tracking ids like DHL, Russian Post, Universal Postal Union (UPU) can be validated by checksum and that saves time by not tracking invalid numbers. All other tracking numbers are being tracked by elaborate tree of trackers both in origin and destination countries.

{:.center}
![National Post Tracking Rules](/assets/how-i-built-parcels/national-post.png)
*National Postal Service Tracking Rules*

For each carrier, postal or courier company I initially built custom scraper and parser using [request](https://github.com/request/request) and [cheerio](https://github.com/cheeriojs/cheerio). Nothing complicated download html or JSON, map HTML table cells or JSON fields to common format and return that JSON to [iOS](https://itunes.apple.com/us/app/apple-store/id1229071393?pt=1235501&mt=8) and [Android](https://play.google.com/store/apps/details?id=com.brightstripe.parcels) apps.

{:.center}
![Bpost Tracker](/assets/how-i-built-parcels/tracker.png)
*Bpost Tracker*

After about 30 custom coded trackers I started noticing patterns and decided to make I standard tracker that uses set of download & parsing rules in JSON.

{:.center}
![Template Tracking Rules](/assets/how-i-built-parcels/template-tracking-rule.png)
*Template Rules Tracker*

That way I've been able to 5-10x my speed of development and started to add 5-6 new trackers each day.

## App Architecture

React Native is God's send for such simple apps like [Parcels](http://parcelsapp.com/en). In 2 weeks I was able to build iOS app, backend, prepare all assets for Apple App Store, release and get app aprroved.

App is simple master-details view with FlatList for packages and FlatList for package statuses.

{:.center}
![Code Structure](/assets/how-i-built-parcels/app-code-structure.png)
*Code Structure*

I settled on Material design, as iOS 10 look grew old on me, and I wanted something visually simple and clutter free.

Reused same JavaScript code for carrier company detection by tracking number as in backend API.

Declarative JSX UI allowed me to quickly and easily iterate on the design.

### CodePush

One of the main reasons for going with React Native was to use [CodePush](https://microsoft.github.io/code-push/).

Ability to add new tracking service, fix bug, detect new tracking number pattern and immediately release an update is invaluable! It saved me so many times!

### Performance

I'm pushing the limits of React Native FlatList and ListView when people are tracking 90 packages at a time and app starts to lag and skip frames when scrolling. Need to dig deep into the problem.

Other than that, React Native is working wonderfully!

### Storage

I used [realm.js](https://github.com/realm/realm-js) to store tracking ids on device. Realm is really great when it works, and horrible when you break it accidentally with your update to the DB schema, when your app is used by thousands of people, it quickly adds up to nightmare.

Realm hadn't given me any problems on iOS, any crashes where due to my stupidity and thanks to CodePush I had an opportunity to fix crashes same day.

On Android though, for some reason, when you update schema version of Realm DB, and release CodePush update, it crashes apps, and you can't do anything about that. Only apologoze before users and rush an updated build to Google Play.

To Google's Play Market credit, its developer console is perfect, nicely designed, fast, and most importantly updates are going live in 30 minutes to 1-2 hours. So even if you screw up, you can quickly recover.

### Fastlane

I'm a fan of Felix Krause` [Fastlane](https://github.com/fastlane/fastlane) and used his [snapshot](https://github.com/fastlane/fastlane/tree/master/snapshot), [supply](https://github.com/fastlane/fastlane/tree/master/supply) and [deliver](https://github.com/fastlane/fastlane/tree/master/deliver) utils to save enormous amounts of time when making localized screenshots, or updating and uploading localized descriptions and release notes to App Store and Play Market.

## Website

I love building web apps with Ruby on Rails so that's what I used to build landing page for my app with help of Twitter Bootstrap.

{:.center}
[![Parcels landing page](/assets/how-i-built-parcels/landing-page.jpg)](http://parcelsapp.com/en)
*Parcels landing page*

## Challenges

First challenge I encountered when app started getting popular and Russian Post JSON api started acting wonky. It would return empty response 2 times in a row and 3rd time it will give results. I used brilliant [async.js](https://caolan.github.io/async/) library to asynchronously requery unreiable tracking web sites

### Server Blocking by IP address
Then they started blocking server by IP outright. I came up with idea to detect when IP blocking is in place and tell client apps to query tracking websites on their own. Then they POST query results for parsing and formatting to Parcels API and get nicely formatted JSON in response. That proved to work perfectly!

Then I added support for Push notifications and that required periodic tracking by server. Even though I built my tracking API to behave nicely and not flood tracking web sites with requests, they still blocked my server.

I found lists of free proxy servers and now API tracks through random proxy when server is being blocked. Proxy servers are dying every day though. I added automatic proxy list parser to always have a list of working proxies.

### CAPTCHAs

Most fun challenge is when tracking web site uses captcha. It's amazing how far breaking image based captchas has advanced. Simple image preprocessing like thresholding, bluring, noise removal and [Tesseract](https://github.com/tesseract-ocr) can solve most captchas, except Google reCAPTCHA.

{:.center}
![Sample CAPTCHAs](/assets/how-i-built-parcels/captchas.png)
*Captchas that [Parcels](http://parcelsapp.com/en) successfully breaks to track packages*

_I believe courier and postal tracking websites that use captchas have lazy and incompetent development teams who don't knows what they are doing. They intentionally make it hard for their clients to track packages they where paid to handle._

Parcels handles 30 000 monthly users who track at least 2 times a day, some people track simultaneously 70-90 packages and CPU load of my [Linode](http://linode.com) barely budges above 3%. Most of the load comes from solving captchas :-)

DHL, UPS, Fedex have excellent and fast tracking so it's definetely possible to build fast, scalable tracking solution.

Parcels doesn't support tracking for postal websites that use Google reCAPTCHA (like Ukraine Post recently) and point app users to respective websites for manual tracking.

## Fun Stuff

I often times keep logs console open to see how tracking is working and what new tracking numbers people are entering.

I started wondering, would not it be cool to see where in the world people are tracking packages with Parcels.

So just for fun I added [WebSockets](https://github.com/websockets/ws) support to API, and with help of [Leaflet.js](https://github.com/Leaflet/Leaflet) and (GeoIP)[https://github.com/bluesmoon/node-geoip] show people on a world map tracking packages in real time.

Check it out [here](http://parcelsapp.com/en)!

{:.center}
![Sample CAPTCHAs](/assets/how-i-built-parcels/live-tracking.png)
*Live Package Tracking*

## In conclusion

Thanks for reading it through! If you want me to go deep into how I built each individual piece of the app, just tweet or email me.

Now, download [Parcels for iOS](https://itunes.apple.com/us/app/apple-store/id1229071393?pt=1235501&mt=8) or [Parcels for Android](https://play.google.com/store/apps/details?id=com.brightstripe.parcels) and let me know what you think.
