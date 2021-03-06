.. _zend.feed.writer:

Zend_Feed_Writer
================

.. _zend.feed.writer.introduction:

Introduction
------------

``Zend_Feed_Writer`` is the sibling component to ``Zend_Feed_Reader`` responsible for generating feeds for output.
It supports the Atom 1.0 specification (*RFC* 4287) and *RSS* 2.0 as specified by the *RSS* Advisory Board (*RSS*
2.0.11). It does not deviate from these standards. It does, however, offer a simple Extension system which allows
for any extension and module for either of these two specifications to be implemented if they are not provided out
of the box.

In many ways, ``Zend_Feed_Writer`` is the inverse of ``Zend_Feed_Reader``. Where ``Zend_Feed_Reader`` focuses on
providing an easy to use architecture fronted by getter methods, ``Zend_Feed_Writer`` is fronted by similarly named
setters or mutators. This ensures the *API* won't pose a learning curve to anyone familiar with
``Zend_Feed_Reader``.

As a result of this design, the rest may even be obvious. Behind the scenes, data set on any ``Zend_Feed_Writer``
Data Container object is translated at render time onto a DOMDocument object using the necessary feed elements. For
each supported feed type there is both an Atom 1.0 and *RSS* 2.0 renderer. Using a DOMDocument class rather than a
templating solution has numerous advantages, the most obvious being the ability to export the DOMDocument for
additional processing and relying on *PHP* *DOM* for correct and valid rendering.

As with ``Zend_Feed_Reader``, ``Zend_Feed_Writer`` is a standalone replacement for ``Zend_Feed``'s Builder
architecture and is not compatible with those classes.

.. _zend.feed.writer.architecture:

Architecture
------------

The architecture of ``Zend_Feed_Writer`` is very simple. It has two core sets of classes: data containers and
renderers.

The containers include the ``Zend_Feed_Writer_Feed`` and ``Zend_Feed_Writer_Entry`` classes. The Entry classes can
be attached to any Feed class. The sole purpose of these containers is to collect data about the feed to generate
using a simple interface of setter methods. These methods perform some data validity testing. For example, it will
validate any passed *URI*\ s, dates, etc. These checks are not tied to any of the feed standards definitions. The
container objects also contain methods to allow for fast rendering and export of the final feed, and these can be
reused at will.

In addition to the main data container classes, there are two additional Atom 2.0 specific classes.
``Zend_Feed_Writer_Source`` and ``Zend_Feed_Writer_Deleted``. The former implements Atom 2.0 source elements which
carry source feed metadata for a specific entry within an aggregate feed (i.e. the current feed is not the entry's
original source). The latter implements the Atom Tombstones *RFC* allowing feeds to carry references to entries
which have been deleted.

While there are two main data container types, there are four renderers - two matching container renderers per
supported feed type. Each renderer accepts a container, and based on its content attempts to generate valid feed
markup. If the renderer is unable to generate valid feed markup, perhaps due to the container missing an obligatory
data point, it will report this by throwing an ``Exception``. While it is possible to ignore ``Exception``\ s, this
removes the default safeguard of ensuring you have sufficient data set to render a wholly valid feed.

To explain this more clearly, you may construct a set of data containers for a feed where there is a Feed
container, into which has been added some Entry containers and a Deleted container. This forms a data hierarchy
resembling a normal feed. When rendering is performed, this hierarchy has its pieces passed to relevant renderers
and the partial feeds (all DOMDocuments) are then pieced together to create a complete feed. In the case of Source
or Deleted (Tomestone) containers, these are rendered only for Atom 2.0 and ignored for *RSS*.

Due to the system being divided between data containers and renderers, it can make Extensions somewhat interesting.
A typical Extension offering namespaced feed and entry level elements, must itself reflect the exact same
architecture, i.e. offer feed and entry level data containers, and matching renderers. There is, fortunately, no
complex integration work required since all Extension classes are simply registered and automatically used by the
core classes. We'll meet Extensions in more detail at the end of this section.

.. _zend.feed.writer.getting.started:

Getting Started
---------------

Using ``Zend_Feed_Writer`` is as simple as setting data and triggering the renderer. Here is an example to generate
a minimal Atom 1.0 feed. As this demonstrates, each feed or entry uses a separate data container.

