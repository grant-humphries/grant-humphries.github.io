---
layout: post
title: Building an ArcGIS Network Dataset with Python
___

## Building an ArcGIS Network Dataset with Python

Beginning about three years ago I inherited a project that had been undertaken at the request of the planning department which aims to identify the amount of real estate development that has occurred near MAX stops due to their line’s creation.  In years past the work had been done essentially by hand and as I made my intermittent updates I had been working to fully automate it.  As of my last pass there was just one piece of the process left that I had been unable to script: creating what ESRI calls a Network Dataset.

A Network Dataset is a routable network that can be configured to have various impedances and is derived from a street centerline file (in this case OpenStreetMap data).  I use this network to generate isochrones that illustrate the areas that can be reached from the MAX stations via the street and trail network with a half mile walk or less.  The `arcpy` python library that I generally use for scripting with ESRI products does not have the capability to create these, so I thought I was out of luck until I came across this post on GIS Stack Exchange.

#### Notes for blog post
* ctype and new_obj functions don’t do much beyond making the comtype function calls less verbose and more VBish, but they’re called so many times that there is meaningfully savings
* the code that I wrote in python mimics vb.net
* writing this code gave insight into the way that vb.net and c# use objects and and interfaces to build things
* Used VBA for network attributes, executes much faster than python
* Seemly random string for arcobjects array object
* The downside with work with these tools is the nonsensical error messages
* Learned the value of dir()
* Illustrate the code I had to translate that had the weird form of the function call that I figured out how to do in python

```python
def test_func(input_num):
    """this code is a test to see how snippets will be rendered"""
    
    constant_num = 5
    summation = constant_num + input_num
    if summation > 100:
        print 'whoa, big number, I quit!'
        exit()
    else:
        return summation
```