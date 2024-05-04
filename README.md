# ServeStatic

<!-- TODO: Fix these links -->
<p>
    <a href="https://github.com/reactive-python/reactpy-django/actions?query=workflow%3ATest">
        <img src="https://github.com/reactive-python/reactpy-django/workflows/Test/badge.svg?event=push">
    </a>
    <a href="https://pypi.python.org/pypi/reactpy-django">
        <img src="https://img.shields.io/pypi/v/reactpy-django.svg?label=PyPI">
    </a>
    <a href="https://github.com/Archmonger/reactpy-django/blob/main/LICENSE.md">
        <img src="https://img.shields.io/badge/License-MIT-purple.svg">
    </a>
    <a href="https://reactive-python.github.io/reactpy-django/">
        <img src="https://img.shields.io/website?down_message=offline&label=Docs&logo=read%20the%20docs&logoColor=white&up_message=online&url=https%3A%2F%2Freactive-python.github.io%2Freactpy-django%2F">
    </a>
</p>

---

Simplified static file serving for Python [ASGI](https://asgi.readthedocs.io/en/latest/)/[WSGI](https://wsgi.readthedocs.io/en/latest/) web apps.

With a couple of lines of config `ServeStatic` allows your web app to serve its own static files, making it a self-contained unit that can be deployed anywhere without relying on nginx, Amazon S3, or any other external service. This is especially useful on Heroku, OpenShift, and other PaaS providers.

It's designed to work nicely with a CDN for high-traffic sites so you don't have to sacrifice performance to benefit from simplicity.

`ServeStatic` works with any ASGI or WSGI compatible app but has some special auto-configuration features for Django.

`ServeStatic` automatically takes care of best-practices for you, for instance:

- Serving compressed content (gzip and Brotli formats, handling Accept-Encoding and Vary headers correctly)
- Setting far-future cache headers on content which won't change

Worried that serving static files with Python is horribly inefficient? Still think you should be using Amazon S3? Have a look at the [FAQ](#).

---

_This project is a fork of [WhiteNoise](https://github.com/evansd/whitenoise) for continued maintenience and feature updates._
