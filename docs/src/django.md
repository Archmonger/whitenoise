# Using ServeStatic with Django

<div class="note">

<div class="title">

Note

</div>

To use ServeStatic with a non-Django application see the `generic WSGI documentation <wsgi>` or the `generic ASGI documentation <asgi>`.

</div>

This guide walks you through setting up a Django project with ServeStatic. In most cases it shouldn't take more than a couple of lines of configuration.

I mention Heroku in a few places as that was the initial use case which prompted me to create ServeStatic, but there's nothing Heroku-specific about ServeStatic and the instructions below should apply whatever your hosting platform.

## 1. Make sure _staticfiles_ is configured correctly

If you're familiar with Django you'll know what to do. If you're just getting started with a new Django project then you'll need add the following to the bottom of your `settings.py` file:

```python
STATIC_ROOT = BASE_DIR / "staticfiles"

```

As part of deploying your application you'll need to run `./manage.py collectstatic` to put all your static files into `STATIC_ROOT`. (If you're running on Heroku then this is done automatically for you.)

Make sure you're using the [static](https://docs.djangoproject.com/en/stable/ref/templates/builtins/#std:templatetag-static) template tag to refer to your static files, rather than writing the URL directly. For example:

```django
{% load static %}
<img src="{% static "images/hi.jpg" %}" alt="Hi!">

<!-- DON'T WRITE THIS -->
<img src="/static/images/hi.jpg" alt="Hi!">
```

