==============================
Chapter 21: Security in Django
==============================

.. admonition:: Chapter Still in Draft!!
	
    This chapter is still a work in progress. You are welcome to browse and
    offer suggestions, but it is likely full of errors, jumbled and not
    recommended for use yet. You have been warned...

    [TODO This is from the Django docs. Need to tidy up the flow and simplify] 

This document is an overview of Django's security features. It includes advice
on securing a Django-powered site.

.. _cross-site-scripting:

Cross site scripting (XSS) protection
=====================================

.. highlightlang:: html+django

XSS attacks allow a user to inject client side scripts into the browsers of
other users. This is usually achieved by storing the malicious scripts in the
database where it will be retrieved and displayed to other users, or by getting
users to click a link which will cause the attacker's JavaScript to be executed
by the user's browser. However, XSS attacks can originate from any untrusted
source of data, such as cookies or Web services, whenever the data is not
sufficiently sanitized before including in a page.

Using Django templates protects you against the majority of XSS attacks.
However, it is important to understand what protections it provides
and its limitations.

Django templates escape specific characters 
which are particularly dangerous to HTML. While this protects users from most
malicious input, it is not entirely foolproof. For example, it will not
protect the following:

.. code-block:: html+django

    <style class={{ var }}>...</style>

If ``var`` is set to ``'class1 onmouseover=javascript:func()'``, this can result
in unauthorized JavaScript execution, depending on how the browser renders
imperfect HTML. (Quoting the attribute value would fix this case.)

It is also important to be particularly careful when using ``is_safe`` with
custom template tags, the ``safe`` template tag, :mod:`mark_safe
<django.utils.safestring>`, and when autoescape is turned off.

In addition, if you are using the template system to output something other
than HTML, there may be entirely separate characters and words which require
escaping.

You should also be very careful when storing HTML in the database, especially
when that HTML is retrieved and displayed.


Cross site request forgery (CSRF) protection
============================================

CSRF attacks allow a malicious user to execute actions using the credentials
of another user without that user's knowledge or consent.

Django has built-in protection against most types of CSRF attacks, providing you
have enabled and used it  where appropriate. However, as with
any mitigation technique, there are limitations. For example, it is possible to
disable the CSRF module globally or for particular views. You should only do
this if you know what you are doing. There are other limitations
if your site has subdomains that are outside of your
control.

CSRF protection works  by checking for a nonce in each
POST request. This ensures that a malicious user cannot simply "replay" a form
POST to your Web site and have another logged in user unwittingly submit that
form. The malicious user would have to know the nonce, which is user specific
(using a cookie).

When deployed with HTTPS ,
``CsrfViewMiddleware`` will check that the HTTP referer header is set to a
URL on the same origin (including subdomain and port). Because HTTPS
provides additional security, it is imperative to ensure connections use HTTPS
where it is available by forwarding insecure connection requests and using
HSTS for supported browsers.

Be very careful with marking views with the ``csrf_exempt`` decorator unless
it is absolutely necessary.

.. _sql-injection-protection:

SQL injection protection
========================

SQL injection is a type of attack where a malicious user is able to execute
arbitrary SQL code on a database. This can result in records
being deleted or data leakage.

By using Django's querysets, the resulting SQL will be properly escaped by
the underlying database driver. However, Django also gives developers power to
write raw queries  or execute
custom sql . These capabilities should be used
sparingly and you should always be careful to properly escape any parameters
that the user can control. In addition, you should exercise caution when using
:meth:`extra() <django.db.models.query.QuerySet.extra>`.

Clickjacking protection
=======================

Clickjacking is a type of attack where a malicious site wraps another site
in a frame. This attack can result in an unsuspecting user being tricked
into performing unintended actions on the target site.

Django contains clickjacking protection  in
the form of the
:mod:`X-Frame-Options middleware <django.middleware.clickjacking.XFrameOptionsMiddleware>`
which in a supporting browser can prevent a site from being rendered inside
a frame. It is possible to disable the protection on a per view basis
or to configure the exact header value sent.

The middleware is strongly recommended for any site that does not need to have
its pages wrapped in a frame by third party sites, or only needs to allow that
for a small section of the site.

.. _security-recommendation-ssl:

SSL/HTTPS
=========

It is always better for security, though not always practical in all cases, to
deploy your site behind HTTPS. Without this, it is possible for malicious
network users to sniff authentication credentials or any other information
transferred between client and server, and in some cases -- **active** network
attackers -- to alter data that is sent in either direction.

If you want the protection that HTTPS provides, and have enabled it on your
server, there are some additional steps you may need:

* If necessary, set ``SECURE_PROXY_SSL_HEADER``, ensuring that you have
  understood the warnings there thoroughly. Failure to do this can result
  in CSRF vulnerabilities, and failure to do it correctly can also be
  dangerous!

* Set up redirection so that requests over HTTP are redirected to HTTPS.

  This could be done using a custom middleware. Please note the caveats under
  ``SECURE_PROXY_SSL_HEADER``. For the case of a reverse proxy, it may be
  easier or more secure to configure the main Web server to do the redirect to
  HTTPS.

