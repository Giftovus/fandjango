# Fandjango

## About

Facebook applications are simply websites that load in iframes on Facebook. Facebook provide documents loaded
within these iframes with various data, such as information about the user accessing it or the Facebook Page
it is accessed from. This data is encapsulated in [signed requests](http://developers.facebook.com/docs/authentication/signed_request/).
Fandjango parses signed requests, abstracts the information contained within and populates the request object accordingly.

## Getting started

You may find a sample application and a walkthrough to replicate it at the [Fandjango Example](https://github.com/jgorset/fandjango-example) repository.

## Usage

### Users

Fandjango saves clients that have authorized your application in its `User` model. You may access
the corresponding model instance in `request.facebook.user`.

Instances of the `User` model have the following properties:

* `facebook_id` - An integer describing the user's Facebook ID.
* `first_name` - A string describing the user's first name.
* `last_name` - A string describing the user's last name.
* `profile_url` - A string describing the URL to the user's Facebook profile.
* `gender` - A string describing the user's gender.
* `oauth_token` - An OAuth Token object.
* `created_at` - A datetime object describing when the user was registered.
* `last_seen_at` - A datetime object describing when the user was last seen.

`oauth_token` is an instance of the `OAuthToken` model, which has the following properties:

* `token` - A string describing the OAuth token itself.
* `issued_at` - A datetime object describing when the token was issued.
* `expires_at` - A datetime object describing when the token expires (or `None` if it doesn't)

If the client has not authorized your application, `request.facebook.user` is `None`.

#### Authorizing users

You may require a client to authorize your application before accessing a view with the
`facebook_authorization_required` decorator.

    from fandjango.decorators import facebook_authorization_required
    
    @facebook_authorization_required()
    def foo(request, *args, **kwargs):
        pass
      
This will redirect the request to the Facebook authorization dialog, which will in
turn redirect back to the original URI. The decorator accepts an optional argument `redirect_uri`,
allowing you to customize the location the user is redirected to after authorizing the application:

    from settings import FACEBOOK_APPLICATION_TAB_URL
    from fandjango.decorators import facebook_authorization_required
    
    @facebook_authorization_required(redirect_uri=FACEBOOK_APPLICATION_TAB_URL)
    def foo(request, *args, **kwargs):
        pass

If you prefer, you may redirect the request in a control flow of your own by using the
`redirect_to_facebook_authorization` function:

    from fandjango.utils import redirect_to_facebook_authorization
    
    def foo(request, *args, **kwargs):
        if not request.facebook.user:
            return redirect_to_facebook_authorization(redirect_uri='http://www.example.org/')

### Pages

If the application is accessed from a tab on a Facebook Page, you'll find an instance of `FacebookPage`
in `request.facebook.page`.

Instances of the `FacebookPage` model have the following properties:

* `id` -- An integer describing the id of the page.
* `is_admin` -- A boolean describing whether or not the current user is an administrator of the page.
* `is_liked` -- A boolean describing whether or not the current user likes the page.
* `url` -- A string describing the URL to the page.

If the application is not accessed from a tab on a Facebook Page, `request.facebook.page` is `None`.
        
## Installation

* `pip install fandjango`
* Add `fandjango` to `INSTALLED_APPS`
* Add `fandjango.middleware.FacebookMiddleware` to `MIDDLEWARE_CLASSES`

*Note:* If you're using Django's built-in CSRF protection middleware, you need to make sure Fandjango's middleware precedes it.
Otherwise, Facebook's requests to your application will qualify cross-site request forgeries.

## Upgrading to 3.5

Fandjango 3.5 introduces several new fields to its `User` model. Thankfully, migrations for [South](http://south.aeracode.org/)
have been bundled with Django since 3.4.1 and upgrading your database is as simple as running `./manage.py migrate fandjango`.

If you're not using South, start using South. If you really don't want to, though, you can upgrade your database manually
by adding the following fields to the `fandjango_user` table:

    Field name                  Field type
    
    facebook_username           varchar (255)
    hometown                    varchar (255)
    location                    varchar (255)
    bio                         text
    relationship_status         varchar (255)
    political_views             varchar (255)
    email                       varchar (255)
    website                     varchar (255)
    locale                      varchar (255)
    verified                    tinyint (1)
    birthday                    datetime
    

## Configuration

Fandjango requires some constants to be set in your settings.py file:

* `FACEBOOK_APPLICATION_ID` - Your Facebook application's ID.
* `FACEBOOK_APPLICATION_SECRET_KEY` - Your Facebook application's secret key.
* `FACEBOOK_APPLICATION_URL` - Your application's canvas URI (ex. http://apps.facebook.com/my_application)
* `FACEBOOK_APPLICATION_INITIAL_PERMISSIONS` - A list of [extended permissions][2] to request upon authorizing the application (optional).
* `FANDJANGO_DISABLED_PATHS` - A list of regular expression patterns describing paths on which Fandjango should not act (optional). These
should typically be paths that are accessed outside of the Facebook Canvas, such as Django's admin site.
* `FANDJANGO_ENABLED_PATHS` - A list of regular expression patterns describing paths on which Fandjango should act (optional). If undefined,
Fandjango will operate on all paths.

[2]: http://developers.facebook.com/docs/authentication/permissions

## Frequently asked questions

**Q:** *Do I need to pass the signed request around?*

**A:** No. Fandjango caches the latest signed request in a cookie so you don't have to worry about it.

**Q:** *Why does Django raise a CSRF exception when my application loads in the Facebook canvas?*

**A:** As of March 2011, Facebook's initial request to your application is a HTTP POST request that evaluates
to an attempt at cross-site request forgery by Django's built-in CSRF protection. Fandjango remedies this by
overriding the request method of POST requests that only contain a signed request to GET, but you need to make
sure that its middleware is loaded before Django's `CsrfViewMiddleware`.

**Q:** *Why does Fandjango set a new header called "P3P"?*

**A:** P3P (or *Platform for Privacy Preferences*) is a W3 standard that enables websites to express
their privacy practices in a standard format that can be retrieved automatically and interpreted easily
by user agents. While this is largely ignored by most browsers, Internet Explorer will ignore cookies
set by third-party websites (ie. websites loaded in iframes) unless it specifies some P3P policies.

You can read more about P3P at [w3.org][3].

[3]: http://www.w3.org/TR/P3P/

**Q:** *What happens when the OAuth token expires?*

**A:** Fandjango will automatically renew the signed request once the OAuth token
expires. It does this by hijacking the request and redirecting the client to Facebook, which
in turn redirects the client back to the URI it was originally retrieving with a new signed
request attached.
