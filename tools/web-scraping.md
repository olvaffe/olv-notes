Web Scraping
============

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