* Use 'secure' cookies.

  If a browser connects initially via HTTP, which is the default for most
  browsers, it is possible for existing cookies to be leaked. For this reason,
  you should set your ``SESSION_COOKIE_SECURE`` and
  ``CSRF_COOKIE_SECURE`` settings to ``True``. This instructs the browser
  to only send these cookies over HTTPS connections. Note that this will mean
  that sessions will not work over HTTP, and the CSRF protection will prevent
  any POST data being accepted over HTTP (which will be fine if you are
  redirecting all HTTP traffic to HTTPS).

* Use HTTP Strict Transport Security (HSTS)

  HSTS is an HTTP header that informs a browser that all future connections
  to a particular site should always use HTTPS. Combined with redirecting
  requests over HTTP to HTTPS, this will ensure that connections always enjoy
  the added security of SSL provided one successful connection has occurred.
  HSTS is usually configured on the web server.

.. _host-headers-virtual-hosting:

Host header validation
======================

Django uses the ``Host`` header provided by the client to construct URLs in
certain cases. While these values are sanitized to prevent Cross Site Scripting
attacks, a fake ``Host`` value can be used for Cross-Site Request Forgery,
cache poisoning attacks, and poisoning links in emails.

Because even seemingly-secure web server configurations are susceptible to fake
``Host`` headers, Django validates ``Host`` headers against the
``ALLOWED_HOSTS`` setting in the
:meth:`django.http.HttpRequest.get_host()` method.

This validation only applies via :meth:`~django.http.HttpRequest.get_host()`;
if your code accesses the ``Host`` header directly from ``request.META`` you
are bypassing this security protection.

For more details see the full ``ALLOWED_HOSTS`` documentation.

.. warning::

   Previous versions of this document recommended configuring your web server to
   ensure it validates incoming HTTP ``Host`` headers. While this is still
   recommended, in many common web servers a configuration that seems to
   validate the ``Host`` header may not in fact do so. For instance, even if
   Apache is configured such that your Django site is served from a non-default
   virtual host with the ``ServerName`` set, it is still possible for an HTTP
   request to match this virtual host and supply a fake ``Host`` header. Thus,
   Django now requires that you set ``ALLOWED_HOSTS`` explicitly rather
   than relying on web server configuration.

Additionally, as of 1.3.1, Django requires you to explicitly enable support for
the ``X-Forwarded-Host`` header (via the ``USE_X_FORWARDED_HOST``
setting) if your configuration requires it.

Session security
================

Similar to the CSRF limitations  requiring a site to
be deployed such that untrusted users don't have access to any subdomains,
:mod:`django.contrib.sessions` also has limitations. See the session
topic guide section on security for details.

.. _user-uploaded-content-security:

User-uploaded content
=====================

.. note::
    Consider serving static files from a cloud service or CDN
    to avoid some of these issues.

* If your site accepts file uploads, it is strongly advised that you limit
  these uploads in your Web server configuration to a reasonable
  size in order to prevent denial of service (DOS) attacks. In Apache, this
  can be easily set using the LimitRequestBody_ directive.

* If you are serving your own static files, be sure that handlers like Apache's
  ``mod_php``, which would execute static files as code, are disabled. You don't
  want users to be able to execute arbitrary code by uploading and requesting a
  specially crafted file.

* Django's media upload handling poses some vulnerabilities when that media is
  served in ways that do not follow security best practices. Specifically, an
  HTML file can be uploaded as an image if that file contains a valid PNG
  header followed by malicious HTML. This file will pass verification of the
  library that Django uses for :class:`~django.db.models.ImageField` image
  processing (Pillow). When this file is subsequently displayed to a
  user, it may be displayed as HTML depending on the type and configuration of
  your web server.

  No bulletproof technical solution exists at the framework level to safely
  validate all user uploaded file content, however, there are some other steps
  you can take to mitigate these attacks:

  1. One class of attacks can be prevented by always serving user uploaded
     content from a distinct top-level or second-level domain. This prevents
     any exploit blocked by `same-origin policy`_ protections such as cross
     site scripting. For example, if your site runs on ``example.com``, you
     would want to serve uploaded content (the ``MEDIA_URL`` setting)
     from something like ``usercontent-example.com``. It's *not* sufficient to
     serve content from a subdomain like ``usercontent.example.com``.

  2. Beyond this, applications may choose to define a whitelist of allowable
     file extensions for user uploaded files and configure the web server
     to only serve such files.

.. _same-origin policy: http://en.wikipedia.org/wiki/Same-origin_policy

.. _additional-security-topics:

Additional security topics
==========================

While Django provides good security protection out of the box, it is still
important to properly deploy your application and take advantage of the
security protection of the Web server, operating system and other components.

* Make sure that your Python code is outside of the Web server's root. This
  will ensure that your Python code is not accidentally served as plain text
  (or accidentally executed).
* Take care with any user uploaded files .
* Django does not throttle requests to authenticate users. To protect against
  brute-force attacks against the authentication system, you may consider
  deploying a Django plugin or Web server module to throttle these requests.
* Keep your ``SECRET_KEY`` a secret.
* It is a good idea to limit the accessibility of your caching system and
  database using a firewall.

.. _LimitRequestBody: http://httpd.apache.org/docs/2.2/mod/core.html#limitrequestbody

Archive of security issues
==========================

Django's development team is strongly committed to responsible reporting and
disclosure of security-related issues, as outlined in Django's security
policies.

As part of that commitment, they maintain an historical list of issues which
have been fixed and disclosed. For the up to date list, see https://docs.djangoproject.com/en/1.8/releases/security/

