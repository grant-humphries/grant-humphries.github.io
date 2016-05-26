## Building an ArcGIS Network Dataset with Python

Beginning about three years ago I inherited a project that had been undertaken at the request of our planning department and which aims to identify the amount of real estate development that has occurred near MAX stops due to their line’s creation.  In years past the work had been done essentially by hand, and as I've intermittently updated the project I've been working to fully automate it.  As of the last pass there was just one piece of the process left that I had been unable to script: creating what ESRI calls a Network Dataset.

A [Network Dataset](http://desktop.arcgis.com/en/arcmap/latest/extensions/network-analyst/what-is-a-network-dataset.htm) is a routable network that can be configured with various impedances and is derived from a street centerline file (in this case OpenStreetMap data).  I use this network to generate [isochrones](https://en.wikipedia.org/wiki/Isochrone_map) that illustrate the areas that can be reached from MAX stations via the street and trail network with a half mile walk or less.  However `arcpy`, the Python package that I use for scripting with ESRI products, does not have the capability to create these networks.  In light of this it looked like my options were to either continue creating these manually with the UI in ArcMap or explore using a different language that was less compatible with the rest of my workflow.  But after doing a little more research I came across [this](http://gis.stackexchange.com/questions/109779) GIS Stack Exchange post that uses the ArcObjects library and Python's `comtypes` package to carry out this task.

[ArcObjects](https://en.wikipedia.org/wiki/ArcObjects) provide functionality to all of the ArcGIS Desktop products, and though written primarily in C++, is a library of COM components.  [COM](https://en.wikipedia.org/wiki/Component_Object_Model) (Component Object Model) is a language neutral way of implementing objects that can be used in environments different from the one they were created in.  Documentation for ArcObjects generally illustrates their use with C#, Java and VB.NET, but Python is also a COM-compatible language and comtypes makes them fairly easy to work with. 

ArcObjects can be accessed in Python by providing the file path of a COM component to comtypes `GetModule` function, which looks something like this: 

```python
from os.path import join
from arcpy import GetInstallInfo
from comtypes.client import GetModule

ARCOBJECTS_DIR = join(GetInstallInfo()['InstallDir'], 'com')
GetModule(join(ARCOBJECTS_DIR, 'esriDataSourcesGDB.olb'))

from comtypes.gen.esriDataSourcesGDB import FileGDBWorkspaceFactory
```

ESRI has published a tutorial on Network Dataset creation with ArcObjects using VB.NET and C# so carrying this out in Python is just a matter of translating that guide.  This is what was done in the Stack Exchange post, and although the settings on my network were different in some cases it provided a strong basis to get started with the conversion.  Below is a snippet of the VB code from the tutorial:

```vbnet
Dim gdbPath As String = "C:/Program Files/ArcGIS/data/SanFrancisco.gdb"
Dim featDataset As String = "Transportation"

' Create an empty data element for a buildable network dataset.
Dim deND As IDENetworkDataset2 = New DENetworkDataset
deND.Buildable = True

' Open the feature dataset and CType to the IGeoDataset interface.
Dim factoryType As Type = _
    Type.GetTypeFromProgID("esriDataSourcesGDB.FileGDBWorkspaceFactory")
Dim gdbWSF As IWorkspaceFactory = _
    CType(Activator.CreateInstance(factoryType), IWorkspaceFactory)
Dim gdbFWS As IFeatureWorkspace = _
    CType(gdbWSF.OpenFromFile(gdbPath, 0), IFeatureWorkspace)
Dim fdsGDS As IGeoDataset = _
    CType(gdbFWS.OpenFeatureDataset(featDataset), IGeoDataset)
```

In order to more easily mimic VB in Python [this post](http://gis.stackexchange.com/questions/129456) recommends creating the following functions.  These are just simple wrappers on comtypes functions, but they emulate VB.NET syntax more closely and are a little more terse which, given how often they need to be called, is meaningful:

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

ArcObjects are mostly comprised of classes (objects) and interfaces.  The `new_obj` function creates a new instance of class and applies the supplied interface and `ctype` changes the interface of an existing object.  In ArcObjects classes often have many compatible interfaces and each one allows access to different properties and methods of a class.  With those in place and the needed ArcObjects imported the conversion of the VB snippet looks like this:

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

As you can see most of the VB-Python conversion is straightforward, but occasionally there is some VB syntax that does something that isn't really possible in Python.  An example from the tutorial:

```vbnet
evalNetAttr.Evaluator( _
        edgeNetSource, _
        esriNetworkEdgeDirection.esriNEDAgainstDigitized) _
    = CType(netFieldEval, INetworkEvaluator)
```

The problem here is that Python doesn't allow an object to be assigned to a method call as is being done above.  It took me awhile to work through this, but using a combination of the Python builtin `dir()` to discover the properties and methods of the objects involved and the [ArcObjects docs](http://resources.arcgis.com/en/help/arcobjects-net/componenthelp/index.html) I was able put together this solution:

```python
eval_net attr.Evaluator.setter(
    eval_net_attr, edge_net_src, esriNetworkEdgeDirection(2),
    ctype(net_field_eval, INetworkEvaluator))
```

So these are some the basics of working with Python and ArcObjects for network creation as well as more broadly.  ESRI's VB.NET/C# Network Dataset walk-through can be found [here](http://resources.arcgis.com/en/help/arcobjects-net/conceptualhelp/index.html#/How_to_create_a_network_dataset/0001000000w7000000/) and the full code for my script that creates a similar network with Python is [here](https://github.com/grant-humphries/dev-near-light-rail/blob/master/lightraildev/create_network_dataset.py).  I'll close below with a few notes on issues that I ran into along the way that may be of use to others:

* The downside to working in this environment is that the error messages that the COM objects return are often cryptic.  For instance, I initially wanted to create the Network Dataset from a shapefile and translated [a version](http://edndoc.esri.com/arcobjects/9.2/NET/06443414-d0a7-455d-a199-dfd49aca7d98.htm) of the guide which does that, but ended up with an error that gave me no leads.  I eventually ended up converting the shapefile to file geodatabase format and things worked from there.  The moral here is that you will likely run into these kinds of road blocks so be prepared to get creative (and drop me a line if you figure out how to create a network from a shapefile!). 
* There's an ArcObjects class called `esriSystem.Array` which can be used as to hold series of other ArcObjects, but when I attempt to create an instance of this class Python throws an error.  I didn't find the cause of the error, but was able to find a workaround in [this presentation](http://www.pierssen.com/arcgis10/upload/python/arcmap_and_python.pdf).  That hack is illustrated below:

    ```python
    from comtypes.gen.esriSystem import IArray
    
    ARRAY_GUID = '{8F2B6061-AB00-11D2-87F4-0000F8751720}'
    ao_array = new_obj(ARRAY_GUID, IArray)
    ```

* Key to configuring a network is what are called Network Attributes.  These settings determine things such as street traversablilty and can provide information like travel time for a trip planned on their network.  The logic for these attributes must be passed to ArcObjects as either VBScript or Python code.  When attempting to use Python for this purpose I again was met with errors, but the VBScript script below, embedded within a python string, did work for me to set pedestrian permissions.  The upside here is that I later found in [this documentation](http://desktop.arcgis.com/en/arcmap/latest/extensions/network-analyst/types-of-evaluators-used-by-a-network.htm) that VBScript executes significant faster than Python in this setting.  This proved to be true for me when comparing build times to my previous method of creating the network with the ArcMap UI which utilized Python logic.

    ```python
    logic = (
        'Set foot_list = CreateObject("System.Collections.ArrayList")'   '\n'
        'foot_list.Add "designated"'                                     '\n'
        'foot_list.Add "permissive"'                                     '\n'
        'foot_list.Add "yes"'                                          '\n\n'

        'Set hwy_list = CreateObject("System.Collections.ArrayList")'    '\n'
        'hwy_list.Add "construction"'                                    '\n'
        'hwy_list.Add "motorway"'                                        '\n'
        'hwy_list.Add "trunk"'                                         '\n\n'

        'If foot_list.Contains([foot]) Then'                             '\n'
        '    restricted = False'                                         '\n'
        'ElseIf _'                                                       '\n'
        '        ([access] = "no") Or _'                                 '\n'
        '        ([foot] = "no") Or _'                                   '\n'
        '        ([indoor] = "yes") Or _'                                '\n'
        '        (hwy_list.Contains([highway])) Then'                    '\n'
        '    restricted = True'                                          '\n'
        'Else'                                                           '\n'
        '    restricted = False'                                         '\n'
        'End If'
    )
    ```
* Last, [here](http://gis.stackexchange.com/questions/109779) are the [links](http://gis.stackexchange.com/questions/80), all in one place, to the GIS Stack Exchange [posts](http://gis.stackexchange.com/questions/129456) that helped me get going with this project.  In the third link see the response that clarifies where the various classes and interfaces are located within the COM objects and how to import them into Python with comtypes.
