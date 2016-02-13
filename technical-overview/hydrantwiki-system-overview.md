---
id: 183
title: HydrantWiki System Overview
date: 2013-03-12T10:28:02+00:00
author: Brian
layout: page
guid: http://hydrantwiki.treegeckoproductions.com/?page_id=183
---
This first post on technical details looks at an overview of the primary system components.  I will delve into specific sub-systems in later posts.  The system is hosted entirely inside of Amazon&#8217;s Web Service (AWS) offerings.

[<img class="alignnone size-full wp-image-178" alt="SystemOverview2" src="http://hydrantwiki.treegeckoproductions.com/files/2013/03/SystemOverview2.png" width="880" height="560" />](http://hydrantwiki.treegeckoproductions.com/files/2013/03/SystemOverview2.png)

# **Components**

## **HTML5 Tag Collector**

The [HydrantWiki HTML5 Tag Collector](http://app.hydrantwiki.com/html5/index.html) runs on the phone platform.  It is written in HTML and javascript using the jQuery and jQueryMobile libraries.  It currently requires a connection to the servers to operate.  As each Hydrant tag is gathered it is pushed through the internet to the web services.

The following information is collected with each Hydrant Tag.

  * User (for attribution purposes)
  * Date and Time of the tag
  * Location of the Tag &#8211; The location is calculated by averaging 10 positions of less than 50 feet in accuracy.
  * Photo of the Hydrant

Future desired features

  * Hydrants around the user&#8217;s current location (In design)

## Web Services****

The Web Services are currently written using Microsoft .Net and are hosted on IIS. The long term goal is to move these services to utilize Node.js.  The web services create objects that are then sent to a AWS Simple Queue Service (SQS) queues.  Because of how DynamoDB is designed the cost of database write operations or 5 times the cost of read operations.  The queues are used to off load expensive CPU processes and to throttle the rate at which write operations can be completed. The web service write the photos out to a S3 bucket that is used to store all original files.

## Tag Review App

The [HydrantWiki Tag Review Web App](http://app.hydrantwiki.com/) allows the users perform most of the backend HydrantWiki operations.

The features of the Tag Review App are detailed by the type of user.

**Tagging Users**

  * Review their Hydrant Tags
  * View map of all hydrants in the system
  * Bulk download of hydrant data (in design and development). This data will also be generally available without login. 
      * KML
      * GeoJSON
  * View Site Metrics 
      * Top Taggers (Data from OSM and from HydrantWiki) will be utilized for this purpose. 
          * Weekly
          * Monthly
          * Annually
          * All time

**Tag Reviewing Users**

  * Review the tags of other users. The tags can be converted into a hydrant or rejected. Rejected tags are not deleted from the creating user. Only the creator is able to view these tags.

**Administrator Users**

  * Manage user rights levels

The Tag Review web app performs many of it&#8217;s operations directly to the DynamoDB directly so that the user experiences a predictable application behavior.

## AWS DynamoDB

DynamoDB is Amazon&#8217;s version of a NoSQL database. Each HydrantWiki object (Row) is stored as a series of name (column) value (cell) items. DynamoDB is designed to scale from a single user up to hundreds of thousands of users. The amount of read and write capacity is able to be turned up and down as the system grows or shrinks. Most tables in the HydrantWiki system are currently configured to cost between $0.60 and $1.50 per month for the read (between 1 and 5 items per second) and writes (between 1 and 2 per second).

## AWS S3

Amazon S3 is used to store all objects that are larger the 64k. 64k is the maximum size a DynamoDB row or a SQS queue message can be. By design the only objects that are currently larger than 64k are the photos of the Hydrants. There are two S3 buckets that are used for the system. One holds the originals and the other hold scaled down images that are served up. The system makes images available that fit in a 100&#215;100 box or 800&#215;800 box. These scaled down images are created by the Queue Manager so that the web server isn&#8217;t burdened by the memory and cpu requirements to create these files. These images are served directly from S3 and do not burden the HydrantWiki web servers.

## Queue Manager

The Queue Manager watches the AWS SQS queues for work to persist to the database or other server processing that is requried.

  * Logs system messages
  * Persists tags to the database
  * Updates session expiration timing
  * Sends emails to users for account verification

Additional queued processes are likely to be used in the future to minimize the required DynamoDB write rate.

## AWS MySQL RDS Database

The MySQL database has a single purpose &#8211; Geospatial Queries. DynamoDB does not have the ability to create a Geospatial Index and so it would require a complete table scan to calculate a geospatial database request. Only Hydrants are indexed since they are the only type of data that is currently capable of being queried without other constraints (like user).

## Hydrant Location KML

A batch program executes a complete Hydrant location data extract from the MySQL database. This operation will be used to provide all hydrant data out in a single file.

In the future more formats will be made available

  * ESRI Shape File
  * GeoJSON

## TileMill

[TileMill](http://mapbox.com/tilemill/) is an open source project that is supported by [MapBox](http://mapbox.com/). TileMill is a tool to render tiles from raw vector (and raster) data. TileMill loads the HydrantWiki lthe Hydrant data into it and renders a heatmap on a transparent background. Currently this runs off of the KML file, but based on information provide by MapBox this will be modified to run against a indexed file that has been reprojected to EPSG:3857. This will optimize the performance for the tile generation. My goal is to optimize the tile generation so that the hydrant heatmap can be generated as often as possible. Using this heatmap makes it possible to view where hydrants are located without having to render an enormous number of points while the user is trying to slide the map around.

The tile data is then loaded to the MapBox servers.

## [MapBox](http://mapbox.com)

The [MapBox javascript API](http://mapbox.com/mapbox.js/api/) is used to render the slippy map to the screen. The map has two layers the base layer is a standard set of map tiles provided by MapBox. The second layer is the transparent heatmap that displays for zoom levels 1 through 13. For zoom levels 14 and beyond the map display individual hydrant markers that are returned from the HydrantWiki web servers in a Comma Separated Value (CSV) format. The MapBox API is able to read this data string directly in and create the pushpin markers from it. The MapBox solution from TileMill to slippy map is one of the simplest to utilize that I have found.

The tiles for the slippy map are supplied by MapBox servers without having to impact HydrantWiki resoures.