couch_view_merger
=================

Extracted from Couchbase Server an apache couchdb based product for the
Refuge project.

Description
-----------

(From [initial commit](https://github.com/couchbase/couchdb/commit/3b80357a37b7c717f096a11fae36b48515cd2d87))

This feature allows for merging view streams from several
local and remote databases.

For map views, the source databases do not need to have
the same map function code, that is, we can merge arbitry
map-only views.

For reduce views however, all the source databases must have
exactly the same reduce function code.
This is a necessary restriction due to the rereduce
operations that might be needed during the merge phase.
However, it a "rereduce" parameter is supplied, which is
a string encoding a rereduce function, it's possible to
combine/merge arbitrary reduce views. Bear in mind however
that combining values originated by different reduce functions
can yield unexpected results sometimes. The optional parameter
"language" can also be supplied, which tells the server in
which language the rereduce function is written (defaults to
"javascript" if ommitted).

This feature is exposed via the URI /_view_merge/ and accepts
all the query parameters that regular view URIs accept
(except for ?update_seq=true, which is not yet implemented).

E.g.

$ curl -H 'Content-Type: application/json' -X POST \
        http://localhost:5984/_view_merge \
        -d '{"views": { \
              "test_db_1": "test/mapview",
              "http://myserver:5984/test_db_2": "test2/mapview2",
              "test_db_3": "test3/mapview3"
            }}'

{"total_rows":20,"rows":[
    {"id":"1","key":1,"value":"1"},
    {"id":"2","key":2,"value":"2"},
    {"id":"3","key":3,"value":"3"},
    {"id":"4","key":4,"value":"4"},
    {"id":"5","key":5,"value":"5"},
    {"id":"6","key":6,"value":"6"},
    {"id":"7","key":7,"value":"7"},
    {"id":"8","key":8,"value":"8"},
    {"id":"9","key":9,"value":"9"},
    {"id":"10","key":10,"value":"10"},
    {"id":"11","key":11,"value":"11"},
    {"id":"12","key":12,"value":"12"},
    {"id":"13","key":13,"value":"13"},
    {"id":"14","key":14,"value":"14"},
    {"id":"15","key":15,"value":"15"},
    {"id":"16","key":16,"value":"16"},
    {"id":"17","key":17,"value":"17"},
    {"id":"18","key":18,"value":"18"},
    {"id":"19","key":19,"value":"19"},
    {"id":"20","key":20,"value":"20"}
]}

The HTTP GET verb is also supported (query with ?views=encoded_json_object).

The view merging spec object can also describe a chained view merging
strategy, allowing for a multi level tree structured view merging.
This specification is done by nesting view spec objects inside view spec
objects. Example:

{
    "views": {
         "test_db_a": "test/redview1",
         "http://serverB:5984/_view_merge": {
              "views": {
                   "test_db_b": "test/redview2",
                   "http://foobar:5985/test_db_c": "test/redview3"
              }
         }
    }
}

There's also support for a policy about what to do when streaming
from a remote server fails. This policy is specified the query
parameter "on_error" which can have 2 possible values:
"continue" or "stop".

"continue" means that if streaming from a remote server fails (whether
at the beginning or in the middle), we continue feeding the client with
view results from the other souces and add a JSON view row which
indicates there was an error streaming from that particular source,
example:

{"total_rows":50,"rows":[
	{"id":"1","key":1,"value":"1"},
	{"id":"2","key":2,"value":"2"},
	{"id":"3","key":3,"value":"3"},
	{"id":"4","key":4,"value":"4"},
	{"id":"6","key":6,"value":"6"},
	(....)
],
    "errors":[
        {"from":"local","reason":"Design document `_design/testfoobar` missing in database `test_db_b`."},
        {"from":"http://localhost:5984/_view_merge/","reason":"Design document `_design/testfoobar` missing in database `test_db_c`."}
}

The "stop" policy means that when we get an error while streaming from
a remote server we stop the view merging process and the last row sent
to the client is an error signaling row like the one given in the
previous example.

There's also a parameter "connection_timeout" (value in milliseconds)
that can be specified per /_view_merge/ request. It defaults to 10s.