Clickjacking Protection
========================

.. module:: django.middleware.clickjacking
   :synopsis: Protects against Clickjacking

The clickjacking middleware and decorators provide easy-to-use protection
against `clickjacking`_.  This type of attack occurs when a malicious site
tricks a user into clicking on a concealed element of another site which they
have loaded in a hidden frame or iframe.

.. _clickjacking: http://en.wikipedia.org/wiki/Clickjacking

An example of clickjacking
--------------------------

Suppose an online store has a page where a logged in user can click "Buy Now" to
purchase an item. A user has chosen to stay logged into the store all the time
for convenience. An attacker site might create an "I Like Ponies" button on one
of their own pages, and load the store's page in a transparent iframe such that
the "Buy Now" button is invisibly overlaid on the "I Like Ponies" button. If the
user visits the attacker's site, clicking "I Like Ponies" will cause an
inadvertent click on the "Buy Now" button and an unknowing purchase of the item.

.. _clickjacking-prevention:

Preventing clickjacking
-----------------------

Modern browsers honor the `X-Frame-Options`_ HTTP header that indicates whether
or not a resource is allowed to load within a frame or iframe. If the response
contains the header with a value of ``SAMEORIGIN`` then the browser will only
load the resource in a frame if the request originated from the same site. If
the header is set to ``DENY`` then the browser will block the resource from
loading in a frame no matter which site made the request.

.. _X-Frame-Options: https://developer.mozilla.org/en/The_X-FRAME-OPTIONS_response_header

Django provides a few simple ways to include this header in responses from your
site:

1. A simple middleware that sets the header in all responses.

2. A set of view decorators that can be used to override the middleware or to
   only set the header for certain views.

How to use it
-------------

Setting X-Frame-Options for all responses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To set the same ``X-Frame-Options`` value for all responses in your site, put
``'django.middleware.clickjacking.XFrameOptionsMiddleware'`` to
``MIDDLEWARE_CLASSES``::

    MIDDLEWARE_CLASSES = [
        ...
        'django.middleware.clickjacking.XFrameOptionsMiddleware',
        ...
    ]

This middleware is enabled in the settings file generated by
``startproject``.

By default, the middleware will set the ``X-Frame-Options`` header to
``SAMEORIGIN`` for every outgoing ``HttpResponse``. If you want ``DENY``
instead, set the ``X_FRAME_OPTIONS`` setting::

    X_FRAME_OPTIONS = 'DENY'

When using the middleware there may be some views where you do **not** want the
``X-Frame-Options`` header set. For those cases, you can use a view decorator
that tells the middleware not to set the header::

    from django.http import HttpResponse
    from django.views.decorators.clickjacking import xframe_options_exempt

    @xframe_options_exempt
    def ok_to_load_in_a_frame(request):
        return HttpResponse("This page is safe to load in a frame on any site.")


Setting X-Frame-Options per view
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To set the ``X-Frame-Options`` header on a per view basis, Django provides these
decorators::

    from django.http import HttpResponse
    from django.views.decorators.clickjacking import xframe_options_deny
    from django.views.decorators.clickjacking import xframe_options_sameorigin

    @xframe_options_deny
    def view_one(request):
        return HttpResponse("I won't display in any frame!")

    @xframe_options_sameorigin
    def view_two(request):
        return HttpResponse("Display in a frame if it's from the same origin as me.")

Note that you can use the decorators in conjunction with the middleware. Use of
a decorator overrides the middleware.

Limitations
-----------

The ``X-Frame-Options`` header will only protect against clickjacking in a
modern browser. Older browsers will quietly ignore the header and need `other
clickjacking prevention techniques`_.

Browsers that support X-Frame-Options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* Internet Explorer 8+
* Firefox 3.6.9+
* Opera 10.5+
* Safari 4+
* Chrome 4.1+

See also
~~~~~~~~

A `complete list`_ of browsers supporting ``X-Frame-Options``.

.. _complete list: https://developer.mozilla.org/en/The_X-FRAME-OPTIONS_response_header#Browser_compatibility
.. _other clickjacking prevention techniques: http://en.wikipedia.org/wiki/Clickjacking#Prevention

Cross Site Request Forgery protection
=====================================

.. module:: django.middleware.csrf
   :synopsis: Protects against Cross Site Request Forgeries

The CSRF middleware and template tag provides easy-to-use protection against
`Cross Site Request Forgeries`_.  This type of attack occurs when a malicious
Web site contains a link, a form button or some javascript that is intended to
perform some action on your Web site, using the credentials of a logged-in user
who visits the malicious site in their browser.  A related type of attack,
'login CSRF', where an attacking site tricks a user's browser into logging into
a site with someone else's credentials, is also covered.

The first defense against CSRF attacks is to ensure that GET requests (and other
'safe' methods, as defined by 9.1.1 Safe Methods, HTTP 1.1,
:rfc:`2616#section-9.1.1`) are side-effect free. Requests via 'unsafe' methods,
such as POST, PUT and DELETE, can then be protected by following the steps
below.

.. _Cross Site Request Forgeries: http://www.squarefree.com/securitytips/web-developers.html#CSRF

.. _using-csrf:

How to use it
-------------

To take advantage of CSRF protection in your views, follow these steps:

