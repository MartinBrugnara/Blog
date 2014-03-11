Django and verbs support [REST]
======

### Considerations ###

Quoting "Jespern" - from Django Piston 

> Django doesn't particularly understand REST.  
> In case we send data over PUT,   
> Django won't actually look at the data and load it.  
> We need to twist its arm here. 

Let me add that it currently (1.6) dose not like neither DELETE nor PATCH. 

### Possible solutions / work arounds ###
* Use a REST framework like

    * django-rest
    * django-pisto
    * ...

* or if only the support for all verbs is needed: continue reading.

This code is a generalization of the original version from django-piston. (```coerce_put_post```)
https://bitbucket.org/jespern/django-piston/src/7c90898072ce9462a6023bbec5d408ad097a362b/piston/utils.py?at=default  

```python
def fix_request_payload(request):
    _method = getattr(request, 'method', request.META.get('REQUEST_METHOD'))
    if request.method not in ['POST', 'GET']:
        if hasattr(request, '_post'):
            del request._post
            del request._files

        try:
            request.method = 'POST'
            request._load_post_and_files()
            request.method = _method
        except AttributeError:
            request.META['REQUEST_METHOD'] = 'POST'
            request._load_post_and_files()
            request.META['REQUEST_METHOD'] = _method

        setattr(request, _method, request.POST)
```


### The *magic* explained ###
The Request method ```_load_post_and_files(self)``` aims to  
"Populate ```self._post``` and ```self._files``` if the content-type is a form type"  
https://github.com/django/django/blob/master/django/http/request.py

What is going on here:

+ Dressing up the *unsupported* methods as POST.
+ Django parse the request and create ```request.POST``` and ```request.FILE``` dictionaries.
+ Reset original request method.
+ Copy the just created ```DATA``` dictionary to a more appropriate attribute   
    (```request.PUT```, ```request.PATCH```, ```request.DELETE```, ...) 

---

### What about ```django.text.Client``` and REST api testing ?

If django does not consider the possibility to send data with verbs other than GET and POST,
why should it's test client configured to do that?<br>
But it's not so bad.<br>
In fact all the code is yet there.<br>

```python
...  [287..294]
def post(self, path, data=None, content_type=MULTIPART_CONTENT,
             secure=False, **extra):
        "Construct a POST request."

        post_data = self._encode_data(data or {}, content_type)

        return self.generic('POST', path, post_data, content_type,
                            secure=secure, **extra)

...  [311..315]:
def put(self, path, data='', content_type='application/octet-stream',
            secure=False, **extra):
        "Construct a PUT request."
        return self.generic('PUT', path, data, content_type,
                            secure=secure, **extra)


... [329..335]:
def generic(self, method, path, data='',
                content_type='application/octet-stream', secure=False,
                **extra):
        """Constructs an arbitrary HTTP request."""
        parsed = urlparse(path)
        data = force_bytes(data, settings.DEFAULT_CHARSET)
        r = {
            'PATH_INFO': self._get_path(parsed),
            'REQUEST_METHOD': str(method),
            'SERVER_PORT': str('443') if secure else str('80'),
            'wsgi.url_scheme': str('https') if secure else str('http'),
        }
        if data:
            r.update({
                'CONTENT_LENGTH': len(data),
                'CONTENT_TYPE': str(content_type),
                'wsgi.input': FakePayload(data),
            })
        r.update(extra)
        # If QUERY_STRING is absent or empty, we want to extract it from the URL.
        if not r.get('QUERY_STRING'):
            query_string = force_bytes(parsed[4])
            # WSGI requires latin-1 encoded strings. See get_path_info().
            if six.PY3:
                query_string = query_string.decode('iso-8859-1')
            r['QUERY_STRING'] = query_string
        return self.request(**r)
```

#### Note ####

* post expects __data__ to be a dict (that will be encoded).
* put expects __data__ to be a string, yet encoded.
* Both methods relay on ```generic()```

### Testing so far ###

So when using ```.delete```, ```.put```, ```.patch``` methods we should:

* pass data parameter as yet encoded string.
* pass approriate content_type.

Here is an example using "multipart encoding".


```python
from django.test import Client
from django.test.client import BOUNDARY, MULTIPART_CONTENT, encode_multipart

c = Client()
res = c.put('/my/beautiful/url/',
            data=encode_multipart(BOUNDARY, param_dict),
            content_type=MULTIPART_CONTENT)
```

:q
