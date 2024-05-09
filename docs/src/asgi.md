# Using ServeStatic with any ASGI application

!!! Note

    These instructions apply to any ASGI application. However, for Django applications you would be better off using the `ServeStaticMiddleware` class which makes integration easier.

To enable ServeStatic you need to wrap your existing ASGI application in a ServeStatic instance and tell it where to find your static files. For example:

```python
from servestatic import AsgiServeStatic

from my_project import MyASGIApp

application = MyWAGIApp()
application = AsgiServeStatic(application, root="/path/to/static/files")
application.add_files("/path/to/more/static/files", prefix="more-files/")
```

On initialization, ServeStatic walks over all the files in the directories that have been added (descending into sub-directories) and builds a list of available static files. Any requests which match a static file get served by ServeStatic, all others are passed through to the original WSGI application.

!!! Tip

    `AsgiServeStatic` inherits all interfaces from WSGI `ServeStatic` but adds support for ASGI applications. See the `WSGI ServeStatic documentation <wsgi>` for more details.

## AsgiServeStatic API

`AsgiServeStatic` inherits its interface from WSGI `ServeStatic`, however, `application` must be an ASGI application.

See the section on WSGI [`ServeStatic` interface](#) for details.

## Compression Support

See the section on WSGI [`ServeStatic` compression support](#) for details.

## Caching Headers

See the section on WSGI [`ServeStatic` caching headers caching](#) for details.

## Index Files

See the section on WSGI [`ServeStatic` index files](#) for details.

## Using a Content Distribution Network

See the instructions for [using a CDN with Django](#). The same principles apply here although obviously the exact method for generating the URLs for your static files will depend on the libraries you're using.

## Redirecting to HTTPS

See the section on WSGI [`ServeStatic` redirecting to HTTPS](#) for details.

## Configuration attributes

See the section on WSGI [`ServeStatic` configuration attributes](#) for details.