.. code-block:: php
   :linenos:

   /**
    * Create the parent feed
    */
   $feed = new Zend_Feed_Writer_Feed;
   $feed->setTitle('Paddy\'s Blog');
   $feed->setLink('http://www.example.com');
   $feed->setFeedLink('http://www.example.com/atom', 'atom');
   $feed->addAuthor(array(
       'name'  => 'Paddy',
       'email' => 'paddy@example.com',
       'uri'   => 'http://www.example.com',
   ));
   $feed->setDateModified(time());
   $feed->addHub('http://pubsubhubbub.appspot.com/');

   /**
    * Add one or more entries. Note that entries must
    * be manually added once created.
    */
   $entry = $feed->createEntry();
   $entry->setTitle('All Your Base Are Belong To Us');
   $entry->setLink('http://www.example.com/all-your-base-are-belong-to-us');
   $entry->addAuthor(array(
       'name'  => 'Paddy',
       'email' => 'paddy@example.com',
       'uri'   => 'http://www.example.com',
   ));
   $entry->setDateModified(time());
   $entry->setDateCreated(time());
   $entry->setDescription('Exposing the difficultly of porting games to English.');
   $entry->setContent(
       'I am not writing the article. The example is long enough as is ;).'
   );
   $feed->addEntry($entry);

   /**
    * Render the resulting feed to Atom 1.0 and assign to $out.
    * You can substitute "atom" with "rss" to generate an RSS 2.0 feed.
    */
   $out = $feed->export('atom');

The output rendered should be as follows:

.. code-block:: xml
   :linenos:

   <?xml version="1.0" encoding="utf-8"?>
   <feed xmlns="http://www.w3.org/2005/Atom">
       <title type="text">Paddy's Blog</title>
       <subtitle type="text">Writing about PC Games since 176 BC.</subtitle>
       <updated>2009-12-14T20:28:18+00:00</updated>
       <generator uri="http://framework.zend.com" version="1.10.0alpha">
           Zend_Feed_Writer
       </generator>
       <link rel="alternate" type="text/html" href="http://www.example.com"/>
       <link rel="self" type="application/atom+xml"
           href="http://www.example.com/atom"/>
       <id>http://www.example.com</id>
       <author>
           <name>Paddy</name>
           <email>paddy@example.com</email>
           <uri>http://www.example.com</uri>
       </author>
       <link rel="hub" href="http://pubsubhubbub.appspot.com/"/>
       <entry>
           <title type="html"><![CDATA[All Your Base Are Belong To
               Us]]></title>
           <summary type="html">
               <![CDATA[Exposing the difficultly of porting games to
                   English.]]>
           </summary>
           <published>2009-12-14T20:28:18+00:00</published>
           <updated>2009-12-14T20:28:18+00:00</updated>
           <link rel="alternate" type="text/html"
                href="http://www.example.com/all-your-base-are-belong-to-us"/>
           <id>http://www.example.com/all-your-base-are-belong-to-us</id>
           <author>
               <name>Paddy</name>
               <email>paddy@example.com</email>
               <uri>http://www.example.com</uri>
           </author>
           <content type="html">
               <![CDATA[I am not writing the article.
                        The example is long enough as is ;).]]>
           </content>
       </entry>
   </feed>

This is a perfectly valid Atom 1.0 example. It should be noted that omitting an obligatory point of data, such as a
title, will trigger an ``Exception`` when rendering as Atom 1.0. This will differ for *RSS* 2.0 since a title may
be omitted so long as a description is present. This gives rise to Exceptions that differ between the two standards
depending on the renderer in use. By design, ``Zend_Feed_Writer`` will not render an invalid feed for either
standard unless the end-user deliberately elects to ignore all Exceptions. This built in safeguard was added to
ensure users without in-depth knowledge of the relevant specifications have a bit less to worry about.

.. _zend.feed.writer.setting.feed.data.points:

Setting Feed Data Points
------------------------

Before you can render a feed, you must first setup the data necessary for the feed being rendered. This utilises a
simple setter style *API* which doubles as an initial method for validating the data being set. By design, the
*API* closely matches that for ``Zend_Feed_Reader`` to avoid undue confusion and uncertainty.

.. note::

   Users have commented that the lack of a simple array based notation for input data gives rise to lengthy tracts
   of code. This will be addressed in a future release.

``Zend_Feed_Writer`` offers this *API* via its data container classes ``Zend_Feed_Writer_Feed`` and
``Zend_Feed_Writer_Entry`` (not to mention the Atom 2.0 specific and Extension classes). These classes merely store
all feed data in a type-agnostic manner, meaning you may reuse any data container with any renderer without
requiring additional work. Both classes are also amenable to Extensions, meaning that an Extension may define its
own container classes which are registered to the base container classes as extensions, and are checked when any
method call triggers the base container's ``__call()`` method.

Here's a summary of the Core *API* for Feeds. You should note it comprises not only the basic *RSS* and Atom
standards, but also accounts for a number of included Extensions bundled with ``Zend_Feed_Writer``. The naming of
these Extension sourced methods remain fairly generic - all Extension methods operate at the same level as the Core
*API* though we do allow you to retrieve any specific Extension object separately if required.

