CouchDB River Plugin for Elasticsearch
==================================

The CouchDB River plugin allows to hook into couchdb `_changes` feed and automatically index it into elasticsearch.

In order to install the plugin, simply run: `bin/plugin -install elasticsearch/elasticsearch-river-couchdb/2.0.0`.

* For 1.0.x elasticsearch versions, look at [master branch](https://github.com/elasticsearch/elasticsearch-river-couchdb/tree/master).
* For 0.90.x elasticsearch versions, look at [1.x branch](https://github.com/elasticsearch/elasticsearch-river-couchdb/tree/1.x).


|       CouchDB Plugin        |    elasticsearch    | Release date |
|-----------------------------|---------------------|:------------:|
| 2.1.0-SNAPSHOT              | 1.0.0.RC1 -> master |  XXXX-XX-XX  |
| 2.0.0                       | 1.0.0.RC1 -> master |  2014-03-03  |
| 2.0.0.RC1                   | 1.0.0.RC1 -> master |  2014-01-15  |

The CouchDB River allows to automatically index couchdb and make it searchable using the excellent [_changes](http://guide.couchdb.org/draft/notifications.html) stream couchdb provides. Setting it up is as simple as executing the following against elasticsearch:

```sh
curl -XPUT 'localhost:9200/_river/my_db/_meta' -d '{
    "type" : "couchdb",
    "couchdb" : {
        "host" : "localhost",
        "port" : 5984,
        "db" : "my_db",
        "filter" : null
    },
    "index" : {
        "index" : "my_db",
        "type" : "my_db",
        "bulk_size" : "100",
        "bulk_timeout" : "10ms"
    }
}'
```

This call will create a river that uses the `_changes` stream to index all data within couchdb. Moreover, any "future" changes will automatically be indexed as well, making your search index and couchdb synchronized at all times.

The couchdb river is provided as a [plugin](https://github.com/elasticsearch/elasticsearch-river-couchdb) (including explanation on how to install it).

On top of that, in case of a failover, the couchdb river will automatically be started on another elasticsearch node, and continue indexing from the last indexed seq.

Bulking
======

Bulking is automatically done in order to speed up the indexing process. If within the specified `bulk_timeout` more changes are detected,
changes will be bulked up to `bulk_size` before they are indexed.

Since 1.3.0, by default, `bulk` size is `100`. A bulk is flushed every `5s`. Number of concurrent requests allowed to be executed is 1.
You can modify those settings within index section:

```javascript
{
    "type" : "couchdb",
    "index" : {
        "index" : "my_index",
        "type" : "my_type",
        "bulk_size" : 1000,
        "flush_interval" : "1s",
        "max_concurrent_bulk" : 3
    }
}
```

Filtering
======

The `changes` stream allows to provide a filter with parameters that will be used by couchdb to filter the stream of changes. Here is how it can be configured:

```javascript
{
    "couchdb" : {
        "filter" : "test",
        "filter_params" : {
            "param1" : "value1",
            "param2" : "value2"
        }
    }
}
```

Script Filters
=========

Filtering can also be performed by providing a script (default to JavaScript) that will further process each changed item within the changes stream. The json provided to the script is under a var called **ctx** with the relevant seq stream change (for example, **ctx.doc** will refer to the document, or **ctx.deleted** is the flag if its deleted or not).

Note, this feature requires the `lang-javascript` plugin.

Any other script language supported by ElasticSearch may be used by setting the `scriptType` parameter to the appropriate value. If unspecified, the default is "js" (javascript). See http://www.elasticsearch.org/guide/reference/modules/scripting.html for details. 

The **ctx.doc** can be changed and its value can will be indexed (assuming its not a deleted change). Also, if **ctx.ignore** is set to true, the change seq will be ignore and not applied.

Other possible values that can be set are **ctx.index** to control the index name to index the doc into, **ctx.type** to control the (mapping) type to index into, **ctx._parent** and **ctx._routing**.

Here is an example setting that adds `field1` with value `value1` to all docs:

```javascript
{
    "type" : "couchdb",
    "couchdb" : {
        "script" : "ctx.doc.field1 = 'value1'"
    }
}
```

Basic Authentication
===============

Basic Authentication can be used by passing the **user** and **password** attributes.

```javascript
{
    "type" : "couchdb",
    "couchdb" : {
        "user" : "alice",
        "password" : "secret"
    }
}
```

HTTPS
=====

To use HTTPS, pass the **protocol** field. Most likely, you will also have to change the **port**. If you have unfixable problems with the servers certificates for any reason, you can disable hostname verification by passing **no_verify**.

```javascript
{
    "type" : "couchdb",
    "couchdb" : {
        "protocol" : "https",
        "port" : 443,
        "no_verify" : "true"
    }
}
```


Ignoring Attachments
====================

You can ignore attachments as provided by couchDb for each document (`_attachments` field).

Here is an example setting that disable *attachments* for all docs:

```javascript
{
  "type":"couchdb",
  "couchdb": {
    "ignore_attachments":true
  }
}
```


Note, by default, attachments are not ignored (**false**)


Heartbeat
=========

By default, couchdb river set _changes API heartbeat to `10s`.
Since 1.3.0, an additional option has been added to control the HTTP connection timeout (default to `30s`).
you can control both settings using `heartbeat` and `read_timeout` options:

```sh
curl -XPUT 'localhost:9200/_river/my_db/_meta' -d '{
    "type" : "couchdb",
    "couchdb" : {
        "host" : "localhost",
        "port" : 5984,
        "db" : "my_db",
        "heartbeat" : "5s",
        "read_timeout" : "15s"
    }
}'
```

Starting at a Specific Sequence
==========

The CouchDB river stores the `last_seq` value in a document called `_seq` in the `_river` index. You can use this fact to start or resume rivers at a particular sequence.

To have the CouchDB river start at a particular `last_seq`, create a document with contents like this:

````sh
curl -XPUT 'localhost:9200/_river/my_db/_seq' -d '
{
  "couchdb": {
    "last_seq": "100"
  }
}'
````

where 100 is the sequence number you want the river to start from. Then create the `_meta` document as before. The CouchDB river will startup and read the last sequence value and start indexing from there.


Examples
========


Indexing Databases with Multiple Types
-------------------------------------

A common pattern in CouchDB is to have a single database hold documents
of multiple types. Typically each document will have a field containing
the type of the document.

For example, a database of products from Amazon might have a book type:

```json
{
  "_id": 1,
  "type" : "book",
  "author" : "Michael McCandless",
  "title" : "Lucene in Action"
}
```

and a CD type:

```json
{
  "_id": 2,
  "type" : "cd",
  "artist" : "Tool",
  "title" : "Undertow"
}
```

Elasticsearch also supports multiple types in the same index and we need
to tell ES where each type from CouchDB goes. You do this with a river
definition like this:

```json
{
  "type" : "couchdb",
  "couchdb" : {
    "host" : "localhost",
    "port" : 5984,
    "db" : "amazon",
    "script" : "ctx._type = ctx.doc.type"
  },
  "index" : {
    "index" : "amazon"
  }
}
```

Setting  `ctx._type` tells Elasticsearch what type in the index to use.
So if doc.type for a CouchDB changeset is "book" then Elasticsearch will
index it as the "book" type.

The script block can be expanded to handle more complicated cases if
your types don't map one-to-one between the systems.

If you need to also handle deleting documents from the right type in
Elasticsearch, be aware that the above setup requires you to delete
documents from CouchDB in a special way. If you use the `DELETE` `HTTP`
verb with CouchDB you will break the above river as ES will be unable
to determine what type you're trying to delete. Instead you must
preserve some information about the document you deleted by using the
CouchDB bulk document interface.

For example, to delete the above "cd" document, you must do something
like this:

```json
POST /amazon/_bulkdocs HTTP/1.1

{
  "docs" : [
    {
        "_id: 2,
        "_rev" : "rev",
        "_deleted" : true,
        "type" : "cd"
    }
  ]
}
```

This deletes the document while preserving the type information for ES.
You can extend this technique to store more data in deleted documents
but be aware of the disk space usage.


Using CouchDB Views
====================

Since 1.2.0, you can fetch documents from CouchDB views.

You will have to define a view in CouchDB that can be called with the document id fetched from _changes API.
So, CouchDB river will call your view with parameter `key="DOCID"`.

As views return a collection of results (aka rows), each row is indexed with an id like DOCID_seq where seq is the sequence number of each row.
If you get back 3 rows for one single change for document with ID=1234, the river will index 3 documents :

* 1234_1
* 1234_2
* 1234_3
 
To use it, you have to define a view in couchDB. For instance, `_design/myviews/_view/myview` with 

```javascript
function(doc) {
 // YOUR CODE HERE
 emit(doc._id, eval('('+YOURCODEHERE+')') );
}
```

You can use it in your couchDb river as follow :

``` javascript
{
  "type":"couchdb",
  "couchdb": {
    "host":"localhost",
    "port":"5984",
    "db":"mydb",
    "view":"myviews/_view/myview",
    "view_ignore_remove":false
  }
}
```

New options :

* `view` : if not null, couchDB river will not fetch content from `_changes` API but only IDs and then will use the
view to retrieve rows using the ID as a key. By default : null

* `view_ignore_remove` : ask the river to ignore removal of rows if there is less rows after a document update.
By default : false so non existing rows will be removed from elastic search.

For example, with the 3 rows described earlier, if you push a new version of the document 1234 in couchDB with only 2 docs, 

If `view_ignore_remove` is false (default), then

* 1234_1 will be updated
* 1234_2 will be updated
* 1234_3 will be removed

If `view_ignore_remove` is true, then

* 1234_1 will be updated
* 1234_2 will be updated
* 1234_3 will not be updated

Script Views
============

The parameter `script_view` allows to create script filters on the view
results. It's working like the `script` parameter, excepted that:

* you have access to the view result, and not the document,
* you must use `view` instead of `ctx`.

Here is a example which set the field `_parent` to the same value as the field
`field1` of the view:

``` javascript
{
    "type": "couchdb",
    "couchdb": {
        ...
        "view": "myviews/_view/myview",
        "script_view": "view._parent = view.field1"
    }
}
```

License
=======

    This software is licensed under the Apache 2 license, quoted below.

    Copyright 2009-2014 Elasticsearch <http://www.elasticsearch.org>

    Licensed under the Apache License, Version 2.0 (the "License"); you may not
    use this file except in compliance with the License. You may obtain a copy of
    the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
    WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
    License for the specific language governing permissions and limitations under
    the License.
