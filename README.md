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