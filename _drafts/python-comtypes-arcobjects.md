---
layout: post
title: Building an ArcGIS Network Dataset with Python
---

## Building an ArcGIS Network Dataset with Python

Beginning about three years ago I inherited a project that had been undertaken at the request of our planning department and which aims to identify the amount of real estate development that has occurred near MAX stops due to their line’s creation.  In years past the work had been done essentially by hand and as I've intermittently updated the project I've been working to fully automate it.  As of the last pass there was just one piece of the process left that I had been unable to script: creating what ESRI calls a Network Dataset.

A [Network Dataset](http://desktop.arcgis.com/en/arcmap/latest/extensions/network-analyst/what-is-a-network-dataset.htm) is a routable network that can be configured to have various impedances and is derived from a street centerline file (in this case OpenStreetMap data).  I use this network to generate [isochrones](https://en.wikipedia.org/wiki/Isochrone_map) that illustrate the areas that can be reached from the MAX stations via the street and trail network with a half mile walk or less.  However `arcpy`, the python package that I use for scripting with ESRI products, does not have the capability to create these.  In light of this I thought that I would be forced to either continue creating these with the UI in ArcMap or use a different language until I came across [this post](http://gis.stackexchange.com/questions/109779) on GIS Stack Exchange that uses ArcObjects and the `comtypes` python package to carry out this task.

[ArcObjects](https://en.wikipedia.org/wiki/ArcObjects) underlie all of the ArcGIS Desktop products (anything that can be done with a desktop product can be done with ArcObjects), and though written primarily in C++, are a library of COM components.  [COM](https://en.wikipedia.org/wiki/Component_Object_Model) (Component Object Model) is a language neutral way of implementing objects that can be used in environments different from the one they were created in.  Documentation for ArcObjects generally illustrates their use with C#, Java and VB.NET, but python is also a COM-compatible language and comtypes makes them fairly easy to work with. 

ArcObjects can be accessed in python by providing the file path at which a COM component is stored to comtypes `GetModule` function, which looks something like this: 

```python
from os.path import join

from arcpy import GetInstallInfo
from comtypes.client import GetModule

ARCOBJECTS_DIR = join(GetInstallInfo()['InstallDir'], 'com')
GetModule(join(ARCOBJECTS_DIR, 'esriDataSourcesGDB.olb'))

from comtypes.gen.esriDataSourcesGDB import FileGDBWorkspaceFactory
```

ESRI has published a tutorial on Network Dataset creation with ArcObjects using VB.NET and C# so carrying this out in python is just a matter of translating that guide.  This is what was done in the Stack Exchange post, and although the settings on my network were different in some cases it gave me a starting point.  Below is a snippet of the VB code from the tutorial:

```vbnet
Dim gdbPath As String = "C:/Program Files/ArcGIS/data/SanFrancisco.gdb"
Dim featDataset As String = "Transportation"

' Create an empty data element for a buildable network dataset.
Dim deND As IDENetworkDataset2 = New DENetworkDataset
deND.Buildable = True

' Open the feature dataset and CType to the IGeoDataset interface.
Dim factoryType As Type = Type.GetTypeFromProgID("esriDataSourcesGDB.FileGDBWorkspaceFactory")
Dim gdbWSF As IWorkspaceFactory = CType(Activator.CreateInstance(factoryType), IWorkspaceFactory)
Dim gdbFWS As IFeatureWorkspace = CType(gdbWSF.OpenFromFile(gdbPath, 0), IFeatureWorkspace)
Dim fdsGDS As IGeoDataset = CType(gdbFWS.OpenFeatureDataset(featDataset), IGeoDataset)
```

In order to more easily mimic VB in python [this post](http://gis.stackexchange.com/questions/129456/guidelines-for-using-arcobjects-from-python) recommends creating the following functions.  These are just simple wrappers on comtypes functions, but they emulate VB.NET functionality more closely and are less verbose which, given how often they need to be called, is meaningful:

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

ArcObjects are mostly comprised of classes (objects) and interfaces.  The `new_obj` function creates a new instance of class and applies the supplied interface and `ctype` changes the interface of an existing object.  In ArcObjects classes often have many interfaces and each one allows access to different properties and methods of a class.

With those in place and the needed ArcObjects imported the conversion of the VB snippet looks like this:

```python
gdb_path = 'C:/Program Files/ArcGIS/data/SanFrancisco.gdb'
feat_dataset = 'Transportation'

# create an empty data element for a buildable network dataset
de_nd = new_obj(DENetworkDataset, IDENetworkDataset2)
de_nd.NetworkType = esriNetworkDatasetType(1)

# open the feature class and ctype to the IGeoDataset interface
gdb_wsf = new_obj(FileGDBWorkspaceFactory, IWorkspaceFactory)
gdb_fws = ctype(gdb_wsf.OpenFromFile(gdb_path, 0), IFeatureWorkspace)
fds_gds = ctype(gdb_fws.OpenFeatureDataset(feat_dataset), IGeoDataset)
```

As you can see most of the VB/python conversion is straight forward, but occasionally there is some VB syntax that does something that isn't really possible in python.  An example from the tutorial:

```vbnet
evalNetAttr.Evaluator(edgeNetSource, esriNetworkEdgeDirection.esriNEDAgainstDigitized) _
    = CType(netFieldEval, INetworkEvaluator)
```

The problem here is that python doesn't allow an object to be assigned to a method call as is being done above.  This had me baffled for awhile, but using a combination of the python builtin `dir()` to discover the properties and methods of the objects involved and the [ArcObjects docs](http://resources.arcgis.com/en/help/arcobjects-net/componenthelp/index.html) I was able put together this solution:

```python
eval_net attr.Evaluator.setter(
    eval_net_attr, edge_net_src, esriNetworkEdgeDirection(2),
    ctype(net_field_eval, INetworkEvaluator))
```

The full VB.NET/C# walk-through can be found [here](http://resources.arcgis.com/en/help/arcobjects-net/conceptualhelp/index.html#/How_to_create_a_network_dataset/0001000000w7000000/) and my python code that creates a Network Dataset is [here](https://github.com/grant-humphries/dev-near-light-rail/blob/master/lightraildev/create_network_dataset.py).  See the notes below for other

 Here are a couple more resources that helped me get going with ArcObjects and python.

#### Notes for blog post

* Used VBA for network attributes, executes much faster than python
* Seemingly random string for arcobjects array object
    * for some reason esriSystem.Array objects can't be created normally via comtypes, I found a workaround on pg 7 of the linked pdf, below is the object's GUID which can be supplied in place of the object
    * http://www.pierssen.com/arcgis10/upload/python/arcmap_and_python.pdf
    * ARRAY_GUID = "{8F2B6061-AB00-11D2-87F4-0000F8751720}"
* The downside with work with these tools is the nonsensical error messages
* Illustrate the code I had to translate that had the weird form of the function call that I figured out how to do in python
    * Learned the value of dir()
