# doc_server

Documentation server which provides JSON for clients of the Kershner
Trading Group cloud.

## Required conda packages for checking docs

```
conda install -c http://kts02:8000 demjson
```

## Adding a New Doc Group

To create a new group of pages in the documentation, create a subdirectory
in the interfaces/ directory.  For example:

```
mkdir interfaces/lessons
```

Then inside this directory, create a JSON file with the same name as the
directory and the first version of tradesim where this documentation is
applicable.  For testing, we used to use the version "9999", though that
number is now obviously too low to use any longer.  However, here is
an example for lessons_9999.json:

<pre>
{
    "name": "lessons",
    "title": "Lessons",
    "type": "doc_group",
    "pages": [
        {
            "title": "Lesson 1",
            "slug": "lesson_001",
            "subPages": [
                {
                    "title": "Part 1",
                    "slug": "part1",
                    "format": "markdown",
                    "content": "{{lesson_001_9999.md}}"
                }
            ]
        }
    ]
}
</pre>

The content string includes a special pattern "{{filename}}" which is
automatically replaced by the doc server with the contents of that file. These
subsituted files do not have to have a version number in the name, but they
can if that is helpful for organization.  (This way you do not have to copy
every file to a new number if you update the docs for a new tradesim release,
just the file you need to modify.)

## Viewing the Cache

A new API call has been added to allow viewing some internal details
in the doc server.  The structure of the returned data is NOT stable,
so reserve the results for human consumption:

```
curl -s -G http://krtathdocs:6001/v1/internals | json_pp
```

Port notes: 5000/5001 are prod, 6000/6001 are qa, final 1 is the doc server
itself, and final 0 is the nginx proxy in front of the doc_server.

## Validating All The Things with bin/precommit-check

A new program -  bin/precommit-check - allows the markdown to be validated
using our internally-extension jinja.  This doesn't cover python code testing
(see the nosetests section) but does cover many other things.
For an overview, run:

```
bin/precommit-check --help
```

It will report back with Valid, Ignoreable, Suspect, Invalid, or Unknown
for each file.  Anything noted as Invalid should be corrected before
committing it.

"Suspect" covers several issues (some legacy), such as scripts with
aberrant extensions/permissions/headers/etc, which don't necessarily
prevent them from working, but should NOT be allowed in new code.

Some notes on the file types it understands follow.

### Markdown *.md

Markdown files are fully verified, including our jinja extensions using
fetch_* functions, and are confirmed to be pure ASCII to prevent copy+paste
and magical quoting from filly our docs with examples that don't actually
work.

### Shell scripts and files

Bash and other shell scripts are checked for having interpreter directives
and execute permission (scripts), or lack of execute permission and having
and extension ("."/"source"-able function libraries, etc).

### Python

Python is checked much like shell scripts, with the added concern that
nosetest cautiously skips testing on executable python scripts.

### Text *.txt

Text files are validated as being either ASCII or UTF-8 only.

### JSON *.json

JSON files are validated as being parseable, but not yet for schema
compliance (this functionality will be pulled in from check_json.py soon).

## Verifying the JSON

Before checking in a JSON update, it is good to verify the JSON is correct:

```
python doc_server/utils/check_json.py
```

This will show you an error if the JSON is invalid before you check in
documentation changes.  If invalid JSON is checked in, the doc server will
automatically skip that file until it is fixed.

## Validating the Python with Nosetests

The recommended command currently is:

```
nosetests --with-coverage
```

The tests/ directory doesn't need to be specified, since nosetests will
automatically find tests there, as well as anywhere else they get put,
such as a foo_test.py file as a sibling to foo.py.

Specific tests can also be run - pay attention to the colon before the
class names:

<pre>
   nosetests doc_server.enumerations_test:TestEnumeration
   nosetests --verbose test.test_doc_server
   nosetests test.test_doc_server:TestCache
   nosetests test.test_doc_server:TestCache.test_setitem
</pre>

## Markup Extensions

Several functions are provided to simplify certain things, including:

* `fetch_image()` - show an archived image or from a URL (no file:// URLs)
* `fetch_interface()` - show the prototype/interface for a function or similar
* `fetch_sample()` - show a use sample
* `fetch_params_table()` - show all the parameters in detail
* `fetch_attributes_table()` - show all the attribute in detail
* `fetch_returns_table()` - show details about the return value(s)
* `fetch_video(None, youtube='...')` - embed an iframe of a YouTube video

See `doc_server/expand_md_jinja.py` for detailed descriptions of the
arguments to these functions.  The fetch_video() params are slightly different,
since currently it only supports YouTube videos using the v=... value, and
we don't currently have references to videos in the core.json data.

### Examples from algo_buy_3600:

<pre>{{ fetch_interface('cloudquant.Order.methods.algo_buy', lang='python') }}
{{ fetch_sample('cloudquant.Order.methods.algo_buy', lang='python') }}
{{ fetch_params_table('cloudquant.Order.methods.algo_buy') }}
{{ fetch_returns_table('cloudquant.Order.methods.algo_buy') }}
</pre>

### Adding Images:

Images can be added to the repository under the `docstatic/` directory.
However, kindly avoid checking in numerous versions of images to avoid
repository bloat.  To test images locally, one can set up a local
CloudQuant front-end from:

       svn checkout http://thor/svn/infocenter/TATH/trunk

...and a local doc_server from the `branches/qa` subdirectory of:

       svn checkout http://thor/svn/DataAnalysis/tath/doc_server

...and run the infocenter server in local-dev mode with slight modification
to talk to the local doc_server as follows:

       # In the infocenter trunk, run

       gulp local-dev
       perl -pi.bak -e 's@^//.*(ds.baseURL.*http://127.*)@$1@' \
           localserver/config.js

       # The perl command uncomments an override to use localhost instead 
       #   of krtathdocs as the host to contact for the doc service

       # Then run the service:
       node localserver/server.js

...and run the doc_server disconnected from the repository with:

       . activate kershner     # you may need a full path for your "activate"

       # in the doc_server branches/qa directory - if you don't have a local
       # RBAC server running, try RBAC_URL=http://krbatchdev01:7007
       
       RBAC_URL=http://127.0.0.1:7007 DOC_SERVER_STATIC_CONTENT_DIR=$(pwd) \
           python -m doc_server.doc_server

At this point you can freely change images in the `docstatic/` directory
and only need to convince the two servers to reload it fresh, typically
by changing the version in the UI.   This will probably get tidier in the
future.

When any image changes are finalized - as in there's a good chance they'll
never change again - commit them into the doc_server repo (still
in branches/qa) normally.

### Naming images:

Feel free to create subdirectories for images.  Ideally each image would
be clearly tied to a page, so it'd be easy to delete unused ones, but
we may write a query tool to automatically list all images in use by
fetch_image() calls so that any others can be deleted automatically.
In the meantime:

* Be conservative with image use.
* Keep images smallish - no 10 megapixel photos
* Do NOT check videos into the repo!
* For high contrast line graphics, PNG is ideal, or GIF.  JPEG not so much.

### Examples for fetch_image() which is new, so there aren't any examples...:

<pre>{{ fetch_image('cq_logo_red.gif', alt='CloudQuant logo', css_class='', is_url=False) }}</pre>

The "alt" will default to the simple file name without the extension, to
"cq_logo_red" in this case, which isn't all that clear if displayed instead
of the image.  The "is_url" defaults to false (and has to be explicit set
to True to use a full http-URL).  The "css_class" default to nothing, and
there isn't yet any doc on which CSS classes are provided.  So the above
example can be reasonably compressed to:

<pre>{{ fetch_image('cq_logo_red.gif', alt='CloudQuant logo') }}</pre>

### Versioning for RBAC permission support

Both *.json files under interfaces and dictionary nodes in the core.json
file can have a key of "versions" included with an associated dictionary
where each key inside connects the doc_group or core JSON dict with a
product such as "cq", "cqlite", and so on.  These look like:

<pre>    "versions": {
        "cq": {},
        "cqlite": {}
    }</pre>

There are special rules about product propagation through the tree of
nodes in the core and interfaces JSON files.  All "versions" keys
like "cq", "cqlite", etc., plus a notional "any" product that all nodes
keep unless overrun by explicit ones - propagate as follows:

* All products are propagated from larger collections down to all leaf nodes.
* All nodes with specific products mentioned remove the "any" product default.
* All products propagate upward to the top, ensuring a way to read the nodes.
* All empty versions dicts remaining are purged.

* Interfaces JSON: If the result is an empty dict/list/string, the content
                   of a file is interpreted as being entirely empty.

In practice, this means that just attaching version info to any node is
enough to ensure that an authorized user can read it, and gets a path to
find it.

# bot
