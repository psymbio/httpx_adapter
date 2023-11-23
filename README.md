# httpx adapter for pyodide

https://requests.readthedocs.io/en/latest/user/advanced/#transport-adapters

Here, look at the statement:

Many of the details of implementing a Transport Adapter are beyond the scope of this documentation, but take a look at the next example for a simple SSL use- case. For more than that, you might look at subclassing the BaseAdapter.

```python
class requests.adapters.BaseAdapter[source]
The Base Transport Adapter

close()[source]
Cleans up adapter specific items.

send(request, stream=False, timeout=None, verify=True, cert=None, proxies=None)[source]
Sends PreparedRequest object. Returns Response object.

Parameters
request – The PreparedRequest being sent.

stream – (optional) Whether to stream the request content.

timeout (float or tuple) – (optional) How long to wait for the server to send data before giving up, as a float, or a (connect timeout, read timeout) tuple.

verify – (optional) Either a boolean, in which case it controls whether we verify the server’s TLS certificate, or a string, in which case it must be a path to a CA bundle to use

cert – (optional) Any user-provided SSL certificate to be trusted.

proxies – (optional) The proxies dictionary to apply to the request.
```

For httpx see: https://www.python-httpx.org/advanced/#custom-transports

Now, in the code here https://github.com/koenvo/pyodide-http/blob/main/pyodide_http/_requests.py - we see that only send and close are overridden. For, `send` function we keep the **kwargs of the base class:

## Code breakdown
```python
stream = kwargs.get("stream", False)
pyodide_request = Request(request.method, request.url)
pyodide_request.timeout = kwargs.get("timeout", 0)
```
+ kwargs (**kwargs for a variable number of keyword arguments) here if "stream" keyword is not present we fetch the key using get and give it a value of False
+ Request from ._core
+ pyodide_request.timeout is also fetched from kwargs and set to zero


Example code:
```python
car = {
  "brand": "Ford",
  "model": "Mustang",
  "year": 1964,
  "price": 760,
}

x = car.get("price", 15000)

print(x) # -> prints 760 because price exists in dict
# if price was not there then the program would print 15000
```

```python
if isinstance(pyodide_request.timeout, tuple):
    if len(pyodide_request.timeout) > 1:
        pyodide_request.timeout = (pyodide_request.timeout[0] or 0) + (
            pyodide_request.timeout[1] or 0
        )
    elif len(pyodide_request.timeout) > 0:
        pyodide_request.timeout = pyodide_request.timeout[0] or 0
if not pyodide_request.timeout:
    pyodide_request.timeout = 0
pyodide_request.params = None  # this is done in preparing request now
pyodide_request.headers = dict(request.headers)

if request.body:
    pyodide_request.set_body(request.body)
try:
    resp = send(pyodide_request, stream)
except _StreamingTimeout:
    from requests import ConnectTimeout

    raise ConnectTimeout(request=pyodide_request)
except _StreamingError:
    from requests import ConnectionError

    raise ConnectionError(request=pyodide_request)
```
+ if it is a tuple stored in "timeout" then set it to the values 
+ set pyodide_request.params to None
+ here pyodide_request.headers is set to the headers of the parameter variable of the function ("request" i.e. not the library)
+ send the request and catch the error: _StreamingTimeout, _StreamingError

```python
import requests
response = requests.Response()
# Fallback to None if there's no status_code, for whatever reason.
response.status_code = getattr(resp, "status_code", None)
# Make headers case-insensitive.
response.headers = CaseInsensitiveDict(resp.headers)
# Set encoding.
response.encoding = get_encoding_from_headers(response.headers)
if isinstance(resp.body, IOBase):
    # streaming response
    response.raw = resp.body
else:
    # non-streaming response, make it look like a stream
    response.raw = BytesIO(resp.body)
def new_read(self, amt=None, decode_content=False, cache_content=False):
    return self.old_read(amt)

# make the response stream look like a urllib3 stream
response.raw.old_read = response.raw.read
response.raw.read = new_read.__get__(response.raw, type(response.raw))

response.reason = ""
response.url = request.url
response.request = request
return response
```