The Feed Level *API* for data is contained in ``Zend_Feed_Writer_Feed``. In addition to the *API* detailed below,
the class also implements the ``Countable`` and ``Iterator`` interfaces.

.. table:: Feed Level API Methods

   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setId()           |Set a unique ID associated with this feed. For Atom 1.0 this is an atom:id element, whereas for RSS 2.0 it is added as a guid element. These are optional so long as a link is added, i.e. the link is set as the ID.                                                                                                                                                                                                                                                                                                 |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setTitle()        |Set the title of the feed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setDescription()  |Set the text description of the feed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setLink()         |Set a URI to the HTML website containing the same or similar information as this feed (i.e. if the feed is from a blog, it should provide the blog's URI where the HTML version of the entries can be read).                                                                                                                                                                                                                                                                                                          |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setFeedLinks()    |Add a link to an XML feed, whether the feed being generated or an alternate URI pointing to the same feed but in a different format. At a minimum, it is recommended to include a link to the feed being generated so it has an identifiable final URI allowing a client to track its location changes without necessitating constant redirects. The parameter is an array of arrays, where each sub-array contains the keys "type" and "uri". The type should be one of "atom", "rss", or "rdf".                     |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |addAuthors()      |Sets the data for authors. The parameter is an array of arrays where each sub-array may contain the keys "name", "email" and "uri". The "uri" value is only applicable for Atom feeds since RSS contains no facility to show it. For RSS 2.0, rendering will create two elements - an author element containing the email reference with the name in brackets, and a Dublin Core creator element only containing the name.                                                                                            |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |addAuthor()       |Sets the data for a single author following the same array format as described above for a single sub-array.                                                                                                                                                                                                                                                                                                                                                                                                          |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setDateCreated()  |Sets the date on which this feed was created. Generally only applicable to Atom where it represents the date the resource described by an Atom 1.0 document was created. The expected parameter may be a UNIX timestamp or a DateTime object.                                                                                                                                                                                                                                                                         |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setDateModified() |Sets the date on which this feed was last modified. The expected parameter may be a UNIX timestamp or a DateTime object.                                                                                                                                                                                                                                                                                                                                                                                              |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setLastBuildDate()|Sets the date on which this feed was last build. The expected parameter may be a UNIX timestamp or a DateTime object. This will only be rendered for RSS 2.0 feeds and is automatically rendered as the current date by default when not explicity set.                                                                                                                                                                                                                                                               |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setLanguage()     |Sets the language of the feed. This will be omitted unless set.                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setGenerator()    |Allows the setting of a generator. The parameter should be an array containing the keys "name", "version" and "uri". If omitted a default generator will be added referencing Zend_Feed_Writer, the current Zend Framework version and the Framework's URI.                                                                                                                                                                                                                                                           |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setCopyright()    |Sets a copyright notice associated with the feed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |addHubs()         |Accepts an array of Pubsubhubbub Hub Endpoints to be rendered in the feed as Atom links so that PuSH Subscribers may subscribe to your feed. Note that you must implement a Pubsubhubbub Publisher in order for real-time updates to be enabled. A Publisher may be implemented using Zend_Feed_Pubsubhubbub_Publisher. The method addHub() allows adding a single hub at a time.                                                                                                                                     |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |addCategories()   |Accepts an array of categories for rendering, where each element is itself an array whose possible keys include "term", "label" and "scheme". The "term" is a typically a category name suitable for inclusion in a URI. The "label" may be a human readable category name supporting special characters (it is HTML encoded during rendering) and is a required key. The "scheme" (called the domain in RSS) is optional but must be a valid URI. The method addCategory() allows adding a single category at a time.|
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setImage()        |Accepts an array of image metadata for an RSS image or Atom logo. Atom 1.0 only requires a URI. RSS 2.0 requires a URI, HTML link, and an image title. RSS 2.0 optionally may send a width, height and image description. The array parameter may contain these using the keys: uri, link, title, description, height and width. The RSS 2.0 HTML link should point to the feed source's HTML page.                                                                                                                   |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |createEntry()     |Returns a new instance of Zend_Feed_Writer_Entry. This is the Entry level data container. New entries are not automatically assigned to the current feed, so you must explicitly call addEntry() to add the entry for rendering.                                                                                                                                                                                                                                                                                      |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |addEntry()        |Adds an instance of Zend_Feed_Writer_Entry to the current feed container for rendering.                                                                                                                                                                                                                                                                                                                                                                                                                               |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |createTombstone() |Returns a new instance of Zend_Feed_Writer_Deleted. This is the Atom 2.0 Tombstone level data container. New entries are not automatically assigned to the current feed, so you must explicitly call addTombstone() to add the deleted entry for rendering.                                                                                                                                                                                                                                                           |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |addTombstone()    |Adds an instance of Zend_Feed_Writer_Deleted to the current feed container for rendering.                                                                                                                                                                                                                                                                                                                                                                                                                             |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |removeEntry()     |Accepts a parameter indicating an array index of the entry to remove from the feed.                                                                                                                                                                                                                                                                                                                                                                                                                                   |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |export()          |Exports the entire data hierarchy to an XML feed. The method has two parameters. The first is the feed type, one of "atom" or "rss". The second is an optional boolean to set whether Exceptions are thrown. The default is TRUE.                                                                                                                                                                                                                                                                                     |
   +------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

.. note::

   In addition to these setters, there are also matching getters to retrieve data from the Entry data container.
   For example, ``setImage()`` is matched with a ``getImage()`` method.

.. _zend.feed.writer.setting.entry.data.points:

Setting Entry Data Points
-------------------------

Here's a summary of the Core *API* for Entries and Items. You should note it comprises not only the basic *RSS* and
Atom standards, but also accounts for a number of included Extensions bundled with ``Zend_Feed_Writer``. The naming
of these Extension sourced methods remain fairly generic - all Extension methods operate at the same level as the
Core *API* though we do allow you to retrieve any specific Extension object separately if required.

The Entry Level *API* for data is contained in ``Zend_Feed_Writer_Entry``.

.. table:: Entry Level API Methods

   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setId()              |Set a unique ID associated with this entry. For Atom 1.0 this is an atom:id element, whereas for RSS 2.0 it is added as a guid element. These are optional so long as a link is added, i.e. the link is set as the ID.                                                                                                                                                                                                                                                                                                                                                       |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setTitle()           |Set the title of the entry.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setDescription()     |Set the text description of the entry.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setContent()         |Set the content of the entry.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setLink()            |Set a URI to the HTML website containing the same or similar information as this entry (i.e. if the feed is from a blog, it should provide the blog article's URI where the HTML version of the entry can be read).                                                                                                                                                                                                                                                                                                                                                          |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setFeedLinks()       |Add a link to an XML feed, whether the feed being generated or an alternate URI pointing to the same feed but in a different format. At a minimum, it is recommended to include a link to the feed being generated so it has an identifiable final URI allowing a client to track its location changes without necessitating constant redirects. The parameter is an array of arrays, where each sub-array contains the keys "type" and "uri". The type should be one of "atom", "rss", or "rdf". If a type is omitted, it defaults to the type used when rendering the feed.|
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |addAuthors()         |Sets the data for authors. The parameter is an array of arrays where each sub-array may contain the keys "name", "email" and "uri". The "uri" value is only applicable for Atom feeds since RSS contains no facility to show it. For RSS 2.0, rendering will create two elements - an author element containing the email reference with the name in brackets, and a Dublin Core creator element only containing the name.                                                                                                                                                   |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |addAuthor()          |Sets the data for a single author following the same format as described above for a single sub-array.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setDateCreated()     |Sets the date on which this feed was created. Generally only applicable to Atom where it represents the date the resource described by an Atom 1.0 document was created. The expected parameter may be a UNIX timestamp or a DateTime object. If omitted, the date used will be the current date and time.                                                                                                                                                                                                                                                                   |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setDateModified()    |Sets the date on which this feed was last modified. The expected parameter may be a UNIX timestamp or a DateTime object. If omitted, the date used will be the current date and time.                                                                                                                                                                                                                                                                                                                                                                                        |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setCopyright()       |Sets a copyright notice associated with the feed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setCategories()      |Accepts an array of categories for rendering, where each element is itself an array whose possible keys include "term", "label" and "scheme". The "term" is a typically a category name suitable for inclusion in a URI. The "label" may be a human readable category name supporting special characters (it is encoded during rendering) and is a required key. The "scheme" (called the domain in RSS) is optional but must be a valid URI.                                                                                                                                |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setCommentCount()    |Sets the number of comments associated with this entry. Rendering differs between RSS and Atom 2.0 depending on the element or attribute needed.                                                                                                                                                                                                                                                                                                                                                                                                                             |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setCommentLink()     |Seta a link to a HTML page containing comments associated with this entry.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setCommentFeedLink() |Sets a link to a XML feed containing comments associated with this entry. The parameter is an array containing the keys "uri" and "type", where the type is one of "rdf", "rss" or "atom".                                                                                                                                                                                                                                                                                                                                                                                   |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setCommentFeedLinks()|Same as setCommentFeedLink() except it accepts an array of arrays, where each subarray contains the expected parameters of setCommentFeedLink().                                                                                                                                                                                                                                                                                                                                                                                                                             |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
   |setEncoding()        |Sets the encoding of entry text. This will default to UTF-8 which is the preferred encoding.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
   +---------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

.. note::

   In addition to these setters, there are also matching getters to retrieve data from the Entry data container.


