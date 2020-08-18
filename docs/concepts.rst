Concepts
########

This page will give an overview of some important concepts to understand when working with PySTAC. If you want to check code examples, see the :ref:`tutorials`.

Reading STACs
=============

PySTAC can read STAC data from JSON. Generally users read in the root catalog, and then use the python objects to crawl through the data. Once you read in the root of the STAC, you can
work with the STAC in memory.

.. code-block:: python

   from pystac import Catalog

   catalog = Catalog.from_file('/some/example/catalog.json')

   for root, catalogs, items in catalog.walk():
       # Do interesting things with the STAC data.

To see how to hook into PySTAC for reading from alternate URIs such as cloud object storage,
see :ref:`using stac_io`.

Writing STACs
=============

While working with STACs in-memory don't require setting file paths, in order to save a STAC,
you'll need to give each STAC object a ``self`` link that describes the location of where
it should be saved to. Luckily, PySTAC makes it easy to create a STAC catalog with a `canonical layout <https://github.com/radiantearth/stac-spec/blob/v1.0.0-beta.2/best-practices.md#catalog-layout>`_ and with the links that follow the `best practices <https://github.com/radiantearth/stac-spec/blob/v1.0.0-beta.2/best-practices.md#use-of-links>`_. You simply call ``normalize_hrefs`` with the root directory of where the STAC will be saved, and then call ``save`` with the type of catalog (described in the :ref:`catalog types` section) that matches your use case.

.. code-block:: python

   from pystac import (Catalog, CatalogType)

   catalog = Catalog.from_file('/some/example/catalog.json')
   catalog.normalize_hrefs('/some/copy/')
   catalog.save(catalog_type=CatalogType.SELF_CONTAINED)

   copycat = Catalog.from_file('/some/copy/catalog.json')


Normalizing HREFs
-----------------

The ``normalize_hrefs`` call sets HREFs for all the links in the STAC according to the
Catalog, Collection and Items, all based off of the root URI that is passed in:

.. code-block:: python

    catalog.normalize_hrefs('/some/location')
    catalog.save(catalog_type=CatalogType.SELF_CONTAINED)

If you want to set your HREFs in a non-canonical format, you can set each STAC object href
manually by using ``set_self_href``:

.. code-block:: python

   import os

   top_level_dir = '/some/location'
   for root, _, items in catalog.walk():

       # Set root's HREF based off the parent
       parent = root.get_parent()
       if parent is None:
           root_dir = top_level_dir
       else:
           d = os.path.dirname(parent.get_self_href())
           root_dir = os.path.join(d, root.id)
       root_href = os.path.join(root_dir, root.DEFAULT_FILE_NAME)
       root.set_self_href(root_href)

       # Set each item's HREF based on it's datetime
       for item in items:
           item_href = '{}/{}-{}/{}.json'.format(root_dir,
                                                 item.datetime.year,
                                                 item.datetime.month,
                                                 item.id)
           item.set_self_href(item_href)

    catalog.save(catalog_type=CatalogType.SELF_CONTAINED)

.. _catalog types:

Catalog Types
-------------

The STAC `best practices document <https://github.com/radiantearth/stac-spec/blob/v1.0.0-beta.2/best-practices.md>`_ lays out different catalog types, and how their links should be formatted. A brief description is below, but check out the document for the official take on these types:

Note that the catalog types do not dictate the asset HREF formats, only link formats. Asset HREFs in any catalog type can be relative or absolute; see the section on :ref:`rel vs abs asset` below.


Self-Contained Catalogs
~~~~~~~~~~~~~~~~~~~~~~~

A self-contained catalog (indicated by ``catalog_type=CatalogType.SELF_CONTAINED``) applies
to STACs that do not have a long term location, and can be moved around. These STACs are
useful for copying data to and from locations, without having to change any link metadata.

A self-contained catalog has two important properties:

- It contains only relative links
- It contains **no** self links.

