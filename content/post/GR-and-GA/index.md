---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Difference between GlideRecord() and GlideAggregate()"
subtitle: ""
keywords:
- script
- methods
summary: ""
authors: []
tags: []
categories: []
date: 2020-01-08T13:51:26-06:00
lastmod: 2020-01-08T13:51:26-06:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image: 
  caption: ""
  focal_point: Bottom
  preview_only: true

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
<br>
As a newer developer, I tend to forget basic JavaScript methods and I find myself doing lots of Google searching.
In ServiceNow, there are plenty of methods you should know to become a fluent developer in the platform. GlideRecord and GlideAggregate are no exception.
<br>
<br>
Let's say you wanted to query all active incidents and then disable them by setting their active field to false. Easy, right? Well this is where GlideRecord comes handy:
<br>
<h6>GlideRecord</h6>

```js
var rec = new GlideRecord('incident') ;
rec.addQuery('active',true);
rec.query(); 
while(rec.next()) { 
  rec.active = false;
  gs.print('Active incident ' + rec.number = ' closed');
  rec.update(); }
```
<br>

But what if we wanted to just simply retrieve the number of records in a table? This is where we can use GlideAggregate:

<h6>GlideAggregate</h6>

```js
var count = new GlideAggregate('incident');
count.addAggregate('COUNT');
count.query();
var incidents = 0;
if(count.next()) 
   incidents = count.getAggregate('COUNT');
```

<br>
<h2>The Difference</h2>
<br>

I'm sure there are obvious differentiators by now, but what is really the difference between the two? Can't you just use GlideRecord to retrieve the sum of incident records? Short answer: No.
<br>

There are alternatives to using GlideAgregate, but the options are slim to none. You can either use the `getRowCount()` method from GlideRecord, or just use GlideAggregate.
Using GlideRecord to count rows of records will cause scalability issues as tables can grow over time. This method retrieves every record and counts them. GlideAggregate retrieves the data from the actual database that's built in to ServiceNow, which is much faster when it comes to performance.
<br>
<br>
_The GlideAggregate class is an extension of GlideRecord and allows database aggregation (COUNT, SUM, MIN, MAX, AVG) queries to be done. This can be helpful in creating customized reports or in calculations for calculated fields. [read more](https://docs.servicenow.com/bundle/jakarta-application-development/page/script/glide-server-apis/concept/c_GlideAggregate.html)_
<br>
<br>

The disadvantage to using GlideAggregate, is that you are unable to access details in a specific record. In this case, you would use `GlideRecord()` to manipulate/read the fields on any given record.
<br>
<br>
