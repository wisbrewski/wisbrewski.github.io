---
layout: post
title:  "Phantom of the Opera Browser"
date:   2018-08-04 19:00:00 -0400
categories: dfir 
---
## Introduction

The other day I came across the Opera web browser.  I haven't heard about them in quite a while.  Years ago they used to be the browser that brought new features to the mainstream.  If memory serves me right, they were the first browser to introduce tabbed browsing.  I decided to have a look at the feature list for Opera 54 to see if there is anything new and exciting about the latest version of Opera.  There were two things that I found interesting:

1. Opera no longer uses their own web engine. The browser is now based on the Chromium engine 
2. Opera comes with a built in VPN feature

These two points were of interest to me from a digital forensic point of view. Being based on Chromium, I was curious whether or not I could analyze Opera artifacts using the same tools that are used for analyzing Chrome artifacts.  The VPN feature was of interest as I'm always interested in ways that users may hide their activity on a system.  Opera provides the VPN feature to enhance your browsing privacy and to hide your browsing from online trackers.  Does it also hide the activity from local investigators?

## What's the challenge?

The main questions that I was looking to answer are:

1. Can Chrome investigation tools be used to analyze/investigate Opera usage?
2. Does the VPN feature leave browsing history behind on the local system for investigators to discover?
3. Is there any evidence locally that the user has the VPN feature enabled?

## What's the plan?

To test Opera I used a fresh updated install of Windows 7 in a virtual machine with a few extra analysis tools installed.
Additional tools:

* Fiddler - web debugging tool(local web proxy)
* Process Hacker - process analyzer

To answer the two main questions that I was interested in, I followed the following steps:

1. Install Opera with default settings
2. Browse to a handful of websites using the default Opera settings
3. Take a copy of all Opera profile files
4. Turn on VPN mode
5. Browse to a handful of different websites using VPN mode
6. Take a copy of all Opera profile files
7. Turn off VPN mode
8. Browse to a handful of further different websites with VPN mode off
9. Take a copy of all Opera profile files
10. Check browser history after VPN test
10. Compare all file changes that occurred between each test
11. Check proxy logs to see what traffic is observed when VPN is OFF vs when VPN is ON

## What is in the History?
With the VPN feature turned on, I wanted to see what historical artifacts are left behind regarding the websites that the user may have visited.  I wanted to see if the VPN feature works like a traditional VPN, and only hides information from the network or if it works more like an Incognito browser mode, and hides visited websites from the local browsing history as well.  