1. The CSRF middleware is activated by default in the
   ``MIDDLEWARE_CLASSES`` setting. If you override that setting, remember
   that ``'django.middleware.csrf.CsrfViewMiddleware'`` should come before any
   view middleware that assume that CSRF attacks have been dealt with.

   If you disabled it, which is not recommended, you can use
   :func:`~django.views.decorators.csrf.csrf_protect` on particular views
   you want to protect (see below).

2. In any template that uses a POST form, use the ``csrf_token`` tag inside
   the ``<form>`` element if the form is for an internal URL, e.g.::

       <form action="." method="post">{% csrf_token %}

   This should not be done for POST forms that target external URLs, since
   that would cause the CSRF token to be leaked, leading to a vulnerability.

3. In the corresponding view functions, ensure that the
   ``'django.template.context_processors.csrf'`` context processor is
   being used. Usually, this can be done in one of two ways:

   1. Use RequestContext, which always uses
      ``'django.template.context_processors.csrf'`` (no matter what template
      context processors are configured in the ``TEMPLATES`` setting).
      If you are using generic views or contrib apps, you are covered already,
      since these apps use RequestContext throughout.

   2. Manually import and use the processor to generate the CSRF token and
      add it to the template context. e.g.::

          from django.shortcuts import render_to_response
          from django.template.context_processors import csrf

          def my_view(request):
              c = {}
              c.update(csrf(request))
              # ... view code here
              return render_to_response("a_template.html", c)

      You may want to write your own
      :func:`~django.shortcuts.render_to_response()` wrapper that takes care
      of this step for you.

.. _csrf-ajax:

AJAX
----

While the above method can be used for AJAX POST requests, it has some
inconveniences: you have to remember to pass the CSRF token in as POST data with
every POST request. For this reason, there is an alternative method: on each
XMLHttpRequest, set a custom ``X-CSRFToken`` header to the value of the CSRF
token. This is often easier, because many javascript frameworks provide hooks
that allow headers to be set on every request.

As a first step, you must get the CSRF token itself. The recommended source for
the token is the ``csrftoken`` cookie, which will be set if you've enabled CSRF
protection for your views as outlined above.

.. note::

    The CSRF token cookie is named ``csrftoken`` by default, but you can control
    the cookie name via the ``CSRF_COOKIE_NAME`` setting.

Acquiring the token is straightforward:

.. code-block:: javascript

    // using jQuery
    function getCookie(name) {
        var cookieValue = null;
        if (document.cookie && document.cookie != '') {
            var cookies = document.cookie.split(';');
            for (var i = 0; i < cookies.length; i++) {
                var cookie = jQuery.trim(cookies[i]);
                // Does this cookie string begin with the name we want?
                if (cookie.substring(0, name.length + 1) == (name + '=')) {
                    cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                    break;
                }
            }
        }
        return cookieValue;
    }
    var csrftoken = getCookie('csrftoken');

The above code could be simplified by using the `jQuery cookie plugin
<http://plugins.jquery.com/cookie/>`_ to replace ``getCookie``:

.. code-block:: javascript

    var csrftoken = $.cookie('csrftoken');

.. note::

    The CSRF token is also present in the DOM, but only if explicitly included
    using ``csrf_token`` in a template. The cookie contains the canonical
    token; the ``CsrfViewMiddleware`` will prefer the cookie to the token in
    the DOM. Regardless, you're guaranteed to have the cookie if the token is
    present in the DOM, so you should use the cookie!

.. warning::

    If your view is not rendering a template containing the ``csrf_token``
    template tag, Django might not set the CSRF token cookie. This is common in
    cases where forms are dynamically added to the page. To address this case,
    Django provides a view decorator which forces setting of the cookie:
    :func:`~django.views.decorators.csrf.ensure_csrf_cookie`.

Finally, you'll have to actually set the header on your AJAX request, while
protecting the CSRF token from being sent to other domains using
`settings.crossDomain <http://api.jquery.com/jQuery.ajax>`_ in jQuery 1.5.1 and
newer:

.. code-block:: javascript

    function csrfSafeMethod(method) {
        // these HTTP methods do not require CSRF protection
        return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
    }
    $.ajaxSetup({
        beforeSend: function(xhr, settings) {
            if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
                xhr.setRequestHeader("X-CSRFToken", csrftoken);
            }
        }
    });

Other template engines
----------------------

When using a different template engine than Django's built-in engine, you can
set the token in your forms manually after making sure it's available in the
template context.

For example, in the Jinja2 template language, your form could contain the
following:

.. code-block:: html

    <div style="display:none">
        <input type="hidden" name="csrfmiddlewaretoken" value="{{ csrf_token }}">
    </div>

You can use JavaScript similar to the AJAX code  above to get
the value of the CSRF token.

The decorator method
--------------------

.. module:: django.views.decorators.csrf

Rather than adding ``CsrfViewMiddleware`` as a blanket protection, you can use
the ``csrf_protect`` decorator, which has exactly the same functionality, on
particular views that need the protection. It must be used **both** on views
that insert the CSRF token in the output, and on those that accept the POST form
data. (These are often the same view function, but not always).

Use of the decorator by itself is **not recommended**, since if you forget to
use it, you will have a security hole. The 'belt and braces' strategy of using
both is fine, and will incur minimal overhead.

