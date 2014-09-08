---
layout: post
title: "Earthquake Map in JS"
image: /images/eq-data-center.JPG
date: 2014-08-22
---

![Site screencap]({{ page.image }})

[Go to live demo of the site]({{ site.url}}/earthquake-data-center) | [Source Code](https://github.com/ScottHacker/earthquake-data-center/blob/master/script.js)

This is a website in Javascript that plots earthquakes on a Google Map based on user search terms. It uses two API calls to geonames.org, one to it's place search service and another to it's recent EarthQuakes database. It also uses Google Maps API to plot the Earthquakes on a map with markers and info windows with additional data. All the code is in Javascript with a bit of JQuery, the site is done in HTML/CSS.