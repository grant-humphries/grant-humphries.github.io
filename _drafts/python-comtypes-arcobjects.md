---
layout: post
title: Building an ArcGIS Network Dataset with Python
---

## Building an ArcGIS Network Dataset with Python

Beginning about three years ago I inherited a project that had been undertaken at the request of the planning department which aims to identify the amount of real estate development that has occurred near MAX stops due to their line’s creation.  In years past the work had been done essentially by hand and as I've intermittently updated the project I've been working to fully automate it.  As of the last pass there was just one piece of the process left that I had been unable to script: creating what ESRI calls a Network Dataset.

A [Network Dataset](http://desktop.arcgis.com/en/arcmap/latest/extensions/network-analyst/what-is-a-network-dataset.htm) is a routable network that can be configured to have various impedances and is derived from a street centerline file (in this case OpenStreetMap data).  I use this network to generate [isochrones](https://en.wikipedia.org/wiki/Isochrone_map) that illustrate the areas that can be reached from the MAX stations via the street and trail network with a half mile walk or less.  However the `arcpy` python package that I use for scripting with ESRI products does not have the capability to create these.  I thought that I was out of luck, at least as far as using python, until I came across [this post](http://gis.stackexchange.com/questions/109779) on GIS Stack Exchange that uses ArcObjects and the python package `comtypes` to carry out this task.

[ArcObjects](https://en.wikipedia.org/wiki/ArcObjects) underlie all of the ArcGIS Desktop products (anything that can be done with a desktop product can be done with ArcObjects), and though written primarily in C++, are a library of COM components.  [COM](https://en.wikipedia.org/wiki/Component_Object_Model) (Component Object Model) is a language neutral way of implementing objects that can be used in environments different from the one they were created in.  Documentation for ArcObjects generally illustrates their use with C#, Java and VB.NET, but python is also a COM-compatible language and the comtypes package makes them fairly easy to work with.  Here are a couple more resources that helped me get going with ArcObjects and python.

To access individual ArcObjects in python you must find the directory in which the COM objects are stored and then use comtypes `GetModule` function to bring them into python, which looks something like this: 

```python
from os.path import join

from arcpy import GetInstallInfo
from comtypes.client import GetModule

ARCOBJECTS_DIR = join(GetInstallInfo()['InstallDir'], 'com')
GetModule(join(ARCOBJECTS_DIR, 'esriDataSourcesGDB.olb'))

from comtypes.gen.esriDataSourcesGDB import FileGDBWorkspaceFactory
```

ESRI has published a tutorial on Network Dataset creation with ArcObjects using VB.NET and C# so carrying this out with python was just a matter of translating those guides.  This is what was done in the Stack Exchange post, and although the settings on my network were different in some cases this gave me a starting point.  Below is a snippet of the of the VB code:

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

In order to more easily mimic VB in python I recommend creating the following functions (which were derived from here).  These are just very simple wrappers on comtypes functions, but they emulate the VB.NET functionality more closely and are also less verbose which given how often they need to be called is meaningful:

```python
from comtypes.client import CreateObject

def new_obj(class_, interface):
    """Creates a new comtypes POINTER object where 'class_' is the class
    to be instantiated, 'interface' is the interface to be assigned
    """

    pointer = CreateObject(class_, interface=interface)
    return pointer

def ctype(obj, interface):
    """Casts obj to interface and returns comtypes POINTER"""

    pointer = obj.QueryInterface(interface)
    return pointer
```

With those in place and the needed ArcObject imported the conversion looks like this:

```python
gdb_path = 'C:/Program Files/ArcGIS/DeveloperKit10.0/Samples/data/SanFrancisco.gdb'
feature_dataset = 'Transportation'

# create an empty data element for a buildable network dataset
de_nd = new_obj(DENetworkDataset, IDENetworkDataset2)
de_nd.Buildable = True
de_nd.NetworkType = esriNetworkDatasetType(1)

# open the feature class and ctype to the IGeoDataset interface
gdb_wsf = new_obj(FileGDBWorkspaceFactory, IWorkspaceFactory)
gdb_fws = ctype(gdb_wsf.OpenFromFile(gdb_path, 0), IFeatureWorkspace)
fds_gds = ctype(gdb_fws.OpenFeatureDataset(feature_dataset), IGeoDataset)
```

From this 


#### Notes for blog post

* writing this code gave insight into the way that vb.net and c# use objects and and interfaces to build things
* Used VBA for network attributes, executes much faster than python
* Seemingly random string for arcobjects array object
    * for some reason esriSystem.Array objects can't be created normally via comtypes, I found a workaround on pg 7 of the linked pdf, below is the object's GUID which can be supplied in place of the object
    * http://www.pierssen.com/arcgis10/upload/python/arcmap_and_python.pdf
    * ARRAY_GUID = "{8F2B6061-AB00-11D2-87F4-0000F8751720}"
    
* The downside with work with these tools is the nonsensical error messages
* Learned the value of dir()
* Illustrate the code I had to translate that had the weird form of the function call that I figured out how to do in python