For a catalog that is the most easy to copy around, it's recommended that item assets use relative links, and reside in the same directory as the item's STAC metadata file.

Relative Published Catalogs
~~~~~~~~~~~~~~~~~~~~~~~~~~~

A relative published catalog (indicated by ``catalog_type=CatalogType.RELATIVE_PUBLISHED``) is
one that is tied at it's root to a specific location, but otherwise contains relative links.
This is designed so that a self-contained catalog can be 'published' online by just adding
one field (the self link) to its root catalog.

A relative published catalog has the following properties:

- It contains **only one** self link: the root of the catalog contains a (necessarily absolute) link to it's published location.
- All other objects in the STAC contain relative links, and no self links.


Absolute Published Catalogs
~~~~~~~~~~~~~~~~~~~~~~~~~~~

An absolute published catalog (indicated by ``catalog_type=CatalogType.ABSOLUTE_PUBLISHED``) uses absolute links for everything. It is preferable where possible, since it allows for the easiest provenance tracking out of all the catalog types.

An absolute published catalog has the following properties:

- Each STAC object contains only absolute links.
- Each STAC object has a self link.

It is not recommended to have relative asset HREFs in an absolute published catalog.


Relative vs Absolute HREFs
--------------------------

HREFs inside a STAC for either links or assets can be relative or absolute.

Relative vs Absolute Link HREFs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Absolute links point to their file locations in a fully described way. Relative links
are relative to the linking object's file location. For example, if a catalog at
``/some/location/catalog.json`` has a link to an item that has an HREF set to ``item-id/item-id.json``, then that link should resolve to the absolute path ``/some/location/item-id/item-id.json``.

The implementation of :class:`~pystac.Link` in PySTAC allows for the link to be marked as
``link_type=LinkType.ABSOLUTE`` or ``link_type=LinkType.RELATIVE``. This means that,
even if the stored HREF of the link is absolute, if the link is marked as relative, serializing
the link will produce a relative link, based on the self link of the parent object.

You can make all the links of a catalog relative or absolute using the :func:`Catalog.make_all_links_relative <pystac.Catalog.make_all_links_relative>` and :func:`Catalog.make_all_links_absolute <pystac.Catalog.make_all_links_absolute>` methods.

.. _rel vs abs asset:

Relative vs Absolute Asset HREFs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Asset HREFs can also be relative or absolute. If an asset HREF is relative, then it is relative to the Item's metadata file. For example, if the item at ``/some/location/item-id/item-id.json`` had an asset with an HREF of ``./image.tif``, then the fully resolved path for that image would be ``/some/location/item-id/image.tif``

You can make all the asset HREFs of a catalog relative or absolute using the :func:`Catalog.make_all_asset_hrefs_relative <pystac.Catalog.make_all_asset_hrefs_relative>` and :func:`Catalog.make_all_asset_hrefs_absolute <pystac.Catalog.make_all_asset_hrefs_absolute>` methods. Note that these will not move any files around, and if the file location does not share a common parent with the asset's item's self HREF, then the asset HREF will remain absolute as no relative path is possible.

Including a ``self`` link
-------------------------

Every stac object has a :func:`~pystac.STACObject.save_object` method, that takes as an argument whether or not to include the object's self link. As noted in the section on :ref:`catalog types`, a self link is necessarily absolute; if an object only contains relative links, then it cannot contain the self link. PySTAC uses self links as a way of tracking the object's file location, either what it was read from or it's pending save location, so each object can have a self link even if you don't ever want that self link written (e.g. if you are working with self-contained catalogs).

.. _using stac_io:

Using STAC_IO
=============

The :class:`~pystac.STAC_IO` class is the way PySTAC reads and writes text from file locations. Since PySTAC aims to be dependency-free, there is no default mechanisms to read and write from anything but the local file system. However, users of PySTAC may want to read and write from other file systems, such as HTTP or cloud object storage. STAC_IO allows users to hook into PySTAC and define their own reading and writing primitives to allow for those use cases.

