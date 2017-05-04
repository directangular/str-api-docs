---
title: API Reference

language_tabs:
  - python

toc_footers:
  - <a href='https://www.popitup.com/developers'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

search: true
---

# Introduction

Welcome to the ShopTheRoe API! You can use our API to access ShopTheRoe API
endpoints, allowing you to build on top of our massive Direct Sales
ecosystem.

You can browse the API [here](https://shoptheroe.com/api/v2/) and view the
full API schema [here](https://shoptheroe.com/api/v2/schema/).

We have example projects set up for a few popular platforms that you can
use for reference:

Platform | Code
---------|-----
Server-side web app | [GitHub](https://github.com/directangular/str-api-demo-web)
Android | [GitHub](https://github.com/directangular/str-api-demo-android)
iOS | TODO

We're working on creating proper client libraries for popular languages,
but in the meantime you can use any off-the-shelf HTTP request library to
get everything done that you need.  In this guide we're using `python` and
`requests`.

# Important URLs

The `API` and `OAUTH` base urls for the staging and production sites are:

### Staging (`https://beta.shoptheroe.com`)
- `OAUTH_BASE` :: `https://beta.shoptheroe.com/o/`
- `API_BASE` :: `https://beta.shoptheroe.com/api/v2/`

### Production (`https://shoptheroe.com`)
- `OAUTH_BASE` :: `https://shoptheroe.com/o/`
- `API_BASE` :: `https://shoptheroe.com/api/v2/`

<aside class="notice">
Note: the trailing <code>/</code> <em>is important</em>.  The STR application
servers redirect all requests that don't end in a <code>/</code> to the
same URL with a <code>/</code> appended.  However, most oauth2 client
libraries don't handle that redirect correctly, so the end result is that
your oauth client will be broken if you forget the trailing <code>/</code>.
</aside>

# Authentication

## Get an Access Token

ShopTheRoe uses API keys and oauth2 to grant access to the API. You can
register a new ShopTheRoe API key at
our [developer portal](https://www.popitup.com/developers).  You might also
want to hop on [our Slack team](https://directangular-slack.herokuapp.com)
to facilitate collaboration.

Once you have credentials for your app, you can use our oauth2 endpoints to
get a personal access token for the user.  The way you do that depends on
what kind of app you're developing.

Application type | Grant type
-----------------|-----------
Mobile app (iOS, Android, etc.) | Implicit
Web app (server-side) | Authorization code

### Implicit Grant (Mobile apps (iOS, Android, etc))

Since the code in a mobile app is distributed to the user, you can't
include your secret key in your code, since that would compromise your
secret key and could result in abuse and blacklisting.  Instead, you rely
on the resource owner (the user) to grant access.  The oauth2 specification
refers to this as an "implicit" grant type, which you can read more
about [here](https://tools.ietf.org/html/rfc6749#section-4.2).

<aside class="notice">
Since this method doesn't include the secret key, it is a bit less secure
and therefore doesn't support all of the features of other grant types.
For example, refresh tokens are not supported when using the implicit grant
type.  But to be clear, the implicit grant type is your <em>only</em>
option when developing a mobile app.
</aside>

In order to request a personal access token for a user, just send them to
the following URL:

`https://shoptheroe.com/o/authorize/?response_type=token&client_id=$CLIENT_ID&state=random_state_string&redirect_uri=$REDIRECT_URI`

Make sure to replace `$CLIENT_ID` and `$REDIRECT_URI` with your client ID
and desired redirect URI, respectively.

Once the user authenticates and grants access to your application they'll
be redirected to `$REDIRECT_URI` with an `access_token` parameter in the
query fragment.  `$REDIRECT_URI` should actually be a custom schema
(e.g. `my-cool-app://str-cb`) that you've registered with your system.
See
[the Android STR API demo](https://github.com/directangular/str-api-demo-android) for
an example of how to register a custom schema handler and how to parse the
`access_token` out of the URL fragment.

### Authorization Code Grant (Server-side Web Apps)

> This is the redirect URI handler.  It will extract the code, exchange it
> for a set of tokens, save the tokens, and redirect the user to your app
> where the access token can be used to make authorized requests to the STR
> API.

```python
OAUTH_BASE = 'https://shoptheroe.com/o/'
TOKEN_URL = OAUTH_BASE + 'token/'

# this is the redirect URI callback handler
@route('/cb')
def cb():
    if request.query.get('error'):
        # ...more error logging/handling...
        redirect('/')
        return

    # exchange the auth code for an access token
    data = {
        "grant_type": "authorization_code",
        "code": request.query['code'],
        "redirect_uri": REDIRECT_URI,
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
    }
    res = requests.post(TOKEN_URL, data=data)
    if res.status_code != 200:
        raise AuthenticationFailed(res)
    resp = json.loads(res.content)

    user = get_current_user()
    user.save_tokens(resp['access_token'], resp['refresh_token'])
    redirect('/app')
```

For server-side apps, you'll use the secret access key to get a personal
access token for the user.  The oauth2 spec refers to this as an
"Authorization code" grant type, which you can read more
about [here](https://tools.ietf.org/html/rfc6749#section-4.1).  This method
offers a bit more security and features (like refresh tokens).

As with all secret keys, you should *never* check your key in to your
source code repo.  Instead, you can deploy it with your app using
environment variables or similar.

The flow for the authorization grant type is a two-step process:

1. Send an authorization request (server responds with a `code`)
1. Send a token request using the `code` from step 1 (server responds with
   an `access_token`)

To get the process started, just send the user to the following URL:

`https://shoptheroe.com/o/authorize/?response_type=code&client_id=$CLIENT_ID&state=random_state_string&redirect_uri=$REDIRECT_URI`

Make sure to replace `$CLIENT_ID` and `$REDIRECT_URI` with your client ID
and desired redirect URI, respectively.  The `REDIRECT_URI` will most
likely be a custom endpoint in your app to handle oauth callbacks, for
example: `https://my-cool-app.example.com/str-cb`.

Once the user authenticates and grants access to your application they'll
be redirected to `$REDIRECT_URI` with a `code` parameter in the
querystring.  You can then use the returned `code` to get access/refresh
tokens by posting to `$OAUTH_BASE/token/` (see code example at the right).

#### Refresh Tokens

> The `access_token` can be used right away to make authorized API
> requests. The `refresh_token`
> ([oauth2 spec](https://tools.ietf.org/html/rfc6749#section-1.5)) can be
> used to get a new access token in the future:

```python
# Swapping out a refresh token for a new access token
data = {
    'client_id': CLIENT_ID,
    'client_secret': CLIENT_SECRET,
    'grant_type': 'refresh_token',
    'refresh_token': REFRESH_TOKEN
}
res = requests.post(oauth_url, data=data)
if res.status_code != 200:
    raise AuthenticationError(res)
resp = json.loads(res.content)
user = get_current_user()
# save the new access/refresh tokens
user.save_tokens(resp['access_token'], resp[refresh_token'])
```

> Generally you'll only do the refresh if a regular request returns an
> Unauthorized error (status code 401).  So you might use a wrapper like
> this:

```python
def make_api_request(user, endpoint):
    """Simple wrapper that does a refresh if needed"""
    resp = _make_api_request(user.access_token, endpoint)
    if resp.status_code == 401:
        user.refresh_tokens()
        resp = _make_api_request(user.access_token, endpoint)
    return resp
```

All personal access tokens have a 14-day expiration on them.  In order to
avoid making the user re-authorize your app by clicking through the oauth
permissions dialog again, you can use the `refresh_token` that gets
returned alongside your `access_token` to get a new personal access token.

Most of the time you'll wait to refresh a token until you get an "invalid
token error" with the current access token.
See [this section](https://tools.ietf.org/html/rfc6749#section-1.5) of the
oauth2 spec for an example sequence diagram.  There's also an example on
the right showing how you might do it.


## Use the Access Token

> To make an authorized request, just add an "Authorization" header with a
> personal access token:

```python
headers = {"Authorization": "Bearer " + access_token}
requests.get(endpoint_url, headers=headers)
```

Once you have personal access token for a user, you can make API requests
on their behalf.  ShopTheRoe expects the API key to be included in all API
requests to the server in an HTTP header that looks like the following:

`Authorization: Bearer <token>`

<aside class="notice">
You must replace <code>&lt;token&gt;</code> with the personal access token.
</aside>

# API Schema

The remainder of this document will provide details on available endpoints
and their usage.  However, the definitive reference is the auto-generated
API schema, [here](https://shoptheroe.com/api/v2/schema/).

# Items

## Get All Items

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
res = requests.get(api_url + 'items/', headers)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data['results'][0]['image']['image_thumbnail']
```

This endpoint retrieves all items.  You can see what the response looks
like in [the API browser](https://shoptheroe.com/api/v2/items/).  The
response is paged.

## Get a Specific Item

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
res = requests.get(api_url + 'items/<ITEM_ID>/', headers)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data['image']['image_thumbnail']
```

This endpoint retrieves a specific item.

## Create an Item

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
data = {
    'itemchoice_id': itemchoice_id,
    'size_id': size_id,
    'image_id': image_id,
    'data': json.dumps({  # optional!
        'attributes': {
            'Color': 'Red',
            'Collection': 'Elegant',
        }
    }),
}
res = requests.post(api_url + 'items/', headers=headers, data=data)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data['image_thumbnail'], data['pk']
```

Create an item object by `POST`ing to `$API_BASE/items/`.

### Required:

  - `itemchoice_id`
  - `size_id`
  - `image_id`

### Optional:

  - `price`
    - If not provided, minimum advertised price for the style (itemchoice)
      will be used
  - `data`
    - JSON object of the form `{'attributes': {'Key': 'Value', ...}}`

<aside class="notice">
Note: the wholesale price will be automatically populated based on the
wholesale price for the <code>itemchoice</code>.
</aside>


# Images

## Get All Images

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
res = requests.get(api_url + 'images/', headers)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data['results'][0]['image_thumbnail']
```

This endpoint retrieves all images.  You can see what the response looks
like in [the API browser](https://shoptheroe.com/api/v2/images/).  The
response is paged.

## Get a Specific Image

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
res = requests.get(api_url + 'images/<IMAGE_ID>/', headers)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data['image_thumbnail']
```

This endpoint retrieves a specific image.

## Create an Image

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
with open('/tmp/my_image.png') as imgfile:
    files = {'image_full': imgfile}
    res = requests.post(api_url + 'images/', headers=headers, files=files)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data['image_thumbnail'], data['pk']
```

You can create an image object by `POST`ing to `$API_BASE/images/`,
providing a file object in the `image_full` parameter.  Images aren't
terribly useful on their own.  Generally you should also create an `Item`
to go along with the `Image`.

# Item Choices (a.k.a. "styles")

## Get All Item Choices

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
res = requests.get(api_url + 'itemchoices/', headers)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data['results'][0]
```

This endpoint retrieves all item choices.  You can see what the response looks
like in [the API browser](https://shoptheroe.com/api/v2/itemchoices/).  The
response is paged.

## Get a Specific Item Choice

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
res = requests.get(api_url + 'itemchoices/<ITEMCHOICE_ID>/', headers)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data
```

This endpoint retrieves a specific item choice.

# Sizes

## Get All Sizes

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
res = requests.get(api_url + 'sizes/', headers)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data['results'][0]
```

This endpoint retrieves all sizes.  You can see what the response looks
like in [the API browser](https://shoptheroe.com/api/v2/sizes/).  The
response is paged.

## Get a Specific Size

```python
import requests

api_url = 'https://shoptheroe.com/api/v2/'
headers = {"Authorization": "Bearer " + access_token}
res = requests.get(api_url + 'sizes/<SIZE_ID>/', headers)
if res.status_code != 200:
    raise AuthenticationError(res)
data = json.loads(res.content)
print data
```

This endpoint retrieves a specific size.