Complete code for http_pyodide requests
```python
class PyodideHTTPAdapter(BaseAdapter):
    """The Base Transport Adapter"""

    def __init__(self):
        super().__init__()

    def send(self, request, **kwargs):
        """Sends PreparedRequest object. Returns Response object.
        :param request: The :class:`PreparedRequest <PreparedRequest>` being sent.
        :param stream: (optional) Whether to stream the request content.
        :param timeout: (optional) How long to wait for the server to send
            data before giving up, as a float, or a :ref:`(connect timeout,
            read timeout) <timeouts>` tuple.
        :type timeout: float or tuple
        :param verify: (optional) Either a boolean, in which case it controls whether we verify
            the server's TLS certificate, or a string, in which case it must be a path
            to a CA bundle to use
        :param cert: (optional) Any user-provided SSL certificate to be trusted.
        :param proxies: (optional) The proxies dictionary to apply to the request.
        """
        stream = kwargs.get("stream", False)
        pyodide_request = Request(request.method, request.url)
        pyodide_request.timeout = kwargs.get("timeout", 0)
        if isinstance(pyodide_request.timeout, tuple):
            if len(pyodide_request.timeout) > 1:
                pyodide_request.timeout = (pyodide_request.timeout[0] or 0) + (
                    pyodide_request.timeout[1] or 0
                )
            elif len(pyodide_request.timeout) > 0:
                pyodide_request.timeout = pyodide_request.timeout[0] or 0
        if not pyodide_request.timeout:
            pyodide_request.timeout = 0
        pyodide_request.params = None  # this is done in preparing request now
        pyodide_request.headers = dict(request.headers)
        if request.body:
            pyodide_request.set_body(request.body)
        try:
            resp = send(pyodide_request, stream)
        except _StreamingTimeout:
            from requests import ConnectTimeout

            raise ConnectTimeout(request=pyodide_request)
        except _StreamingError:
            from requests import ConnectionError

            raise ConnectionError(request=pyodide_request)
        import requests

        response = requests.Response()
        # Fallback to None if there's no status_code, for whatever reason.
        response.status_code = getattr(resp, "status_code", None)
        # Make headers case-insensitive.
        response.headers = CaseInsensitiveDict(resp.headers)
        # Set encoding.
        response.encoding = get_encoding_from_headers(response.headers)
        if isinstance(resp.body, IOBase):
            # streaming response
            response.raw = resp.body
        else:
            # non-streaming response, make it look like a stream
            response.raw = BytesIO(resp.body)

        def new_read(self, amt=None, decode_content=False, cache_content=False):
            return self.old_read(amt)

        # make the response stream look like a urllib3 stream
        response.raw.old_read = response.raw.read
        response.raw.read = new_read.__get__(response.raw, type(response.raw))

        response.reason = ""
        response.url = request.url
        response.request = request
        return response

    def close(self):
        """Cleans up adapter specific items."""
        pass
```


Requests base adapter looks like:
```python
class BaseAdapter:
    """The Base Transport Adapter"""

    def __init__(self):
        super().__init__()

    def send(
        self, request, stream=False, timeout=None, verify=True, cert=None, proxies=None
    ):
        """Sends PreparedRequest object. Returns Response object.

        :param request: The :class:`PreparedRequest <PreparedRequest>` being sent.
        :param stream: (optional) Whether to stream the request content.
        :param timeout: (optional) How long to wait for the server to send
            data before giving up, as a float, or a :ref:`(connect timeout,
            read timeout) <timeouts>` tuple.
        :type timeout: float or tuple
        :param verify: (optional) Either a boolean, in which case it controls whether we verify
            the server's TLS certificate, or a string, in which case it must be a path
            to a CA bundle to use
        :param cert: (optional) Any user-provided SSL certificate to be trusted.
        :param proxies: (optional) The proxies dictionary to apply to the request.
        """
        raise NotImplementedError

    def close(self):
        """Cleans up adapter specific items."""
        raise NotImplementedError
```

__init_(): just call the parent class definition in child class
send(): instead of args call **kwargs in the child class
close(): will raise NotImplementedError therefore, in child class implement this saying simply: pass