To enable reading from other types of file systems, it is recommended that in the `__init__.py` of the client module, or at the beginning of the script using PySTAC, you overwrite the :func:`STAC_IO.read_text_method <pystac.STAC_IO.read_text_method>` and :func:`STAC_IO.write_text_method <pystac.STAC_IO.write_text_method>` members of STAC_IO with functions that read and write however you need. For example, this code will allow for reading from AWS's S3 cloud object storage using `boto3 <https://boto3.amazonaws.com/v1/documentation/api/latest/index.html>`_:

.. code-block:: python

   from urllib.parse import urlparse
   import boto3
   from pystac import STAC_IO

   def my_read_method(uri):
       parsed = urlparse(uri)
       if parsed.scheme == 's3':
           bucket = parsed.netloc
           key = parsed.path[1:]
           s3 = boto3.resource('s3')
           obj = s3.Object(bucket, key)
           return obj.get()['Body'].read().decode('utf-8')
       else:
           return STAC_IO.default_read_text_method(uri)

   def my_write_method(uri, txt):
       parsed = urlparse(uri)
       if parsed.scheme == 's3':
           bucket = parsed.netloc
           key = parsed.path[1:]
           s3 = boto3.resource("s3")
           s3.Object(bucket, key).put(Body=txt)
       else:
           STAC_IO.default_write_text_method(uri, txt)

   STAC_IO.read_text_method = my_read_method
   STAC_IO.write_text_method = my_write_method

If you are only going to read from another source, e.g. HTTP, you could only replace the read method. For example, using the `requests library <https://requests.kennethreitz.org/en/master>`_:

.. code-block:: python

   from urllib.parse import urlparse
   import requests
   from pystac import STAC_IO

   def my_read_method(uri):
       parsed = urlparse(uri)
       if parsed.scheme.startswith('http'):
           return requests.get(uri).text
       else:
           return STAC_IO.default_read_text_method(uri)

   STAC_IO.read_text_method = my_read_method

Validation
==========

PySTAC includes validation functionality that allows users to validate PySTAC objects as well JSON-encoded STAC objects from STAC versions `0.8.0` and later.

Enabling validation
-------------------

To enable the validation feature you'll need to have installed PySTAC with the optional dependency via:

.. code-block:: bash

   > pip install pystac[validation]

This installs the ``jsonschema`` package which is used with the default validator. If you
define your own validation class as described below, you are not required to have this
extra dependency.

Validating PySTAC objects
-------------------------

You can validate any :class:`~pystac.Catalog`, :class:`~pystac.Collection` or :class:`~pystac.Item` by calling the :meth:`~pystac.STACObject.validate` method:

.. code-block:: python

   item.validate()

This will validate against the latest set of JSON schemas hosted at https://schemas.stacspec.org, including any extensions that the object extends. If there are validation errors, a :class:`~pystac.validation.STACValidationError` will be raised.

You can also call :meth:`~pystac.Catalog.validate_all` on a Catalog or Collection to recursively walk through a catalog and validate all objects within it.

.. code-block:: python

   catalog.validate_all()

Validating STAC JSON
--------------------

You can validate STAC JSON represented as a ``dict`` using the :meth:`pystac.validation.validate_dict` method:

.. code-block:: python

   import json
   from pystac.validation import validate_dict

   with open('/path/to/item.json') as f:
       js = json.load(f)
   validate_dict(js)

Using your own validator
------------------------

By default PySTAC uses the :class:`~pystac.validation.JsonSchemaSTACValidator` implementation for validation. Users can define their own implementations of :class:`~pystac.validation.STACValidator` and register it with pystac using :meth:`pystac.validation.set_validator`.

The :class:`~pystac.validation.JsonSchemaSTACValidator` takes a :class:`~pystac.validation.SchemaUriMap`, which by default uses the :class:`~pystac.validation.schema_uri_map.DefaultSchemaUriMap`. If desirable, users cn create their own implementation of :class:`~pystac.validation.SchemaUriMap` and register a new instance of :class:`~pystac.validation.JsonSchemaSTACValidator` using that schema map with :meth:`pystac.validation.set_validator`.


