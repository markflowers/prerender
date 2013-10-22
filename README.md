Forked from `http://github.com/collectiveip/prerender` to allow setting the port number for local development.

Prerender Service
=========================== 

This is a node server from [prerender.io](http://prerender.io) that uses phantomjs to render a javascript-rendered page as HTML.

It should be used in conjunction with [these middleware libraries](#middleware) to serve the prerendered HTML to crawlers for SEO. Facebook and Twitter also crawl the prerendered HTML when someone posts a link to your site on their social network. You don't have to run this service on your own since I have it deployed on Heroku already. Get started in two lines of code using [Rails](https://github.com/collectiveip/prerender_rails) or [Node](https://github.com/collectiveip/prerender-node). 

This service also adheres to google's `_escaped_fragment_` proposal for AJAX calls if you use it on your website.

It is also meant to be proxied through your server so that any relative links to things like CSS will work.

You can use Amazon S3 to [cache your prerendered HTML](#s3-html-cache).

It is currently deployed at `http://prerender.herokuapp.com`, or you can deploy your own.



## Deploying your own

	$ git clone https://github.com/collectiveip/prerender.git
	$ heroku create
	$ git push heroku master



## Running locally
If you are running the prerender service locally. Make sure you set your middleware to point to your local instance with:
`export PRERENDER_SERVICE_URL=<your local url>`
Otherwise, it will 404 and your normal routing will take over and render the normal JS page.
	
	$ npm install
	$ node index.js
	// also supports heroku style invocation using foreman
	$ foreman start



## How it works
This is a simple service that only takes a url and returns the rendered HTML (with all script tags removed).

Note: you should proxy the request through your server so that relative links to CSS still work (see [prerender_rails](https://github.com/collectiveip/prerender_rails) or [prerender-node](https://github.com/collectiveip/prerender-node) for an example)

`GET` http://prerender.herokuapp.com/https://google.com

`GET` http://prerender.herokuapp.com/https://google.com/search?q=angular



## Plugins

We use a plugin system in the same way that Connect and Express use middleware. Our plugins are a little different and we don't want to confuse the prerender plugins with the [prerender middleware](#middleware), so we opted to call them "plugins".

Plugins are in the `lib/plugins` directory, and add functionality to the prerender service.

Each plugin can implement any of the plugin methods:

####`init = function(){}`
called when you call `prerender.use(require('my_plugin'));`.

Use this function to initialize defaults.

####`beforePhantomRequest = function(req, res, next){}`
called at the beginning of the request lifecycle, before phantomjs starts to load the url.

Use this function to short circuit the lifecycle.  
Examples:

* Find and return a cached version of the url before loading it.
* Reject a request based on the host sending too many requests per second.

####`pluginsOnPhantomPageCreate = function(req, res, next){}`
called after the phantomjs page has been created.

Use this function to bind custom functions to phantomjs events.  
Example:

* Outputting to the terminal console when phantomjs has console output.

####`afterPhantomRequest = function(req, res, next){}`
called at the end of the request lifecycle, after phantomjs successfully loads the HTML for a url.

Use this function to access/modify the HTML returned from a url.  
Examples:

* Save off the HTML to a cache for quick access later.  
* Change the HTML to remove all script tags.


##### The req object has these extra properties on it that you can access in your plugin.
```js
console.log(req.prerender);

{
	//the url that will be hit (transformed from _escaped_fragment_ if passed in)
	url: 'http://site.com/#!/path/to/a/site',

	//the HTML that came back from the webpage (only in afterPhantomRequest)
	documentHTML: '<html></html>'
}
```

## Available plugins

### remove-script-tags
###### Turn off the remove-script-tags plugin (comment it in `index.js`) to disable script tag removal.

We remove script tags because we don't want any framework specific routing/rendering to happen on the rendered HTML once it's executed by the crawler. The crawlers may not execute javascript, but we'd rather be safe than have something get screwed up.

For example, if you rendered the HTML of an angular page but left the angular scripts in there, your browser would try to execute the angular routing and rendering on a page that no longer has any angular bindings.

### http-headers
###### Turn off the html-headers plugin (comment it in `index.js`) to disable soft-http-headers.

If your Javascript routing has a catch-all for things like 404's, you can tell the prerender service to serve a 404 to google instead of a 200. This way, google won't index your 404's.

Add these tags in the `<head>` of your page if you want to serve soft http headers. Note: Prerender will still send the HTML of the page. This just modifies the status code and headers being sent.

Example: telling prerender to server this page as a 404
```html
<meta name="prerender-status-code" content="404">
```

Example: telling prerender to serve this page as a 302 redirect
```html
<meta name="prerender-status-code" content="302">
<meta name="prerender-header" content="Location: http://www.google.com">
```

### whitelist
###### Turn on the whitelist plugin (uncomment it in `index.js`) to enable the whitelist.

If you only want to allow requests to a certain domain, use this plugin to cause a 404 for any other domains.

You can add the whitelisted domains to the plugin itself, or use the `ALLOWED_DOMAINS` environment variable.

`export ALLOWED_DOMAINS=www.prerender.io,prerender.io`

### blacklist
###### Turn off the blacklist plugin (comment it in `index.js`) to disable the blacklist.

If you want to disallow requests to a certain domain, use this plugin to cause a 404 for the domains.

You can add the blacklisted domains to the plugin itself, or use the `BLACKLISTED_DOMAINS` environment variable.

`export BLACKLISTED_DOMAINS=yahoo.com,www.google.com`


### <a id='s3-html-cache'></a>
### s3-html-cache
###### Turn on the s3-html-cache plugin (uncomment it in `index.js`) to enable s3 caching.

A `GET` request will check S3 for a cached copy. If a cached copy is found, it will return that. Otherwise, it will make the request to your server and then persist the HTML to the S3 cache.

A `POST` request will skip the S3 cache. It will make a request to your server and then persist the HTML to the S3 cache. The `POST` is meant to update the cache.

You'll need to sign up with Amazon Web Services and export these 3 environment variables.

```
$ export AWS_ACCESS_KEY_ID=<aws access key>
$ export AWS_SECRET_ACCESS_KEY=<aws secret access key>
$ export S3_BUCKET_NAME=<bucket name>
```

Warning! Your keys should be kept private and you'll be charged for all files uploaded to S3.

> If Prerender is hosted on a EC2 instance, you can also take advantage of [IAM instance roles](http://aws.typepad.com/aws/2012/06/iam-roles-for-ec2-instances-simplified-secure-access-to-aws-service-apis-from-ec2.html)
so that you don't need to export your AWS credentials.

> You can also export the S3_PREFIX_KEY variable so that the key (which is by default the complete requested URL) is
prefixed. This is useful if you want to organize the snapshots in the same bucket.


### in-memory-html-cache
###### Turn on the in-memory-html-cache plugin (uncomment it in `index.js`) to enable local caching.

The default is an in memory cache but you can easily change it to any caching system compatible with the `cache-manager` nodejs package.

For example, with the request:

`GET` http://prerender.herokuapp.com/https://facebook.com

First time: Overall Elapsed:	00:00:03.3174661

With cache: Overall Elapsed:	00:00:00.0360119


### logger
###### Turn on the logger plugin (uncomment it in `index.js`) to enable logging to the console from phantomjs.

This will show console.log's from the phantomjs page in your local console. Great for debugging.



### <a id='middleware'></a>
## Middleware

This is a list of middleware available to use with the prerender service:

#### Official middleware

###### Javascript
* [prerender-node](https://github.com/collectiveip/prerender-node) (Express)

###### Ruby
* [prerender_rails](https://github.com/collectiveip/prerender_rails) (Rails)

#### Community middleware

###### PHP
* [zfr-prerender](https://github.com/zf-fr/zfr-prerender) (Zend Framework 2)
* [YuccaPrerenderBundle](https://github.com/rjanot/YuccaPrerenderBundle) (Symfony 2)

###### Java
* [prerender-java](https://github.com/greengerong/prerender-java)

###### Nginx
* [Reverse Proxy Example](https://gist.github.com/Stanback/6998085)

###### Apache
* [mod_rewrite](https://gist.github.com/Stanback/7028309)

Request more middleware for a different framework in this [issue](https://github.com/collectiveip/prerender/issues/12).



## License

The MIT License (MIT)

Copyright (c) 2013 Todd Hooper &lt;todd@collectiveip.com&gt;

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
