Enable a webapp to get the optimal `http` implementation based on how
it is deployed. Also provide a way to get the `baseUrl` of the
deployment.

Ways one might deploy a node-based webapp:

* [`http`](https://nodejs.org/api/http.html)
* [`node-fastcgi`](https://github.com/fbbdev/node-fastcgi)
* Phusion Passenger

# Usage

`script.cgi`:

    #!/usr/bin/env node
    'use strict';
    
    const autoserve = require('autoserve');
    
    const app = function (req, res) {
        res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8', });
        // It is recommended to use a framework like express to calculate
        // the host and protocol of your deployment. baseUrl only
        // solves the path portion of the deployment.
        res.end(`Hello, World! I am deployed at https://${req.headers.host}${req.baseUrl}`);
    };
    
    autoserve(app);

## Express Usage

Express doesn’t seem to handle `request.baseUrl` being set. Use
[express-autoserve](https://github.com/binki/express-autoserve).

# API

## Module

The module is invokable and behaves the same as `serve()`.

    const autoserve = require('autoserve');

### `serve(app)`

You normally call this by calling the module itself:

    autoserve(function (request, response) {
        …
    });

This is analogous to the following snippet which directly uses the
[`http`](https://nodejs.org/api/http.html) module. In fact, when the
`http` platform is chosen by `detect()`, it will be equivalent.

    const http = require('http');
    http.createServer(function (request, response) {
        …
    }).listen(3000);

The parameters
[`request`](https://nodejs.org/api/http.html#http_class_http_incomingmessage)
and
[`response`](https://nodejs.org/api/http.html#http_class_http_serverresponse)
shall mimic the behavior of the equivalent `http` module-provided
objects. However, many platforms other than `http` will only support a
subset of operations. An application wishing to be portable can assume
that the following APIs are present:

* `request`
  * [`headers`](https://nodejs.org/api/http.html#http_message_headers)
  * [`method`](https://nodejs.org/api/http.html#http_message_method)
  * [`url`](https://nodejs.org/api/http.html#http_message_url)
* `response`
  * [`end`](https://nodejs.org/api/http.html#http_response_end_data_encoding_callback)
  * [`write`](https://nodejs.org/api/http.html#http_response_write_chunk_encoding_callback) (though the platform might buffer everything instead of streaming chunks).
  * [`writeHead`](https://nodejs.org/api/http.html#http_response_writehead_statuscode_statusmessage_headers)

Many platforms strive to provide undocumented and unmentioned APIs on
`request` and `response` to support popular frameworks such as
Express.

`autoserve` also defines the following properties:

* `request`

  * `baseUrl`: The URI used to access the script. This is useful for
    applications which want to generate fully qualified internal links
    which is useful when issuing redirects (the [HTTP `Location:`
    header](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.30)
    requires fully qualified URIs).

    Note that the URI provided here excludes host and protocol
    fields—it will begin with a `/`. In fact, it will behave quite
    like [Express’s `req.baseUrl`
    property](https://expressjs.com/en/4x/api.html#req.baseUrl).

    The `http` module does not provide this property because it
    doesn’t support mounting applications at anything other than
    `/`. However, in many deployment scenarios, an application will be
    made accessible as a subtree of a host. This might be accomplished
    using rewrites/aliases or implicitly based on the application’s
    position in the filesystem hierarchy.

    For example, Apache’s `mod_fcgid` might be configured as follows.
    This would result in
    [`autoserve-platform-node-fastcgi`](https://github.com/binki/autoserve-platform-node-fastcgi)
    being detected and used:

        # FilesMatch is more secure (exhibiting less confusing behavior) than AddHandler.
        <FilesMatch "\.cgi$">
            SetHandler fcgid-script
            Options +ExecCGI
        </FilesMatch>

    And a request might come in for
    `/some/path/script.cgi/sub/resource`. In this case, `baseUrl` will
    be `/some/path/script.cgi` while `url` will be
    `/some/path/script.cgi/sub/resource`. Thus, `baseUrl` is
    implicitly calculated by where the script is placed in the
    filesystem. If you had a `RewriteRule` or `Alias` that runs
    `script.cgi` when `/some/path/` is accessed, the `baseUrl` would
    be explicitly set to the aliased path.