.. function:: csrf_protect(view)

    Decorator that provides the protection of ``CsrfViewMiddleware`` to a view.

    Usage::

        from django.views.decorators.csrf import csrf_protect
        from django.shortcuts import render

        @csrf_protect
        def my_view(request):
            c = {}
            # ...
            return render(request, "a_template.html", c)

    If you are using class-based views, you can refer to
    Decorating class-based views.

Rejected requests
=================

By default, a '403 Forbidden' response is sent to the user if an incoming
request fails the checks performed by ``CsrfViewMiddleware``.  This should
usually only be seen when there is a genuine Cross Site Request Forgery, or
when, due to a programming error, the CSRF token has not been included with a
POST form.

The error page, however, is not very friendly, so you may want to provide your
own view for handling this condition.  To do this, simply set the
``CSRF_FAILURE_VIEW`` setting.

.. _how-csrf-works:

How it works
------------

The CSRF protection is based on the following things:

1. A CSRF cookie that is set to a random value (a session independent nonce, as
   it is called), which other sites will not have access to.

   This cookie is set by ``CsrfViewMiddleware``.  It is meant to be permanent,
   but since there is no way to set a cookie that never expires, it is sent with
   every response that has called ``django.middleware.csrf.get_token()``
   (the function used internally to retrieve the CSRF token).

2. A hidden form field with the name 'csrfmiddlewaretoken' present in all
   outgoing POST forms.  The value of this field is the value of the CSRF
   cookie.

   This part is done by the template tag.

3. For all incoming requests that are not using HTTP GET, HEAD, OPTIONS or
   TRACE, a CSRF cookie must be present, and the 'csrfmiddlewaretoken' field
   must be present and correct. If it isn't, the user will get a 403 error.

   This check is done by ``CsrfViewMiddleware``.

4. In addition, for HTTPS requests, strict referer checking is done by
   ``CsrfViewMiddleware``.  This is necessary to address a Man-In-The-Middle
   attack that is possible under HTTPS when using a session independent nonce,
   due to the fact that HTTP 'Set-Cookie' headers are (unfortunately) accepted
   by clients that are talking to a site under HTTPS.  (Referer checking is not
   done for HTTP requests because the presence of the Referer header is not
   reliable enough under HTTP.)

This ensures that only forms that have originated from your Web site can be used
to POST data back.

It deliberately ignores GET requests (and other requests that are defined as
'safe' by :rfc:`2616`). These requests ought never to have any potentially
dangerous side effects , and so a CSRF attack with a GET request ought to be
harmless. :rfc:`2616` defines POST, PUT and DELETE as 'unsafe', and all other
methods are assumed to be unsafe, for maximum protection.

Caching
=======

If the ``csrf_token`` template tag is used by a template (or the
``get_token`` function is called some other way), ``CsrfViewMiddleware`` will
add a cookie and a ``Vary: Cookie`` header to the response. This means that the
middleware will play well with the cache middleware if it is used as instructed
(``UpdateCacheMiddleware`` goes before all other middleware).

However, if you use cache decorators on individual views, the CSRF middleware
will not yet have been able to set the Vary header or the CSRF cookie, and the
response will be cached without either one. In this case, on any views that
will require a CSRF token to be inserted you should use the
:func:`django.views.decorators.csrf.csrf_protect` decorator first::

  from django.views.decorators.cache import cache_page
  from django.views.decorators.csrf import csrf_protect

  @cache_page(60 * 15)
  @csrf_protect
  def my_view(request):
      ...

If you are using class-based views, you can refer to Decorating
class-based views.

Testing
=======

The ``CsrfViewMiddleware`` will usually be a big hindrance to testing view
functions, due to the need for the CSRF token which must be sent with every POST
request.  For this reason, Django's HTTP client for tests has been modified to
set a flag on requests which relaxes the middleware and the ``csrf_protect``
decorator so that they no longer rejects requests.  In every other respect
(e.g. sending cookies etc.), they behave the same.

If, for some reason, you *want* the test client to perform CSRF
checks, you can create an instance of the test client that enforces
CSRF checks::

    >>> from django.test import Client
    >>> csrf_client = Client(enforce_csrf_checks=True)

.. _csrf-limitations:

Limitations
===========

Subdomains within a site will be able to set cookies on the client for the whole
domain.  By setting the cookie and using a corresponding token, subdomains will
be able to circumvent the CSRF protection.  The only way to avoid this is to
ensure that subdomains are controlled by trusted users (or, are at least unable
to set cookies).  Note that even without CSRF, there are other vulnerabilities,
such as session fixation, that make giving subdomains to untrusted parties a bad
idea, and these vulnerabilities cannot easily be fixed with current browsers.

Edge cases
==========

Certain views can have unusual requirements that mean they don't fit the normal
pattern envisaged here. A number of utilities can be useful in these
situations. The scenarios they might be needed in are described in the following
section.

Utilities
---------

The examples below assume you are using function-based views. If you
are working with class-based views, you can refer to Decorating
class-based views.

.. function:: csrf_exempt(view)

    This decorator marks a view as being exempt from the protection ensured by
    the middleware. Example::

        from django.views.decorators.csrf import csrf_exempt
        from django.http import HttpResponse

        @csrf_exempt
        def my_view(request):
            return HttpResponse('Hello world')

