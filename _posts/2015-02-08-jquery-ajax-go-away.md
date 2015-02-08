---
layout: post
title:  "jQuery AJAX please go away"
date:   2015-02-08 12:28:32
categories: web jquery ajax
---
jQuery AJAX API is crap. That probably isn't a controversial statement but if you aren't convinced, here's some problems with it:

- HTTP method is specified using a parameter called type instead of method. GET, POST etc. are called methods of the request in the [HTTP spec][http-spec-methods] instead of something else.
- Error handlers aren't given a Javascript Error object. Instead you get a jqXHR object, string of the request status and a string with the error's name that was thrown. What!?
- If you want to use promises instead of callbacks, good luck! The promise implementation of jQuery is quite [broken][jquery-deferred-problems].

There's more but those came quickly to me.

I got fed up with it and decided to look for alternatives. Easily the most popular JS request library seems to be the aptly named [request][request-js] library. I glanced through the documentation quickly and the library seemed quite large. I compiled the library to a standalone bundle to find out how big it actually is.

{% highlight bash %}
$ npm install request

$ browserify -r request > bundle.js

$ uglifyjs bundle.js -c > bundle.min.js

$ $ ls -alh
-rw-r--r--   1 hilzu  staff   1.3M Feb  7 18:10 bundle.js
-rw-r--r--   1 hilzu  staff   769K Feb  7 18:13 bundle.min.js
{% endhighlight %}

Woah! 1.3M bundled and 769K minified. That's huge (for reference AngularJS is 123K minified). That's way too large for browsers. Time to look for simpler alternatives!

I really wanted a small focused library that returns promises so I can combo it with [RxJS][rxjs] (flatMap promises BOOM!).

- [Reqwest][reqwest]: Most popular alternative. Doesn't seem to return promises. Not updated in a while (2 Nov 2014).
- [request-promise][request-promise]: Now this seems promising! ...oh it's just a wrapper for [request][request-js] and additionally pulls [bluebird][bluebird] AND [lodash][lodash] as dependecies (not exactly small libraries). Pass!
- [then-request][then-request]: A simple API, return promises, nothing big as dependency, actively developed. I think we have a winner!

With then-request JSON request are a bit annoying. I wrote a nice `jsonRequest` wrapper that you can use!

{% gist 36247054ebf8b56e0e1c %}

This doesn't have the callback as an alternative to the promise but you don't really want it. You also pass your non-string payload directly as options.body instead of a separate options.json parameter. The body is stringified by `JSON.parse` automatically if it isn't a string.

With this making AJAX requests from the browser to a JSON backend is enjoyable again.

[http-spec-methods]: http://tools.ietf.org/html/rfc7231#section-4
[jquery-deferred-problems]: http://stackoverflow.com/questions/23744612/problems-inherent-to-jquery-deferred/23744774#23744774
[request-js]: https://github.com/request/request
[rxjs]: https://github.com/Reactive-Extensions/RxJS
[reqwest]: https://github.com/ded/reqwest
[bluebird]: https://github.com/petkaantonov/bluebird
[lodash]: https://lodash.com
[request-promise]: https://github.com/tyabonil/request-promise
[then-request]: https://github.com/then/request
