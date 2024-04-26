## Debugging API Calls Like A ~~Boob~~ Boss

Recently, I stood up a simple JSON web service on Apache in my usual way, the basis for an API for new client calls to be expected from a commercial system used in $dayjob.  Tests of the API with a remote Python script, and by a colleague using Postman, were both successful.

The calls from the commerical system hung.  And hung.  And finally timed out.

Friends, your author gets perplexed more easily than you might suspect.  When a certain approach works, and works, and continues to work some two decades or more running... the mind grows lazy.  Happy experience creates a box diffcult to think outside of.

Inside this mental box, I couldn't believe these client calls were even making it to my server.  What's a hang?  It's the network, of course.  It's always the network, when things are hanging.  Right?

After careful research (okay fine, after a conversation with chatGPT and various AIs) I arrived at the following effective `tcpdump` command with human readable output, and I was able to see...

```
sudo tcpdump -i [network interface] -n -t -vv -X -A src [IP address]
```

... that I was wrong (gasp!): the client call *was* making it to my server.  Something *else* had to be hanging things up.

Careful of the examination of the headers being sent by the commercial system revealed this guy:

```
Expect: 100-continue
```

I wasn't familiar with this.  Developer.mozilla.org says:

>The HTTP 100 Continue informational status response code indicates that everything so far is OK and that the client should continue with the request or ignore it if it is already finished.

Apparently this has wide browser support and is generally not a nuisance.  But as it happens, the Microsoft .NET client (making the calls from our commercial system) behaves in a default manner incompatible with Apache's implementation of the 100-continue message (I'm running httpd 2.4.6).  This is all immortalized in  [the Stack Overflow question](https://stackoverflow.com/questions/3889574/apache-and-mod-proxy-not-handling-http-100-continue-from-client-http-417) where brave developers before me struggled with and finally slew the dragon by applying the hacky strategy of simply stripping the `Expect: 100-continue` header early enough in the request cycle to allow Apache to ignore it:

```
# this was placed in the Apache config file
<IfModule mod_headers.c>
    RequestHeader unset Expect early
</IfModule>
```
A restart of Apache, and problem solved.  But to lose good working hours to what amounts to a lack of cooperation between industry and the open source community is an all too common frustration in this business.





