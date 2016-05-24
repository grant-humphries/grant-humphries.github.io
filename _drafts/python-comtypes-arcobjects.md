---
layout: post
title: Building an ArcGIS Network Dataset with Python
---

## Building an ArcGIS Network Dataset with Python

Beginning about three years ago I inherited a project that had been undertaken at the request of the planning department which aims to identify the amount of real estate development that has occurred near MAX stops due to their line’s creation.  In years past the work had been done essentially by hand and as I've intermittently updated the project I've been working to fully automate it.  As of the last pass there was just one piece of the process left that I had been unable to script: creating what ESRI calls a Network Dataset.

A [Network Dataset](http://desktop.arcgis.com/en/arcmap/latest/extensions/network-analyst/what-is-a-network-dataset.htm) is a routable network that can be configured to have various impedances and is derived from a street centerline file (in this case OpenStreetMap data).  I use this network to generate isochrones that illustrate the areas that can be reached from the MAX stations via the street and trail network with a half mile walk or less.  However the `arcpy` python package that I use for scripting with ESRI products does not have the capability to create these, and the wizard that ArcMap provides takes several minutes to get through.  I thought that I was out of luck, at least as far as using python, until I came across [this post](http://gis.stackexchange.com/questions/109779/build-network-dataset-with-python-comtypes) on GIS Stack Exchange that uses ArcObjects and the python package `comtypes` to carry out this task.

[ArcObjects](https://en.wikipedia.org/wiki/ArcObjects) are the foundation of ArcGIS Desktop products (i.e. anything that can be done with a desktop product can be done with ArcObjects), and though written primarily in C++, are a library of COM components .  [COM](https://en.wikipedia.org/wiki/Component_Object_Model) (Component Object Model) is a language neutral way of implementing objects that can be used in environments different from the one they were created in.  Documentation for ArcObjects generally illustrates their use with C#, Java and VB.NET, but python is also a COM-compatible language and the comtypes package makes them fairly easy to work with.

ESRI has documentation on Network Dataset creation with ArcObjects and VB.NET and C# so carrying this out with python was just a matter of translating those guides.  This is what was done in the Stack Exchange post, and although the settings on my network were different in some cases this gave me a starting point.  Below is a snippet of the 



```vbnet
' Create an empty data element for a buildable network dataset.
Dim deND As IDENetworkDataset2 = New DENetworkDataset
deND.Buildable = True

' Open the feature dataset and CType to the IGeoDataset interface.
Dim factoryType As Type = Type.GetTypeFromProgID("esriDataSourcesGDB.FileGDBWorkspaceFactory")
Dim gdbWSF As IWorkspaceFactory = CType(Activator.CreateInstance(factoryType), IWorkspaceFactory)
Dim gdbFWS As IFeatureWorkspace = CType(gdbWSF.OpenFromFile("C:\Program Files\ArcGIS\DeveloperKit10.0\Samples\data\SanFrancisco.gdb", 0), IFeatureWorkspace)
Dim fdsGDS As IGeoDataset = CType(gdbFWS.OpenFeatureDataset("Transportation"), IGeoDataset)
```


```python
# create an empty data element for a buildable network dataset
net = new_obj(DENetworkDataset, IDENetworkDataset2)
net.Buildable = True
net.NetworkType = esriNetworkDatasetType(1)

# open the feature class and ctype to the IGeoDataset interface
gdb_ws_factory = new_obj(FileGDBWorkspaceFactory, IWorkspaceFactory)
gdb_workspace = ctype(gdb_ws_factory.OpenFromFile(OSM_PED_GDB, 0),
                      IFeatureWorkspace)
gdb_feat_ds = ctype(gdb_workspace.OpenFeatureDataset(FDS_NAME),
                    IGeoDataset)
```



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