Extensions
==========

Accessing Extension functionality
---------------------------------

All STAC objects are accessed through ``Catalog``, ``Collection`` and ``Item``, and all extension functionality
is accessed through the ``ext`` property on those objects. For instance, to access the band information
from the ``eo`` extension for an item that implements the extension, you use:

.. code-block:: python

   # All of the below are equivalent:
   item.ext['eo'].bands
   item.ext[pystac.Extensions.EO].bands
   item.ext.eo.bands

Notice the ``eo`` property on ``ext`` - this utilizes the `__getattr__ <https://docs.python.org/3/reference/datamodel.html#object.__getattr__>`_ method to delegate the property name to the ``__getitem__`` method, so we can access any registered extension as if it were a property on ``ext``.

Extensions wrap the objects they extend. Extensions hold
no values of their own, but instead use Python `properties <https://docs.python.org/3/library/functions.html#property>`_
to directly modify the values of the objects they wrap.

Any object that is returned by extension methods therefore also wrap components of the STAC objects.
For instance, the ``LabelClasses`` holds a reference to the original ``Item``'s ``label:classes`` property, so that
modifying the ``LabelClasses``
properties through the setters will modify the item properties directly. For example:

.. code-block:: python

    from pystac.extensions import label

    label_classes = item.ext.label.label_classes
    label_classes[0].classes.append("other_class")
    assert "other_class" in item.properties['label:classes'][0]['classes']

Because these objects wrap the object's dictionary, the __init__ methods need to take the
``dict`` they wrap. Therefore to create a new object, use the class's `.create` method, for example:

.. code-block:: python

   item.ext.label.label_classes = [label.LabelClasses.create(['class1', 'class2'], name='label')]

An `apply` method is available in extension wrappers and any objects that they return. This allows
you to pass in property values pertaining to the extension. These will require arguments for properties
required as part of the extension specification and have `None` default values for optional parameters:

.. code-block:: python

   eo_ext = item.ext.eo
   eo_ext.apply(0.5, bands, cloud_cover=None) # Do not have to specify cloud_cover


If you attempt to retrieve an extension wrapper for an extension that the object doesn't implement, PySTAC will
throw a `pystac.extensions.ExtensionError`.

Enabling an extension
---------------------

You'll need to enable an extension on an object before using it. For example, if you are creating an Item and want to
apply the label extension, you can do so in two ways.

You can add the extension in the list of extensions when you create the Item:

.. code-block:: python

   item = Item(id='Labels',
               geometry=item.geometry,
               bbox=item.bbox,
               datetime=datetime.utcnow(),
               properties={},
               stac_extensions=[pystac.Extensions.LABEL])

or you can call ``ext.enable`` on an Item (which will work for any item, whether you created it or are modifying it):

.. code-block:: python

   item = Item(id='Labels',
               geometry=item.geometry,
               bbox=item.bbox,
               datetime=datetime.utcnow(),
               properties={})

   item.ext.enable(pystac.Extensions.LABEL)

Item Asset properties
=====================

Properties that apply to Items can be found in two places: the Item's properties or in any of an Item's Assets. If the property is on an Asset, it applies only that specific asset. For example, gsd defined for an Item represents the best Ground Sample Distance (resolution) for the data within the Item. However, some assets may be lower resolution and thus have a higher gsd. In that case, the `gsd` can be found on the Asset.

See the STAC documentation on `Additional Fields for Assets <https://github.com/radiantearth/stac-spec/blob/v1.0.0-beta.2/item-spec/item-spec.md#additional-fields-for-assets>`_ and the relevant `Best Practices <https://github.com/radiantearth/stac-spec/blob/v1.0.0-beta.2/best-practices.md#common-use-cases-of-additional-fields-for-assets>`_ for more information.

