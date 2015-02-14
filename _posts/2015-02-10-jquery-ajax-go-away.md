---
layout: post
title:  "jQuery AJAX please go away"
date:   2015-02-10 20:40:00
last_modified: 2015-02-14 11:20:00
categories: web jquery ajax
excerpt: jQuery AJAX API is crap. Luckily there are good alternatives.
---
jQuery AJAX API is crap. That probably isn't a controversial statement but if you aren't convinced, here's some problems with it:

- HTTP method is specified using a type parameter instead of method. GET, POST etc. are called methods of the request in the [HTTP spec][http-spec-methods] instead of something else.
- Error handlers aren't given a Javascript Error object. Instead you get a [jqXHR] object, string of the request status and a string with the error's name that was thrown. What!?
- If you want to use promises instead of callbacks, good luck! The promise implementation of jQuery is quite [broken][jquery-deferred-problems].

I got fed up with it and decided to look for alternatives. Easily the most popular JS request library seems to be the aptly named [request]. I glanced through the documentation and the library seemed quite large. I compiled the library to a standalone bundle to find out how big it actually is.

{% highlight bash %}
$ browserify -r request > request.js

$ uglifyjs request.js -c > request.min.js

$ ls -lh
-rw-r--r--   1 hilzu  staff   1.3M Feb 10 16:30 request.js
-rw-r--r--   1 hilzu  staff   769K Feb 10 18:31 request.min.js
{% endhighlight %}

Woah! 1.3M bundled and 769K minified. That's huge! (For reference AngularJS is 123K minified). That's way too large for browsers [[1]](#1). Time to look for simpler alternatives!

I really want a small focused library that returns promises for requests. Promises are nice because I can compose them and pass them to other functions. Promises also work nicely with [RxJS] that I like to use (flatMap promises BOOM!).

Here are some of the alternatives that I found:

- [Reqwest][reqwest]: Most popular alternative. Doesn't seem to return promises. Not updated in a while (Nov 2, 2014).
- [request-promise][request-promise]: Now this seems promising! ...oh it's just a wrapper for [request] and additionally pulls [bluebird] AND [lodash] as dependecies (not exactly small libraries). Pass!
- [then-request][then-request]: A simple API, return promises, nothing big as dependency, actively developed. I think we have a winner!

For comparison I decided to run the same size test for [then-request]:

{% highlight bash %}
$ browserify -r request > request.js

$ uglifyjs request.js -c > request.min.js

$ ls -lh
-rw-r--r--   1 hilzu  staff    26K Feb 10 16:41 then-request.js
-rw-r--r--   1 hilzu  staff    17K Feb 10 16:42 then-request.min.js
{% endhighlight %}

Much better.

I'm mainly sending requests to JSON APIs so ease of use with them is really important to me. With [then-request] JSON bodies can be sent easily using `options.json` and the library does the encoding and `Content-Type` header setting for you.

{% highlight javascript %}
request('POST', 'http://example.com', { json: {field: 42} })
  .then(function (res) { console.log(res.getBody()) })
  .catch(function (err) { console.error(err) })
{% endhighlight %}

Sending GETs and using the response bodies is a different story. With GET requests you usually want to set the `Accept` header to signal what format you except the response to be in. Some APIs require it to return JSON and not XML or something even more horrible. You can of course set the header manually on every request but that's boring.

The library also doesn't parse response bodies. `res.getBody()` above returns the raw string body. Parsing the response yourself and handling errors that `JSON.parse` might throw is annoying in addition to boring.

Luckily it's quite easy to wrap these behaviours in a new function and just use that. In fact I have already done just that for you:

{% gist 36247054ebf8b56e0e1c %}

As you can see I'm augmenting JSON parse errors with the response data so that you can analyze the error more easily. You also don't have worry about that throw. It's done inside promise's `then` handler and just rejects the returned promise with that error. You can handle all errors with just the rejected handler instead of adding `try..catch` around the request function call.

With plain [then-request] you can pass in a callback to handle the response. I've omitted that from my wrapper and instead rely on promises.

Here's an example that retrieves a list of posts from `example.com` and prints the title of the first post to console:

{% highlight javascript %}
jsonRequest('GET', 'http://example.com/posts')
  .then(function (res) { console.log(res.body[0].title) })
  .catch(function (err) { console.error(err) })
{% endhighlight %}

With this making AJAX requests from the browser to a JSON API is enjoyable again.

Edit 2015-02-14: The upcoming [Fetch Standard](https://fetch.spec.whatwg.org/) is also an interesting option in the future as the standard matures, polyfills appear and browers gain support for it. Chrome [already has](http://caniuse.com/#feat=fetch) initial support for the API.

[1] <a name="1"></a>[Request] is probably nice on node but overkill for browser usage.

[http-spec-methods]: http://tools.ietf.org/html/rfc7231#section-4
[jquery-deferred-problems]: http://stackoverflow.com/questions/23744612/problems-inherent-to-jquery-deferred/23744774#23744774
[request]: https://github.com/request/request
[rxjs]: https://github.com/Reactive-Extensions/RxJS
[reqwest]: https://github.com/ded/reqwest
[bluebird]: https://github.com/petkaantonov/bluebird
[lodash]: https://lodash.com
[request-promise]: https://github.com/tyabonil/request-promise
[then-request]: https://github.com/then/request
[jqXHR]: http://api.jquery.com/Types/#jqXHR