.. function:: requires_csrf_token(view)

    Normally the ``csrf_token`` template tag will not work if
    ``CsrfViewMiddleware.process_view`` or an equivalent like ``csrf_protect``
    has not run. The view decorator ``requires_csrf_token`` can be used to
    ensure the template tag does work. This decorator works similarly to
    ``csrf_protect``, but never rejects an incoming request.

    Example::

        from django.views.decorators.csrf import requires_csrf_token
        from django.shortcuts import render

        @requires_csrf_token
        def my_view(request):
            c = {}
            # ...
            return render(request, "a_template.html", c)

.. function:: ensure_csrf_cookie(view)

    This decorator forces a view to send the CSRF cookie.

Scenarios
---------

CSRF protection should be disabled for just a few views
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Most views requires CSRF protection, but a few do not.

Solution: rather than disabling the middleware and applying ``csrf_protect`` to
all the views that need it, enable the middleware and use
:func:`~django.views.decorators.csrf.csrf_exempt`.

CsrfViewMiddleware.process_view not used
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are cases when ``CsrfViewMiddleware.process_view`` may not have run
before your view is run - 404 and 500 handlers, for example - but you still
need the CSRF token in a form.

Solution: use :func:`~django.views.decorators.csrf.requires_csrf_token`

Unprotected view needs the CSRF token
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There may be some views that are unprotected and have been exempted by
``csrf_exempt``, but still need to include the CSRF token.

Solution: use :func:`~django.views.decorators.csrf.csrf_exempt` followed by
:func:`~django.views.decorators.csrf.requires_csrf_token`. (i.e. ``requires_csrf_token``
should be the innermost decorator).

View needs protection for one path
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A view needs CSRF protection under one set of conditions only, and mustn't have
it for the rest of the time.

Solution: use :func:`~django.views.decorators.csrf.csrf_exempt` for the whole
view function, and :func:`~django.views.decorators.csrf.csrf_protect` for the
path within it that needs protection. Example::

    from django.views.decorators.csrf import csrf_exempt, csrf_protect

    @csrf_exempt
    def my_view(request):

        @csrf_protect
        def protected_path(request):
            do_something()

        if some_condition():
           return protected_path(request)
        else:
           do_something_else()

Page uses AJAX without any HTML form
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A page makes a POST request via AJAX, and the page does not have an HTML form
with a ``csrf_token`` that would cause the required CSRF cookie to be sent.

Solution: use :func:`~django.views.decorators.csrf.ensure_csrf_cookie` on the
view that sends the page.

Contrib and reusable apps
=========================

Because it is possible for the developer to turn off the ``CsrfViewMiddleware``,
all relevant views in contrib apps use the ``csrf_protect`` decorator to ensure
the security of these applications against CSRF.  It is recommended that the
developers of other reusable apps that want the same guarantees also use the
``csrf_protect`` decorator on their views.

Settings
========

A number of settings can be used to control Django's CSRF behavior:

* ``CSRF_COOKIE_AGE``
* ``CSRF_COOKIE_DOMAIN``
* ``CSRF_COOKIE_HTTPONLY``
* ``CSRF_COOKIE_NAME``
* ``CSRF_COOKIE_PATH``
* ``CSRF_COOKIE_SECURE``
* ``CSRF_FAILURE_VIEW``

Cryptographic signing
=====================

.. module:: django.core.signing
   :synopsis: Django's signing framework.

The golden rule of Web application security is to never trust data from
untrusted sources. Sometimes it can be useful to pass data through an
untrusted medium. Cryptographically signed values can be passed through an
untrusted channel safe in the knowledge that any tampering will be detected.

Django provides both a low-level API for signing values and a high-level API
for setting and reading signed cookies, one of the most common uses of
signing in Web applications.

You may also find signing useful for the following:

* Generating "recover my account" URLs for sending to users who have
  lost their password.

* Ensuring data stored in hidden form fields has not been tampered with.

* Generating one-time secret URLs for allowing temporary access to a
  protected resource, for example a downloadable file that a user has
  paid for.

Protecting the SECRET_KEY
=========================

When you create a new Django project using ``startproject``, the
``settings.py`` file is generated automatically and gets a random
``SECRET_KEY`` value. This value is the key to securing signed
data -- it is vital you keep this secure, or attackers could use it to
generate their own signed values.

Using the low-level API
=======================

Django's signing methods live in the ``django.core.signing`` module.
To sign a value, first instantiate a ``Signer`` instance::

    >>> from django.core.signing import Signer
    >>> signer = Signer()
    >>> value = signer.sign('My string')
    >>> value
    'My string:GdMGD6HNQ_qdgxYP8yBZAdAIV1w'

The signature is appended to the end of the string, following the colon.
You can retrieve the original value using the ``unsign`` method::

    >>> original = signer.unsign(value)
    >>> original
    'My string'

If the signature or value have been altered in any way, a
``django.core.signing.BadSignature`` exception will be raised::

    >>> from django.core import signing
    >>> value += 'm'
    >>> try:
    ...    original = signer.unsign(value)
    ... except signing.BadSignature:
    ...    print("Tampering detected!")

By default, the ``Signer`` class uses the ``SECRET_KEY`` setting to
generate signatures. You can use a different secret by passing it to the
``Signer`` constructor::

    >>> signer = Signer('my-other-secret')
    >>> value = signer.sign('My string')
    >>> value
    'My string:EkfQJafvGyiofrdGnuthdxImIJw'