My first thought was that I should be able to use Chrome forensic tools to analyze Opera, as version 54 of Opera is based on Chromium.  Unfortunately I kept getting error when I tried to analyze Opera using my favourite Chrome analysis tool, Hindsight.  Plan B was to fall back to the trusty [SQLITEBrowser](https://sqlitebrowser.org/ "SQLiteBrowser"). 

In Opera, just as in Chrome, browsing history is stored in a SQLite database file called History. The file can be found in the users profile at:
{% highlight conf %}
C:\Users\$username\AppData\Roaming\Opera Software\Opera Stable\History.
{% endhighlight %}

I opened up the History sqlite database using SQLiteBrowser and analysed the 'urls' table.  Scrolling to the bottom of the 'urls' table, I can see the URL's that were visited while VPN was turned on.  In this case, Twitter.com, Yahoo.com, and Flickr.com

![Opera History]({{ "/assets/operahistory.png" | absolute_url }})

This shows that the Opera VPN feature does not work similar to Incognito mode in other browsers.  Browsing history is available to investigators

## What do we see on the network?

The Opera VPN mode doesn't have an impact on the browsing history being saved, but what effect does it have from the network traffic point of view?

I used the [Fiddler](https://www.telerik.com/fiddler "Fiddler") web debugging proxy to simulate the use of a corporate internet proxy. 

### Browsing with VPN off
When browsing the internet with the VPN feature turned off, Fiddler clearly sees the user visiting Google and Facebook.

![Fiddler Screenshot]({{ "/assets/NOVPNBrowsing1.png" | absolute_url }})
![Fiddler Screenshot]({{ "/assets/NOVPNBrowsing2.png" | absolute_url }})

### Browsing with VPN on
When browsing with the VPN feature turned on, Fiddler no longer sees the users browsing activity.  For this test, the user browsed to yahoo.com, twitter.com, and flickr.com. The only web traffic showing up in the Fiddler log is to sitecheck.opera.com.

![Fiddler Screenshot]({{ "/assets/YESVPNBrowsing.png" | absolute_url }})

### Browsing with VPN once again turned off
When browsing with the VPN feature turned off again, Fiddler once again can see the users browsing activity.  In this case the user browsed to github.com and wikipedia.org

![Fiddler Screenshot]({{ "/assets/NOVPNBrowsing3.png" | absolute_url }})
![Fiddler Screenshot]({{ "/assets/NOVPNBrowsing4.png" | absolute_url }})

## Evidence of feature use?

The last thing I wanted to confirm is whether or not the Opera profile contains any evidence that the user is using the VPN feature while browsing.

After reviewing the various JSON configuration files and SQLite databases found in the Opera profile directory, the "Preferences" configuration file has the details I'm looking for.

The "Preferences" file is a configuration file in JSON format.  The section we are interested in is called "freedom".  The section looks like this prior to the VPN feature being used for the first time:

{% highlight json %}
"freedom": {
        "proxy_switcher": {
            "bytes_transferred": "0"
        }
    },
{% endhighlight %}

Once the VPN feature is turned on, the 'enabled' flag is set.

{% highlight json %}
"freedom": {
        "proxy_switcher": {
            "bytes_transferred": "0",
            "enabled": true,
            "last_ui_interaction_time": 1533175512.638015,
            "stats": {
                "last_date_stored": "13177569600000000",
                "values": [
                    "4079234"
                ]
            },
            "ui_visible": true
        }
    },
{% endhighlight %}

Once the VPN feature is turned back off, the 'enabled' flag is set to **false**.  This means that if the 'enabled' flag is set to **false**, the user has used the VPN feature at least once.  
The 'last_ui_interaction_time' value is the timestamp of when this setting was toggled.  
The 'bytes_transferred' value should provide details regarding how much data was transferred over the Opera VPN but for some reason during my tests the value never incremented.  This may require some more testing.  
The 'last_date_store' value doesn't make much sense either.  The current value translates to October 4th, 2011.

{% highlight json %}
"freedom": {
        "proxy_switcher": {
            "bytes_transferred": "0",
            "enabled": false,
            "last_ui_interaction_time": 1533175730.769898,
            "stats": {
                "last_date_stored": "13177569600000000",
                "values": [
                    "4886155"
                ]
            },
            "ui_visible": false
        }
    },
{% endhighlight %}

## Summary of Findings
Looking back at the original questions we wanted to answer.

**Can Chrome investigation tools be used to analyze/investigate Opera usage?**

In theory it should be possible to use the same tools to investigate Opera usage as investigators use to investigate Chrome.  However, my initial tests of using Hindsight(version 1.5 and 2.0) have failed.  This may require some extra testing with possibly other Chrome tools.

**Does the VPN feature leave artifacts behind on the local system for investigators to discover?**

My testing shows that the VPN feature only hides user activity from network analysis.  All browsing is performed through the Opera VPN tunnel so any web filtering proxies that are used in the environment will not be able to see the users activity.  It could be possible to identify and possibly block Opera VPN usage at the proxy level by looking for sitecheck.opera.com.

All browsing activity while using the Opera VPN feature is still logged in the local browser history database.  Investigators can easily review this activity using an SQLite tool like SQLiteBrowser. 

**Is there any evidence locally that the user has the VPN feature enabled?**

The VPN feature toggle is logged in the 'Preferences' JSON file located in the users Opera profile path.  Investigators can tell if the user is currently using the VPN feature and when the last time was that the VPN feature was turned toggled

## Conclusion
Opera's new VPN feature can hide browser traffic from your network controls, but standard endpoint forensic techniques are unaffected. 