For further details see the Django
[staticfiles](https://docs.djangoproject.com/en/stable/howto/static-files/)
guide.

## 2. Enable ServeStatic

Edit your `settings.py` file and add ServeStatic to the `MIDDLEWARE` list. The ServeStatic middleware should be placed directly after the Django
[SecurityMiddleware](https://docs.djangoproject.com/en/stable/ref/middleware/#module-django.middleware.security)
(if you are using it) and before all other middleware:

```python
MIDDLEWARE = [
    # ...
    "django.middleware.security.SecurityMiddleware",
    "servestatic.middleware.ServeStaticMiddleware",
    # ...
]
```

That's it -- ServeStatic will now serve your static files (you can confirm it's working using the `steps below <check-its-working>`).
However, to get the best performance you should proceed to step 3 below and enable compression and caching.

<div class="note">

<div class="title">

Note

</div>

You might find other third-party middleware that suggests it should be given highest priority at the top of the middleware list. Unless you understand exactly what is happening you should ignore this advice and always place `ServeStaticMiddleware` above other middleware. If you plan to have other middleware run before ServeStatic you should be aware of the [request_finished bug](https://code.djangoproject.com/ticket/29069) in Django.

</div>

## 3. Add compression and caching support

ServeStatic comes with a storage backend which compresses your files and hashes them to unique names, so they can safely be cached forever. To use it, set it as your staticfiles storage backend in your settings file.

On Django 4.2+:

```python
STORAGES = {
    # ...
    "staticfiles": {
        "BACKEND": "servestatic.storage.CompressedManifestStaticFilesStorage",
    },
}
```

On older Django versions:

```python
STATICFILES_STORAGE = "servestatic.storage.CompressedManifestStaticFilesStorage"
```

This combines automatic compression with the caching behaviour provided by Django's
[ManifestStaticFilesStorage](https://docs.djangoproject.com/en/stable/ref/contrib/staticfiles/#manifeststaticfilesstorage) backend. If you want to apply compression but don't want the caching behaviour then you can use the alternative backend:

```python
"servestatic.storage.CompressedStaticFilesStorage"
```

<div class="note">

<div class="title">

Note

</div>

If you are having problems after switching to the ServeStatic storage backend please see the `troubleshooting guide <storage-troubleshoot>`.

</div>

If you need to compress files outside of the static files storage system you can use the supplied `command line utility <cli-utility>`

### Brotli compression

As well as the common gzip compression format, ServeStatic supports the newer, more efficient [brotli](https://en.wikipedia.org/wiki/Brotli)
format. This helps reduce bandwidth and increase loading speed. To enable brotli compression you will need the [Brotli Python
package](https://pypi.org/project/Brotli/) installed by running `pip install servestatic[brotli]`.

Brotli is supported by [all major browsers](https://caniuse.com/#feat=brotli) (except IE11). ServeStatic will only serve brotli data to browsers which request it so there are no compatibility issues with enabling brotli support.

Also note that browsers will only request brotli data over an HTTPS connection.

## 4. Use a Content-Delivery Network

The above steps will get you decent performance on moderate traffic sites, however for higher traffic sites, or sites where performance is a concern you should look at using a CDN.

Because ServeStatic sends appropriate cache headers with your static content, the CDN will be able to cache your files and serve them without needing to contact your application again.

Below are instruction for setting up ServeStatic with Amazon CloudFront, a popular choice of CDN. The process for other CDNs should look very similar though.

### Instructions for Amazon CloudFront

Go to CloudFront section of the AWS Web Console, and click "Create Distribution". Put your application's domain (without the http prefix) in the "Origin Domain Name" field and leave the rest of the settings as they are.

It might take a few minutes for your distribution to become active. Once it's ready, copy the distribution domain name into your `settings.py` file so it looks something like this:

```python
STATIC_HOST = "https://d4663kmspf1sqa.cloudfront.net" if not DEBUG else ""
STATIC_URL = STATIC_HOST + "/static/"
```

Or, even better, you can avoid hardcoding your CDN into your settings by
doing something like this:

```python
STATIC_HOST = os.environ.get("DJANGO_STATIC_HOST", "")
STATIC_URL = STATIC_HOST + "/static/"
```

This way you can configure your CDN just by setting an environment
variable. For apps on Heroku, you'd run this command

```bash
heroku config:set DJANGO_STATIC_HOST=https://d4663kmspf1sqa.cloudfront.net
```

### Using compression algorithms other than gzip

By default, CloudFront will discard any `Accept-Encoding` header browsers include in requests, unless the value of the header is gzip. If it is gzip, CloudFront will fetch the uncompressed file from the origin, compress it, and return it to the requesting browser.

To get CloudFront to not do the compression itself as well as serve files compressed using other algorithms, such as Brotli, you must configure your distribution to [cache based on the Accept-Encoding header](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html#compressed-content-custom-origin). You can do this in the `Behaviours` tab of your distribution.

<div class="note">

<div class="title">

Note

</div>

By default your entire site will be accessible via the CloudFront URL. It's possible that this can cause SEO problems if these URLs start showing up in search results. You can restrict CloudFront to only proxy your static files by following `these directions <restricting-cloudfront>`.

</div>

## 5. Using ServeStatic in development

In development Django's `runserver` automatically takes over static file handling. In most cases this is fine, however this means that some of the improvements that ServeStatic makes to static file handling won't be available in development and it opens up the possibility for differences in behaviour between development and production environments. For this reason it's a good idea to use ServeStatic in development as well.

You can disable Django's static file handling and allow ServeStatic to take over simply by passing the `--nostatic` option to the `runserver` command, but you need to remember to add this option every time you call `runserver`. An easier way is to edit your `settings.py` file and add `servestatic.runserver_nostatic` to the top of your `INSTALLED_APPS` list:

```python
INSTALLED_APPS = [
    # ...
    "servestatic.runserver_nostatic",
    "django.contrib.staticfiles",
    # ...
]
```

<div class="note">

<div class="title">

Note

</div>

In older versions of ServeStatic (below v4.0) it was not possible to use `runserver_nostatic` with [Channels](https://channels.readthedocs.io/) as Channels provides its own implementation of runserver. Newer versions of ServeStatic do not have this problem and will work with Channels or any other third-party app that provides its own implementation of runserver.

</div>

## 6. Index Files

When the `SERVESTATIC_INDEX_FILE` option is enabled:

- Visiting `/example/` will serve the file at `/example/index.html`
- Visiting `/example` will redirect (302) to `/example/`
- Visiting `/example/index.html` will redirect (302) to `/example/`

If you want to something other than `index.html` as the index file, then you can also set this option to an alternative filename.

## Available Settings

The ServeStaticMiddleware class takes all the same configuration options as the ServeStatic base class, but rather than accepting keyword arguments to its constructor it uses Django settings. The setting names are just the keyword arguments upper-cased with a 'SERVESTATIC\_' prefix.

<div class="attribute">

SERVESTATIC_ROOT

default  
`None`

Absolute path to a directory of files which will be served at the root of your application (ignored if not set).

Don't use this for the bulk of your static files because you won't benefit from cache versioning, but it can be convenient for files like `robots.txt` or `favicon.ico` which you want to serve at a specific URL.

</div>

<div class="attribute">

SERVESTATIC_AUTOREFRESH

default  
`settings.DEBUG`

Recheck the filesystem to see if any files have changed before responding. This is designed to be used in development where it can be convenient to pick up changes to static files without restarting the server. For both performance and security reasons, this setting should not be used in production.

</div>

<div class="attribute">

SERVESTATIC_USE_FINDERS

default  
`settings.DEBUG`

Instead of only picking up files collected into `STATIC_ROOT`, find and serve files in their original directories using Django's "finders" API. This is useful in development where it matches the behaviour of the old `runserver` command. It's also possible to use this setting in production, avoiding the need to run the `collectstatic` command during the build, so long as you do not wish to use any of the caching and compression features provided by the storage backends.

</div>

<div class="attribute">

SERVESTATIC_MAX_AGE

default  
`60 if not settings.DEBUG else 0`

Time (in seconds) for which browsers and proxies should cache **non-versioned** files.

Versioned files (i.e. files which have been given a unique name like `base.a4ef2389.css` by including a hash of their contents in the name) are detected automatically and set to be cached forever.

The default is chosen to be short enough not to cause problems with stale versions but long enough that, if you're running ServeStatic behind a CDN, the CDN will still take the majority of the strain during times of heavy load.

Set to `None` to disable setting any `Cache-Control` header on non-versioned files.

</div>

<div class="attribute">

SERVESTATIC_INDEX_FILE

default  
`False`

If `True` enable `index file serving <index-files-django>`. If set to a non-empty string, enable index files and use that string as the index file name.

</div>

<div class="attribute">

SERVESTATIC_MIMETYPES

default  
`None`

A dictionary mapping file extensions (lowercase) to the mimetype for that extension. For example: :

    {'.foo': 'application/x-foo'}

Note that ServeStatic ships with its own default set of mimetypes and does not use the system-supplied ones (e.g. `/etc/mime.types`). This ensures that it behaves consistently regardless of the environment in which it's run. View the defaults in the `media_types.py <servestatic/media_types.py>` file.

In addition to file extensions, mimetypes can be specified by supplying the entire filename, for example: :

    {'some-special-file': 'application/x-custom-type'}

</div>

<div class="attribute">

SERVESTATIC_CHARSET

default  
`'utf-8'`

Charset to add as part of the `Content-Type` header for all files whose mimetype allows a charset.

</div>

<div class="attribute">

SERVESTATIC_ALLOW_ALL_ORIGINS

default  
`True`

Toggles whether to send an `Access-Control-Allow-Origin: *` header for all static files.

This allows cross-origin requests for static files which means your static files will continue to work as expected even if they are served via a CDN and therefore on a different domain. Without this your static files will _mostly_ work, but you may have problems with fonts loading in Firefox, or accessing images in canvas elements, or other mysterious things.

The W3C [explicitly state](https://www.w3.org/TR/cors/#security) that this behaviour is safe for publicly accessible files.

</div>

<div class="attribute">

SERVESTATIC_SKIP_COMPRESS_EXTENSIONS

default  
`('jpg', 'jpeg', 'png', 'gif', 'webp','zip', 'gz', 'tgz', 'bz2', 'tbz', 'xz', 'br', 'swf', 'flv', 'woff', 'woff2')`

File extensions to skip when compressing.

Because the compression process will only create compressed files where this results in an actual size saving, it would be safe to leave this list empty and attempt to compress all files. However, for files which we're confident won't benefit from compression, it speeds up the process if we just skip over them.

</div>

<div class="attribute">

SERVESTATIC_ADD_HEADERS_FUNCTION

default  
`None`

Reference to a function which is passed the headers object for each static file, allowing it to modify them.

For example: :

    def force_download_pdfs(headers, path, url):
        if path.endswith('.pdf'):
            headers['Content-Disposition'] = 'attachment'

    SERVESTATIC_ADD_HEADERS_FUNCTION = force_download_pdfs

The function is passed:

headers  
A [wsgiref.headers](https://docs.python.org/3/library/wsgiref.html#module-wsgiref.headers) instance (which you can treat just as a dict) containing the headers for the current file

path  
The absolute path to the local file

url  
The host-relative URL of the file e.g. `/static/styles/app.css`

The function should not return anything; changes should be made by modifying the headers dictionary directly.

</div>

<div class="attribute">

SERVESTATIC_IMMUTABLE_FILE_TEST

default  
See `immutable_file_test <servestatic/middleware.py#L134>` in source

Reference to function, or string.

If a reference to a function, this is passed the path and URL for each static file and should return whether that file is immutable, i.e. guaranteed not to change, and so can be safely cached forever. The default is designed to work with Django's ManifestStaticFilesStorage backend, and any derivatives of that, so you should only need to change this if you are using a different system for versioning your static files.

If a string, this is treated as a regular expression and each file's URL is matched against it.

Example: :

    def immutable_file_test(path, url):
        # Match filename with 12 hex digits before the extension
        # e.g. app.db8f2edc0c8a.js
        return re.match(r'^.+\.[0-9a-f]{12}\..+$', url)

    SERVESTATIC_IMMUTABLE_FILE_TEST = immutable_file_test

The function is passed:

path  
The absolute path to the local file

url  
The host-relative URL of the file e.g. `/static/styles/app.css`

</div>

<div class="attribute">

SERVESTATIC_STATIC_PREFIX

default  
Path component of `settings.STATIC_URL` (with `settings.FORCE_SCRIPT_NAME` removed if set)

The URL prefix under which static files will be served.

Usually this can be determined automatically by using the path component of `STATIC_URL`. So if `STATIC_URL` is `https://example.com/static/` then `SERVESTATIC_STATIC_PREFIX` will be `/static/`.

If your application is not running at the root of the domain and `FORCE_SCRIPT_NAME` is set then this value will be removed from the `STATIC_URL` path first to give the correct prefix.

If your deployment is more complicated than this (for instance, if you are using a CDN which is doing path rewriting) then you may need to configure this value directly.

</div>

<div class="attribute">

SERVESTATIC_KEEP_ONLY_HASHED_FILES

default  
`False`

Stores only files with hashed names in `STATIC_ROOT`.

By default, Django's hashed static files system creates two copies of each file in `STATIC_ROOT`: one using the original name, e.g. `app.js`, and one using the hashed name, e.g. `app.db8f2edc0c8a.js`. If ServeStatic's compression backend is being used this will create another two copies of each of these files (using Gzip and Brotli compression) resulting in six output files for each input file.

In some deployment scenarios it can be important to reduce the size of the build artifact as much as possible. This setting removes the "un-hashed" version of the file (which should be not be referenced in any case) which should reduce the space required for static files by half.

Note, this setting is only effective if the ServeStatic storage backend is being used.

</div>

<div class="attribute">

SERVESTATIC_MANIFEST_STRICT

default  
`True`

Set to `False` to prevent Django throwing an error if you reference a static file which doesn't exist in the manifest. Note, if the static file does not exist, it will still throw an error.

This works by setting the [manifest_strict](https://docs.djangoproject.com/en/stable/ref/contrib/staticfiles/#django.contrib.staticfiles.storage.ManifestStaticFilesStorage.manifest_strict) option on the underlying Django storage instance, as described in the Django documentation:

> If a file isn't found in the `staticfiles.json` manifest at runtime, a > `ValueError` is raised. This behavior can be disabled by subclassing > `ManifestStaticFilesStorage` and setting the `manifest_strict` > attribute to `False` -- nonexistent paths will remain unchanged.

Note, this setting is only effective if the ServeStatic storage backend is being used.

</div>

## Additional Notes

### Django Compressor

For performance and security reasons ServeStatic does not check for new files after startup (unless using Django <span class="title-ref">DEBUG</span> mode). As such, all static files must be generated in advance. If you're using Django Compressor, this can be performed using its [offline compression](https://django-compressor.readthedocs.io/en/stable/usage.html#offline-compression) feature.

---

### Serving Media Files

ServeStatic is not suitable for serving user-uploaded "media" files. For one thing, as described above, it only checks for static files at startup and so files added after the app starts won't be seen. More importantly though, serving user-uploaded files from the same domain as your main application is a security risk (this [blog post](https://security.googleblog.com/2012/08/content-hosting-for-modern-web.html) from Google security describes the problem well). And in addition to that, using local disk to store and serve your user media makes it harder to scale your application across multiple machines.

For all these reasons, it's much better to store files on a separate dedicated storage service and serve them to users from there. The [django-storages](https://django-storages.readthedocs.io/) library provides many options e.g. Amazon S3, Azure Storage, and Rackspace CloudFiles.

---

### How do I know it's working?

You can confirm that ServeStatic is installed and configured correctly by running you application locally with `DEBUG` disabled and checking that your static files still load.

First you need to run `collectstatic` to get your files in the right place:

```bash
python manage.py collectstatic
```

Then make sure `DEBUG` is set to `False` in your `settings.py` and start the server:

```bash
python manage.py runserver
```

You should find that your static files are served, just as they would be in production.

---

### Troubleshooting the ServeStatic Storage backend

If you're having problems with the ServeStatic storage backend, the chances are they're due to the underlying Django storage engine. This is because ServeStatic only adds a thin wrapper around Django's storage to add compression support, and because the compression code is very simple it generally doesn't cause problems.

The most common issue is that there are CSS files which reference other files (usually images or fonts) which don't exist at that specified path. When Django attempts to rewrite these references it looks for the corresponding file and throws an error if it can't find it.

To test whether the problems are due to ServeStatic or not, try swapping the ServeStatic storage backend for the Django one:

```python
STATICFILES_STORAGE = "django.contrib.staticfiles.storage.ManifestStaticFilesStorage"
```

If the problems persist then your issue is with Django itself (try the [docs](https://docs.djangoproject.com/en/stable/ref/contrib/staticfiles/) or the [mailing list](https://groups.google.com/d/forum/django-users)). If the problem only occurs with ServeStatic then raise a ticket on the [issue tracker](https://github.com/evansd/servestatic/issues).

---

### Restricting CloudFront to static files

The instructions for setting up CloudFront given above will result in the entire site being accessible via the CloudFront URL. It's possible that this can cause SEO problems if these URLs start showing up in search results. You can restrict CloudFront to only proxy your static files by following these directions:

1.  Go to your newly created distribution and click "_Distribution Settings_", then the "_Behaviors_" tab, then "_Create Behavior_". Put `static/*` into the path pattern and click "_Create_" to save.
2.  Now select the `Default (*)` behaviour and click "_Edit_". Set "_Restrict Viewer Access_" to "_Yes_" and then click "_Yes, Edit_" to save.
3.  Check that the `static/*` pattern is first on the list, and the default one is second. This will ensure that requests for static files are passed through but all others are blocked.

### Using other storage backends

ServeStatic will only work with storage backends that stores their files on the local filesystem in `STATIC_ROOT`. It will not work with backends that store files remotely, for instance on Amazon S3.

### ServeStatic makes my tests run slow!

ServeStatic is designed to do as much work as possible upfront when the application starts so that it can serve files as efficiently as possible while the application is running. This makes sense for long-running production processes, but you might find that the added startup time is a problem during test runs when application instances are frequently being created and destroyed.

The simplest way to fix this is to make sure that during testing the `SERVESTATIC_AUTOREFRESH` setting is set to `True`. (By default it is `True` when `DEBUG` is enabled and `False` otherwise.) This stops ServeStatic from scanning your static files on start up but other than that its behaviour should be exactly the same.

It is also worth making sure you don't have unnecessary files in your `STATIC_ROOT` directory. In particular, be careful not to include a `node_modules` directory which can contain a very large number of files and significantly slow down your application startup. If you need to include specific files from `node_modules` then you can create symlinks from within your static directory to just the files you need.

### Why do I get "ValueError: Missing staticfiles manifest entry for ..."?

If you are seeing this error that means you are referencing a static file in your templates (using something like `{% static "foo" %}` which doesn't exist, or at least isn't where Django expects it to be. If you don't understand why Django can't find the file you can use

```sh
python manage.py findstatic --verbosity 2 foo
```

which will show you all the paths which Django searches for the file "foo".

If, for some reason, you want Django to silently ignore such errors you can set `SERVESTATIC_MANIFEST_STRICT` to `False`.

### Using ServeStatic with Webpack / Browserify / $LATEST_JS_THING

A simple technique for integrating any frontend build system with Django is to use a directory layout like this:

```sh
./static_src
        ↓
  $ ./node_modules/.bin/webpack
        ↓
./static_build
        ↓
  $ ./manage.py collectstatic
        ↓
./static_root
```

Here `static_src` contains all the source files (JS, CSS, etc) for your project. Your build tool (which can be Webpack, Browserify or whatever you choose) then processes these files and writes the output into `static_build`.

The path to the `static_build` directory is added to `settings.py`:

```python
STATICFILES_DIRS = [BASE_DIR / "static_build"]
```

This means that Django can find the processed files, but doesn't need to know anything about the tool which produced them.

The final `manage.py collectstatic` step writes "hash-versioned" and compressed copies of the static files into `static_root` ready for production.

Note, both the `static_build` and `static_root` directories should be excluded from version control (e.g. through `.gitignore`) and only the `static_src` directory should be checked in.

### Deploying an application which is not at the root of the domain

Sometimes Django apps are deployed at a particular prefix (or "subdirectory") on a domain e.g. <https://example.com/my-app/> rather than just <https://example.com>.

In this case you would normally use Django's [FORCE_SCRIPT_NAME](https://docs.djangoproject.com/en/1.11/ref/settings/#force-script-name) setting to tell the application where it is located. You would also need to ensure that `STATIC_URL` uses the correct prefix as well. For example:

```python
FORCE_SCRIPT_NAME = "/my-app"
STATIC_URL = FORCE_SCRIPT_NAME + "/static/"
```

If you have set these two values then ServeStatic will automatically configure itself correctly. If you are doing something more complex you may need to set `SERVESTATIC_STATIC_PREFIX` explicitly yourself.
