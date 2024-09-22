+++
title = 'Python, Requests, and SSL'
slug = "python-requests-and-ssl"
aliases = ['/articles/python-requests-and-ssl']
date = 2016-06-20
draft = false
tags = ["python"]
+++

{{< notice warning >}}
This post was written in 2016 when `asyncio` was the highlight feature in the recently released Python 3.5.

There are probably much better ways to handle this now. Proceed with caution!
{{< /notice >}}

## SSL and Synchronous Requests

(scroll down for async requests)

In Python, the main way in which one makes a web request is via the ``requests`` library, like so:

```python
import requests
r = requests.get("http://google.com")
```


Where in this example Google's website is the route that you are interested in. Typically, this would be some API route that returns JSON-encoded data.

Alright, so lets say you're building something for work, and you'd like to hit an internal-API which only accepts connections over HTTPS. Your first approach might be something like this:

```python
import requests
r = requests.get("https://internalsite/api")
```


But this is going to return a Stacktrace with this exception:

```text
requests.exceptions.SSLError: [Errno 1] _ssl.c:503: error:14090086:SSL routines:SSL3_GET_SERVER_CERTIFICATE:certificate verify failed
```

So what can we do here? Well, the easiest is just to disable SSL verfication:

```python
import requests
r = requests.get("https://internalsite/api", verify=False)
```

But then you'll have to look at SSL disabled warnings every time you make a query, and who wants that?

Instead, what we want to do is specify our certificate bundle file location where we've included our certificates for the internal sites we would like to access. There are a few ways to do this with the ``requests`` package.

1) Define the environment variable ``REQUESTS_CA_BUNDLE`` which points to your certificate bundle file. For example: ``/etc/ssl/certs/ca-certificates.crt``. When this variable has been defined, ``requests`` will use it as the default for all connections, and so you don't need to modify any of your code.

2) Fork package [`certifi`](https://github.com/certifi/python-certifi),  add your internal root-CA certificate to this, and then install with ``python setup.py install``. When ``certifi`` is present, ``requests`` will default to using it has the root-CA authority and will do SSL-verification against the certificates found there.

3) Modify your code to point to the certificate bundle file like so:

```python
import requests
r = requests.get("https://internalsite/api", verify="/etc/ssl/certs/ca-certificates.crt")
```

## SSL and Asynchronous Requests

So things are a little bit different with async requests under ``asyncio`` and ``aiohttp``. Instead what we have to do here is create an SSL context with the ``ssl`` standard library, and pass that into the appropriate objects from ``aiohttp``. Here is an example of this in action:

```python {linenos=true}
import asyncio
import aiohttp
import ssl

def foobar():
    # The URLs and headers (blank in this demo) that will be requested async
    routes = [("https://internalsite/api/1", ""), ("https://internalsite/api/2", "")]

    # Create out SSL context object with our CA cert file
    sslcontext = ssl.create_default_context(cafile="/etc/ssl/certs/ca-certificates.crt")

    # Pass this SSL context to aiohttp and create a TCPConnector
    conn = aiohttp.TCPConnector(ssl_context=sslcontext)

    # Using this TCPConnector, open a session
    with aiohttp.ClientSession(connector=conn) as client:
        # This is the asyncio part
        # Create a list of futures
        futures = [
            fetch_json(client, url=url, headers=headers)
            for (url, headers) in routes
        ]

        # Then wait for the futures to all complete
        content = asyncio.get_event_loop().run_until_complete(asyncio.wait(futures))

        # Extract the resulting data
        data = [item.result() for item in content[0]]
    return data

async def fetch_json(client, url, headers):
    async with client.get(url, headers=headers) as resp:
        return await resp.json()
```

(Sidenote: I noticed in some of the discussion on the ``requests`` GitHub page that they would like the ability to take SSL context objects similar to ``aiohttp`` (as shown above) and the standard library ``urllib``)

And that's what you need to do to get your SSL authentication all squared away!
