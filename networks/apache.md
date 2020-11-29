Apache
======

## Scope

- `<Directory>` limits the scope by directory
- `<Files>` limits the scope by filename
- `<Location>` limits the scope by URL
- There is a "Match" variant which implies regular expression by default
- The order (latter sections override earlier ones)
  1. `<Directory>` and .htaccess
  2. `<DirectoryMatch>`
  3. `<Files>` and `<FilesMatch>`
  4. `<Location>` and `<LocationMatch>`
- Except `<Directory>`, each group is processed in the order they appear in the
  configuration.  `<Directory>` is processed from the shortest to longest.

## Handler

- Files are handled differently depending on their types
- e.g. `AddHandler cgi-script .cgi` to handle `.cgi` by `cgi-script` handler

## Mapping

Mapping maps URLs to filesystem locations

- URL-Path (the part of the URL after host and port) is appended to `<DocumentRoot>`
- `Alias URL-path file-path|directory-path`.  Trailing `/` matters.
- When `AcceptPathInfo` is on (default for CGI), trailing path info of an
  existing file will be set in `PATH_INFO`.

## `mode_access`

- Allow from network/domain/host
- Order Allow,Deny
  1. Evaluate Allow; reject if none matches
  2. Evaluate Deny; reject if any matches
  3. Reject by default
- Order Deny,Allow
  1. Evaluate Deny; reject if any matches, unless it also matches an Allow
  2. Accept by default

## vhost

- <http://httpd.apache.org/docs/2.2/vhosts/>
- `Listen 8080` to make the server listen on port 8080
- `NameVirtualHost *:8080` to designate the IP address/port for virtual hosting
- Multiple `<VirtualHost *:8080>` blocks for each host
  - At least, `ServerName` and `DocumentRoot` must be given
  - `ServerAlias` to list other names
- The requests are handled this way

  > Now when a request arrives, the server will first check if it is using an IP
  > address that matches the NameVirtualHost. If it is, then it will look at each
  > `<VirtualHost>` section with a matching IP address and try to find one where the
  > ServerName or ServerAlias matches the requested hostname. If it finds one,
  > then it uses the configuration for that server. If no matching virtual host is
  > found, then the first listed virtual host that matches the IP address will be
  > used.

## CGI

- <http://hoohoo.ncsa.illinois.edu/cgi/env.html>
- env
  - `PATH_INFO`: The extra path information, as given by the client. In other
    words, scripts can be accessed by their virtual pathname, followed by extra
    information at the end of this path. The extra information is sent as
    `PATH_INFO`.  This information should be decoded by the server if it comes
    from a URL before it is passed to the CGI script.
  - `SCRIPT_NAME`: A virtual path to the script being executed, used for
    self-referencing URLs. 
  - `QUERY_STRING`: The information which follows the ? in the URL which
    referenced this script.  This is the query information. It should not be
    decoded in any fashion. This variable should always be set when there is
    query information, regardless of command line decoding. 
- script should be able to reference itself with
  `http://$SERVER_NAME:$SERVER_PORT$SCRIPT_NAME$PATH_INFO`

## `mod_rewrite`

- 

## Request

- <http://httpd.apache.org/docs/2.2/developer/request.html>
- `ap_process_request_internal`
  1. Unescapes the URL
  2. Strips Parent and This Elements from the URI
  3. Initial URI Location Walk
  4. translate_name: determine the filename or alter the URL path.
  5. Hook: map_to_storage: `ap_directory_walk` and `ap_file_walk`
  6. URI Location Walk
  7. Hook: header_parser
  8. security checking
  9. Hook: type_checker: determine content type
  10. Hook: fixups
- `ap_invoke_handler`
  1. Hook: insert_filter
  2. Hook: handler
- Hooks

        ap_hook_post_config
            this is where the old _init routines get registered
        ap_hook_http_method
            retrieve the http method from a request. (legacy)
        ap_hook_open_logs
            open any specified logs
        ap_hook_auth_checker
            check if the resource requires authorization
        ap_hook_access_checker
            check for module-specific restrictions
        ap_hook_check_user_id
            check the user-id and password
        ap_hook_default_port
            retrieve the default port for the server
        ap_hook_pre_connection
            do any setup required just before processing, but after accepting
        ap_hook_process_connection
            run the correct protocol
        ap_hook_child_init
            call as soon as the child is started
        ap_hook_create_request
            ??
        ap_hook_fixups
            last chance to modify things before generating content
        ap_hook_handler
            generate the content
        ap_hook_header_parser
            lets modules look at the headers, not used by most modules, because they use post_read_request for this
        ap_hook_insert_filter
            to insert filters into the filter chain
        ap_hook_log_transaction
            log information about the request
        ap_hook_optional_fn_retrieve
            retrieve any functions registered as optional
        ap_hook_post_read_request
            called after reading the request, before any other phase
        ap_hook_quick_handler
            called before any request processing, used by cache modules.
        ap_hook_translate_name
            translate the URI into a filename
        ap_hook_type_checker
            determine and/or set the doc type 