.. class:: Signer(key=None, sep=':', salt=None)

    Returns a signer which uses ``key`` to generate signatures and ``sep`` to
    separate values. ``sep`` cannot be in the `URL safe base64 alphabet
    <http://tools.ietf.org/html/rfc4648#section-5>`_.  This alphabet contains
    alphanumeric characters, hyphens, and underscores.

Using the salt argument
-----------------------

If you do not wish for every occurrence of a particular string to have the same
signature hash, you can use the optional ``salt`` argument to the ``Signer``
class. Using a salt will seed the signing hash function with both the salt and
your ``SECRET_KEY``::

    >>> signer = Signer()
    >>> signer.sign('My string')
    'My string:GdMGD6HNQ_qdgxYP8yBZAdAIV1w'
    >>> signer = Signer(salt='extra')
    >>> signer.sign('My string')
    'My string:Ee7vGi-ING6n02gkcJ-QLHg6vFw'
    >>> signer.unsign('My string:Ee7vGi-ING6n02gkcJ-QLHg6vFw')
    'My string'

Using salt in this way puts the different signatures into different
namespaces.  A signature that comes from one namespace (a particular salt
value) cannot be used to validate the same plaintext string in a different
namespace that is using a different salt setting. The result is to prevent an
attacker from using a signed string generated in one place in the code as input
to another piece of code that is generating (and verifying) signatures using a
different salt.

Unlike your ``SECRET_KEY``, your salt argument does not need to stay
secret.

Verifying timestamped values
----------------------------

``TimestampSigner`` is a subclass of :class:`~Signer` that appends a signed
timestamp to the value. This allows you to confirm that a signed value was
created within a specified period of time::

    >>> from datetime import timedelta
    >>> from django.core.signing import TimestampSigner
    >>> signer = TimestampSigner()
    >>> value = signer.sign('hello')
    >>> value
    'hello:1NMg5H:oPVuCqlJWmChm1rA2lyTUtelC-c'
    >>> signer.unsign(value)
    'hello'
    >>> signer.unsign(value, max_age=10)
    ...
    SignatureExpired: Signature age 15.5289158821 > 10 seconds
    >>> signer.unsign(value, max_age=20)
    'hello'
    >>> signer.unsign(value, max_age=timedelta(seconds=20))
    'hello'

.. class:: TimestampSigner(key=None, sep=':', salt=None)

    .. method:: sign(value)

        Sign ``value`` and append current timestamp to it.

    .. method:: unsign(value, max_age=None)

        Checks if ``value`` was signed less than ``max_age`` seconds ago,
        otherwise raises ``SignatureExpired``. The ``max_age`` parameter can
        accept an integer or a :py:class:`datetime.timedelta` object.

Protecting complex data structures
----------------------------------

If you wish to protect a list, tuple or dictionary you can do so using the
signing module's ``dumps`` and ``loads`` functions. These imitate Python's
pickle module, but use JSON serialization under the hood. JSON ensures that
even if your ``SECRET_KEY`` is stolen an attacker will not be able
to execute arbitrary commands by exploiting the pickle format::

    >>> from django.core import signing
    >>> value = signing.dumps({"foo": "bar"})
    >>> value
    'eyJmb28iOiJiYXIifQ:1NMg1b:zGcDE4-TCkaeGzLeW9UQwZesciI'
    >>> signing.loads(value)
    {'foo': 'bar'}

Because of the nature of JSON (there is no native distinction between lists
and tuples) if you pass in a tuple, you will get a list from
``signing.loads(object)``::

    >>> from django.core import signing
    >>> value = signing.dumps(('a','b','c'))
    >>> signing.loads(value)
    ['a', 'b', 'c']

.. function:: dumps(obj, key=None, salt='django.core.signing', compress=False)

    Returns URL-safe, sha1 signed base64 compressed JSON string. Serialized
    object is signed using :class:`~TimestampSigner`.

.. function:: loads(string, key=None, salt='django.core.signing', max_age=None)

    Reverse of ``dumps()``, raises ``BadSignature`` if signature fails.
    Checks ``max_age`` (in seconds) if given.

Security middleware
-------------------

.. module:: django.middleware.security
    :synopsis: Security middleware.

.. warning::
    If your deployment situation allows, it's usually a good idea to have your
    front-end Web server perform the functionality provided by the
    ``SecurityMiddleware``. That way, if there are requests that aren't served
    by Django (such as static media or user-uploaded files), they will have
    the same protections as requests to your Django application.

The ``django.middleware.security.SecurityMiddleware`` provides several security
enhancements to the request/response cycle. Each one can be independently
enabled or disabled with a setting.

* ``SECURE_BROWSER_XSS_FILTER``
* ``SECURE_CONTENT_TYPE_NOSNIFF``
* ``SECURE_HSTS_INCLUDE_SUBDOMAINS``
* ``SECURE_HSTS_SECONDS``
* ``SECURE_REDIRECT_EXEMPT``
* ``SECURE_SSL_HOST``
* ``SECURE_SSL_REDIRECT``

HTTP Strict Transport Security
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For sites that should only be accessed over HTTPS, you can instruct modern
browsers to refuse to connect to your domain name via an insecure connection
(for a given period of time) by setting the `"Strict-Transport-Security"
header`_. This reduces your exposure to some SSL-stripping man-in-the-middle
(MITM) attacks.

``SecurityMiddleware`` will set this header for you on all HTTPS responses if
you set the ``SECURE_HSTS_SECONDS`` setting to a non-zero integer value.

