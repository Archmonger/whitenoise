# Using WhiteNoise with any ASGI application

!!! Note

    These instructions apply to any ASGI application. However, for Django applications you would be better off using the `WhiteNoiseMiddleware` class which makes integration easier.

To enable WhiteNoise you need to wrap your existing ASGI application in a WhiteNoise instance and tell it where to find your static files. For example:

```python
from whitenoise import AsgiWhiteNoise

from my_project import MyASGIApp

application = MyWAGIApp()
application = AsgiWhiteNoise(application, root="/path/to/static/files")
application.add_files("/path/to/more/static/files", prefix="more-files/")
```

On initialization, WhiteNoise walks over all the files in the directories that have been added (descending into sub-directories) and builds a list of available static files. Any requests which match a static file get served by WhiteNoise, all others are passed through to the original WSGI application.

!!! Tip

    `AsgiWhiteNoise` inherits all interfaces from WSGI `WhiteNoise` but adds support for ASGI applications. See the `WSGI WhiteNoise documentation <wsgi>` for more details.

## AsgiWhiteNoise API

`AsgiWhiteNoise` inherits its interface from WSGI `WhiteNoise`, however, `application` must be an ASGI application.

See the section on WSGI [`WhiteNoise` interface](#) for details.

## Compression Support

See the section on WSGI [`WhiteNoise` compression support](#) for details.

## Caching Headers

See the section on WSGI [`WhiteNoise` caching headers caching](#) for details.

## Index Files

See the section on WSGI [`WhiteNoise` index files](#) for details.

## Using a Content Distribution Network

See the instructions for [using a CDN with Django](#). The same principles apply here although obviously the exact method for generating the URLs for your static files will depend on the libraries you're using.

## Redirecting to HTTPS

See the section on WSGI [`WhiteNoise` redirecting to HTTPS](#) for details.

## Configuration attributes

See the section on WSGI [`WhiteNoise` configuration attributes](#) for details.