The implementation of this feature in PySTAC uses the method described here and is consistent across Item and ItemExtensions. The bare property names represent values for the Item only, but for each property where it is possible to set on both the Item or the Asset there is a ``get_`` and ``set_`` methods that optionally take an Asset. For the ``get_`` methods, if the property is found on the Asset, the Asset's value is used; otherwise the Item's value will be used. For the ``set_`` method, if an Asset is passed in the value will be applied to the Asset and not the Item.

For example, if we have an Item with a ``gsd`` of 10 with three bands, and only asset "band3" having a ``gsd`` of 20, the ``get_gsd`` method will behave in the following way:

  .. code-block:: python

     assert item.common_metadata.gsd == 10
     assert item.common_metadata.get_gsd() == 10
     assert item.common_metadata.get_gsd(item.asset['band1']) == 10
     assert item.common_metadata.get_gsd(item.asset['band3']) == 20

Similarly, if we set the asset at 'band2' to have a ``gsd`` of 30, it will only affect that asset:

  .. code-block:: python

     item.common_metadata.set_gsd(30, item.assets['band2']
     assert item.common_metadata.gsd == 10
     assert item.common_metadata.get_gsd(item.asset['band2']) == 30

Manipulating STACs
==================

PySTAC is designed to allow for STACs to be manipulated in-memory. This includes :ref:`copy stacs`, walking over all objects in a STAC and mutating their properties, or using collection-style `map` methods for mapping over items.


Walking over a STAC
-------------------

You can walk through all sub-catalogs and items of a catalog with a method inspired
by the Python Standard Library `os.walk() <https://docs.python.org/3/library/os.html#os.walk>`_ method: :func:`Catalog.walk() <pystac.Catalog.walk>`:

.. code-block:: python

   for root, subcats, items in catalog.walk():
       # Root represents a catalog currently being walked in the tree
       root.title = '{} has been walked!'.format(root.id)

       # subcats represents any catalogs or collections owned by root
       for cat in subcatalogs:
           cat.title = 'About to be walked!'

       # items represent all items that are contained by root
       for item in items:
           item.title = '{} - owned by {}'.format(item.id, root.id)

Mapping over Items
------------------

The :func:`Catalog.map_items <pystac.Catalog.map_items>` method is useful for manipulating items in a STAC. This will create a full copy of the STAC, so will leave the original catalog unmodified. In the method that manipulates and returns the modified item, you can return multiple items, in case you are generating new objects (e.g. creating a :class:`~pystac.LabelItem` for image items in a stac), or splitting items into smaller chunks (e.g. tiling out large image items).

.. code-block:: python

   def modify_item_title(item):
       item.title = 'Some new title'
       return item

   def create_label_item(item):
       # Assumes the GeoJSON labels are in the
       # same location as the image
       img_href = item.assets['ortho'].href
       label_href = '{}.geojson'.format(os.path.splitext(img_href)[0])
       label_item = LabelItem(id='Labels',
                         geometry=item.geometry,
                         bbox=item.bbox,
                         datetime=datetime.utcnow(),
                         properties={},
                         label_description='labels',
                         label_type='vector',
                         label_properties='label',
                         label_classes=[
                         LabelClasses(classes=['one', 'two'],
                                      name='label')
                         ],
                         label_tasks=['classification'])
       label_item.add_source(item, assets=['ortho'])
       label_item.add_geojson_labels(label_href)

       return [item, label_item]


   c = catalog.map_items(modify_item_title)
   c = c.map_items(create_label_item)
   new_catalog = c

.. _copy stacs:

Copying STACs in-memory
-----------------------

The in-memory copying of STACs to create new ones is crucial to correct manipulations and mutations of STAC data. The :func:`STACObject.full_copy <pystac.STACObject.full_copy>` mechanism handles this in a way that ties the elements of the copies STAC together correctly. This includes situations where there might be cycles in the graph of connected objects of the STAC (which otherwise would be `a tree <https://en.wikipedia.org/wiki/Tree_(graph_theory)>`_). For example, if a :class:`~pystac.LabelItem` lists a :attr:`~pystac.LabelItem.source` that is an item also contained in the root catalog; the full copy of the STAC will ensure that the :class:`~pystac.Item` instance representing the source imagery item is the same instance that is linked to by the :class:`~pystac.LabelItem`.

Resolving STAC objects
======================

PySTAC tries to only "resolve" STAC Objects - that is, load the metadata contained by STAC files pointed to by links into Python objects in-memory - when necessary. It also ensures that two links that point to the same object resolve to the same in-memory object.

Lazy resolution of STAC objects
-------------------------------

Links are read only when they need to be. For instance, when you load a catalog using :func:`Catalog.from_file <pystac.Catalog.from_file>`, the catalog and all of its links are read into a :class:`~pystac.Catalog` instance. If you iterate through :attr:`Catalog.links <pystac.Catalog.links>`, you'll see the :attr:`~pystac.Link.target` of the :class:`~pystac.Link` will refer to a string - that is the HREF of the link. However, if you call :func:`Catalog.get_items <pystac.Catalog.get_items>`, for instance, you'll get back the actual :class:`~pystac.Item` instances that are referred to by each item link in the Catalog. That's because at the time you call ``get_items``, PySTAC is "resolving" the links for any link that represents an item in the catalog.

The resolution mechanism is accomplished through :func:`Link.resolve_stac_object <pystac.Link.resolve_stac_object>`. Though this method is used extensively internally to PySTAC, ideally this is completely transparent to users of PySTAC, and you won't have to worry about how and when links get resolved. However, one important aspect to understand is how object resolution caching happens.

Resolution Caching
------------------

The root :class:`~pystac.Catalog` instance of a STAC (the Catalog which is linked to by every associated object's ``root`` link) contains a cache of resolved objects. This cache points to in-memory instances of :class:`~pystac.STACObject` s that have already been resolved through PySTAC crawling links associated with that root catalog. The cache works off of the stac object's ID, which is why **it is necessary for every STAC object in the catalog to have a unique identifier, which is unique across the entire STAC**.

When a link is being resolved from a STACObject that has it's root set, that root is passed into the :func:`Link.resolve_stac_object <pystac.Link.resolve_stac_object>` call. That root's :class:`~pystac.resolved_object_cache.ResolvedObjectCache` will be used to ensure that if the link is pointing to an object that has already been resolved, then that link will point to the same, single instance in the cache. This ensures working with STAC objects in memory doesn't create a situation where multiple copies of the same STAC objects are created from different links, manipulated, and written over each other.

Working with STAC JSON
======================

The ``pystac.serialization`` package has some functionality around working directly with STAC
JSON objects, without utilizing PySTAC object types. This is used internally by PySTAC, but might also be useful to users working directly with JSON (e.g. on validation).


Identifing STAC objects from JSON
---------------------------------

Users can identify STAC information, including the object type, version and extensions,
from JSON. The main method for this is :func:`~pystac.serialization.identify_stac_object`,
which returns an object that contains the object type, the range of versions this object is
valid for (according to PySTAC's best guess), the common extensions implemented by this object,
and any custom extensions (represented by URIs to JSON Schemas).

.. code-block:: python

   from pystac.serialization import identify_stac_object

   json_dict = ...

   info = identify_stac_object(json_dict)

   # The object type
   info.object_type

   # The version range
   info.version_range

   # The common extensions
   info.common_extensions

   # The custom Extensions
   info.custom_extensions

Merging common properties
-------------------------

For pre-1.0.0 STAC, The :func:`~pystac.serialization.merge_common_properties` will take a JSON dict that represents an item, and if it is associated with a collection, merge in the collection's properties. You can pass in a dict that contains previously read collections that caches collections by the HREF of the collection link and/or the collection ID, which can help avoid multiple reads of
collection links.

Note that this feature was dropped in STAC 1.0.0-beta.1
