Web Scraping
============

## URL

- <https://datatracker.ietf.org/doc/html/rfc3986>
- `<scheme>://<authority>/<path>?<query>#<fragment>`
  - `<authority>` is `<userinfo>@<host>:<port>`
  - `<path>` can be suffixed by `;<param>`

## HTTP Request

- a request line
  - e.g., `GET /images/logo.png HTTP/1.1`
  - `GET`
  - `HEAD`
  - `POST`
  - `PUT`
  - `DELETE`
  - `CONNECT`
  - `OPTIONS`
  - `TRACE`
  - `PATCH`
- one or more request header fields
  - e.g., `Host: www.example.com`
  - standard fields
    - `A-IM` acceptable instance-manipulations
    - `Accept` acceptable media type, such as `text/html`
    - `Accept-Charset` acceptable character sets, such as `utf-8`
    - `Accept-Encoding` acceiptable encodings, such as `gzip`
    - `Accept-Language` acceptable languages, such as `en-US`
    - `Access-Control-Request-Method` and `Access-Control-Request-Headers`
    - `Authorization` credential for http auth
    - `Cache-Control`
    - `Connection` connection options, such as `keep-alive`
    - `Content-Encoding` data encoding, such as `gzip`
    - `Content-Length` length of the request body in octets (8-bit bytes)
    - `Content-Type` media type of request body, such as `application/x-www-form-urlencoded`
    - `Cookie` cookie
    - `Date` date and time of the request generation
    - `Expect`
    - `Forwarded` inserted by proxy
    - `From` email, such as `user@example.com`
    - `Host` the domain name (for virtual hosting) and port of the dst
    - `If-Match`
    - `If-Modified-Since`
    - `If-None-Match`
    - `If-Range`
    - `If-Unmodified-Since`
    - `Max-Forwards` max number of proxy/gateway forwards
    - `Origin`
    - `Pragma` implementation-specific
    - `Prefer`
    - `Proxy-Authorization` credential for proxy
    - `Range` a range of the entity, such as `bytes=500-999`
    - `Referer` the addr of prior web page
    - `TE` transfer encodings
    - `Trailer`
    - `Transfer-Encoding`
    - `User-Agent` the user agent, such as `User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:12.0) Gecko/20100101 Firefox/12.0`
    - `Upgrade` ask the server to upgrade to another protocol
    - `Via` list of proxies
- an empty line
- optional request body

## HTTP Response

- a status line
  - e.g., `HTTP/1.1 200 OK`
  - `1XX` is informational
  - `2XX` is successful
  - `3XX` is redirection
  - `4XX` is client error
  - `5XX` is server error
- zero or more response header fields
  - standard fields
    - `Access-Control-*`
    - `Accept-Patch` patch document format
    - `Accept-Ranges` range types, such as `bytes`
    - `Age` object age in proxy in seconds
    - `Allow` valid methods, such as `GET, HEAD`
    - `Alt-Svc` alternative url
    - `Cache-Control` cache control, such as `max-age=3600`
    - `Connection` connection options
    - `Content-Disposition`, such as `attachment; filename="fname.ext"`
    - `Content-Encoding` data encoding, such as `gzip`
    - `Content-Language` data language, such as `en-US`
    - `Content-Length` length of the response body in octets (8-bit bytes)
    - `Content-Location` alternative location, such as `/index.htm`
    - `Content-Range`, such as `bytes 21010-47021/47022`
    - `Content-Type` mime type, such as `text/html; charset=utf-8`
    - `Date` date and time of the response
    - `Delta-Base`
    - `ETag`
    - `Expires` response expiration data and time
    - `IM` instance-manipulations applied
    - `Last-Modified` last modified date for the requested object
    - `Link` relation to another resource
    - `Location` redirection or resource creation
    - `P3P`
    - `Pragma` implementation-specific fields
    - `Preference-Applied` whether `Prefer` was honored
    - `Proxy-Authenticate` request proxy auth
    - `Public-Key-Pins`
    - `Retry-After` retry later in seconds or data/time
    - `Server` server name, such as `Apache/2.4.1 (Unix)`
    - `Set-Cookie` cookie
    - `Strict-Transport-Security` hsts policy
    - `Trailer`
    - `Transfer-Encoding`
    - `Tk` tracking Status in response to DNT (do-not-track)
    - `Upgrade` ask the client to upgrade to another protocol
    - `Vary`
    - `Via` list of proxies
    - `WWW-Authenticate` auth scheme, such as `Basic`
- an empty line
- optional response body

## Modules

- <https://pypi.org/project/requests/> for synchronous http requests
  - `requests.request` sends a request synchronously
    - `method` is `GET`, `POST`, etc.
    - `url` is the url
    - `params` is the query string (`?<query-string>` in the req)
    - `data` or `json` is the body
    - `headers` is the req headers
    - `cookies` is the req cookies
    - `files`
    - `auth` for http auth
    - `timeout`
    - `allow_redirects`
    - `proxies`
    - `verify`
    - `stream`
    - `cert`
  - `requests.get` sends a `GET` request
- <https://pypi.org/project/beautifulsoup4/> for html parsing
  - `bs4.BeautifulSoup` parses html
  - `soup.css.select` selects a tag using a css selector