When enabling HSTS, it's a good idea to first use a small value for testing,
for example, ``SECURE_HSTS_SECONDS = 3600<SECURE_HSTS_SECONDS>`` for one
hour. Each time a Web browser sees the HSTS header from your site, it will
refuse to communicate non-securely (using HTTP) with your domain for the given
period of time. Once you confirm that all assets are served securely on your
site (i.e. HSTS didn't break anything), it's a good idea to increase this value
so that infrequent visitors will be protected (31536000 seconds, i.e. 1 year,
is common).

Additionally, if you set the ``SECURE_HSTS_INCLUDE_SUBDOMAINS`` setting
to ``True``, ``SecurityMiddleware`` will add the ``includeSubDomains`` tag to
the ``Strict-Transport-Security`` header. This is recommended (assuming all
subdomains are served exclusively using HTTPS), otherwise your site may still
be vulnerable via an insecure connection to a subdomain.

.. warning::
    The HSTS policy applies to your entire domain, not just the URL of the
    response that you set the header on. Therefore, you should only use it if
    your entire domain is served via HTTPS only.

    Browsers properly respecting the HSTS header will refuse to allow users to
    bypass warnings and connect to a site with an expired, self-signed, or
    otherwise invalid SSL certificate. If you use HSTS, make sure your
    certificates are in good shape and stay that way!

.. note::
    If you are deployed behind a load-balancer or reverse-proxy server, and the
    ``Strict-Transport-Security`` header is not being added to your responses,
    it may be because Django doesn't realize that it's on a secure connection;
    you may need to set the ``SECURE_PROXY_SSL_HEADER`` setting.

.. _"Strict-Transport-Security" header: http://en.wikipedia.org/wiki/Strict_Transport_Security

``X-Content-Type-Options: nosniff``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some browsers will try to guess the content types of the assets that they
fetch, overriding the ``Content-Type`` header. While this can help display
sites with improperly configured servers, it can also pose a security
risk.

If your site serves user-uploaded files, a malicious user could upload a
specially-crafted file that would be interpreted as HTML or Javascript by
the browser when you expected it to be something harmless.

To learn more about this header and how the browser treats it, you can
read about it on the `IE Security Blog`_.

To prevent the browser from guessing the content type and force it to
always use the type provided in the ``Content-Type`` header, you can pass
the ``X-Content-Type-Options: nosniff`` header.  ``SecurityMiddleware`` will
do this for all responses if the ``SECURE_CONTENT_TYPE_NOSNIFF`` setting
is ``True``.

Note that in most deployment situations where Django isn't involved in serving
user-uploaded files, this setting won't help you. For example, if your
``MEDIA_URL`` is served directly by your front-end Web server (nginx,
Apache, etc.) then you'd want to set this header there. On the other hand, if
you are using Django to do something like require authorization in order to
download files and you cannot set the header using your Web server, this
setting will be useful.

.. _IE Security Blog: http://blogs.msdn.com/b/ie/archive/2008/09/02/ie8-security-part-vi-beta-2-update.aspx

``X-XSS-Protection: 1; mode=block``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some browsers have the ability to block content that appears to be an `XSS
attack`_. They work by looking for Javascript content in the GET or POST
parameters of a page. If the Javascript is replayed in the server's response,
the page is blocked from rendering and an error page is shown instead.

The `X-XSS-Protection header`_ is used to control the operation of the
XSS filter.

To enable the XSS filter in the browser, and force it to always block
suspected XSS attacks, you can pass the ``X-XSS-Protection: 1; mode=block``
header. ``SecurityMiddleware`` will do this for all responses if the
``SECURE_BROWSER_XSS_FILTER`` setting is ``True``.

.. warning::
    The browser XSS filter is a useful defense measure, but must not be
    relied upon exclusively. It cannot detect all XSS attacks and not all
    browsers support the header. Ensure you are still validating and
    sanitizing all input to prevent XSS attacks.

.. _XSS attack: http://en.wikipedia.org/wiki/Cross-site_scripting
.. _X-XSS-Protection header: http://blogs.msdn.com/b/ie/archive/2008/07/02/ie8-security-part-iv-the-xss-filter.aspx

SSL Redirect
~~~~~~~~~~~~

If your site offers both HTTP and HTTPS connections, most users will end up
with an unsecured connection by default. For best security, you should redirect
all HTTP connections to HTTPS.

If you set the ``SECURE_SSL_REDIRECT`` setting to True,
``SecurityMiddleware`` will permanently (HTTP 301) redirect all HTTP
connections to HTTPS.

.. note::

    For performance reasons, it's preferable to do these redirects outside of
    Django, in a front-end load balancer or reverse-proxy server such as
    `nginx`_. ``SECURE_SSL_REDIRECT`` is intended for the deployment
    situations where this isn't an option.

If the ``SECURE_SSL_HOST`` setting has a value, all redirects will be
sent to that host instead of the originally-requested host.

If there are a few pages on your site that should be available over HTTP, and
not redirected to HTTPS, you can list regular expressions to match those URLs
in the ``SECURE_REDIRECT_EXEMPT`` setting.

.. note::
    If you are deployed behind a load-balancer or reverse-proxy server and
    Django can't seem to tell when a request actually is already secure, you
    may need to set the ``SECURE_PROXY_SSL_HEADER`` setting.

.. _nginx: http://nginx.org

