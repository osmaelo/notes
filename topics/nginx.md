# nginx

It only knows about static content. While it can serve static HTML, for dynamically generated HTML, it has to offload requests to other processes in order to generate HTML. Ex. User requests a PHP file, and nginx sends it to an interpreter, which then returns the HTML, which is sent back by nginx.

Nginx communicates with the PHP interpreter via a **unix socket** file. Nginx communicates with the website visitor via a **TCP socket** file.  

To access a guest (VM) nginx from a host, set a port forwarding in Virtualbox with name `nginx`, host port `80`, guest port `80`.

## Install
`add-apt-repository ppa:nginx/stable` - Add 3rd party repo.  
`apt-get update` - Refresh packages list.  
`apt-get install nginx` - Install the web server.  

## Configuration

`/usr/share/nginx/html/` has the index.html file.  

`nginx -t` - After changes, check the syntax and run a test to make sure the web-server will work.  

`systemctl reload nginx` - Load the changes because the server is running with the old config file.    

`/etc/nginx/nginx.conf` is the main server configuration file. For each site we run, a separate file is needed, unless we want to run just one site.

Specific site configurations must be placed in the `/etc/nginx/conf.d/` folder. These override any `nginx.conf` settings. These are loaded by the `include /etc/nginx/conf.d/*.conf;` line.  

`http {}` block - For all incoming http requests, use these settings.

## Caching

Reusing page renders after they have been loaded once. Instead of everyone making a request, running a script, fetching data from the database, rendering a page and sending a response, only the first one does the work, the results is cached in a database, and everyone that wants the same is simply given the results from the database.

## Proxy vs Reverse Proxy

**Normal:** Client > Server  
**Proxy:** Client > Proxy > Server

Used to hide the origin of the request.  

**Normal:** Client > App  
**Reverse Proxy:** Client > Server > App

Used to prevent direct access to app i.e. exploit bad code.  

To run any Linux server on port 80, you need to be running as root. If you want to run node directly on port 80, that means you have to run node as root. If your application has any security flaws, it will become significantly compounded by the fact that it has access to do *anything it wants*. For example, if you write something that accidentally allows the user to read arbitrary files, now they can read files from other server user's accounts.  

If you use nginx in front of node, you can run your node as a limited user on port 3000 (or whatever) and then have nginx run as root on port 80, forwarding requests to port 3000. Now the node application is limited to the user permissions you give it.  

It's always better to use nginx as a proxy for a node.js server; nginx can proxy to a number of node backends and should any of them die can fail-over automatically with the benefit that, if there is an issue with the node interpreter (for instance while upgrading) and it stops responding, it can serve a fall-back HTML page.

Additionally, you shouldn't be using node.js for serving "static" files such as images, js/css files, etc. Use node.js for the complex stuff and let nginx take care of the things it's good at - serving files from the disk or from a cache.

## PHP Interpreter Setup

Nginx only understands static files. It doesn't know how to handle code. For that, we use an interpreter (sits behind the web-server), which communicates with nginx via a socket file.  

1. `apt-get install php-mysql php-fpm`. - Install the interpreter.
    - **FastCGI Process Manager** is used to create a pool of PHP interpreter processes waiting to deal with web-server requests. Without this, there would be only one process which would handle requests one by one, resulting in long wait times.  
2. `apt-get install php-json php-xmlrpc php-curl php-gd php-xml` - Install extra extensions needed for Wordpress.
3. `mkdir /var/run/php-fpm` - Create directory for php-fpm sockets (webserver -> PHP)
4. `ls /etc/php/7.0/fpm/pool.d` - Check if the pool configurations directory exists. If not, create it.
5. Configure `/etc/php/7.0/fpm/php-fpm.conf`:
    - Make a backup copy.
    - Add **just** the following inside:
        ```
        [global]
        pid = /run/php-fpm.pid
        error_log = /var/log/php-fpm.log
        include = /etc/php/fpm/pool.d/*.conf
        ```
6. Create a default pool configuration in `/etc/php/7.0/fpm/pool.d/www.conf`:
    - Make a backup copy.
    - Add **just** the following inside to define a private socket between nginx and the interpreter i.e. www-data from nginx communicates with www-data from fpm:
        ```
        [default]
        security.limit_extensions = .php
        listen = /var/run/php/yourserverhostname.sock
        listen.owner = www-data
        listen.group = www-data
        listen.mode = 0660
        user = www-data
        group = www-data
        pm = dynamic
        pm.max_children = 75
        pm.start_servers = 8
        pm.min_spare_servers = 5
        pm.max_spare_servers = 20
        pm.max_requests = 500
        ```
7. Edit the top level php.ini file in `/etc/php/7.0/fpm/php.ini`:
    - Make a backup copy.
    - Add **just** the following inside:
        ```
        [PHP]
        engine = On
        short_open_tag = Off
        asp_tags = Off
        precision = 14
        output_buffering = 4096
        zlib.output_compression = Off
        implicit_flush = Off
        unserialize_callback_func =
        serialize_precision = 17
        disable_functions =
        disable_classes =
        zend.enable_gc = On
        expose_php = Off
        max_execution_time = 30
        max_input_time = 60
        memory_limit = 128M
        error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
        display_errors = Off
        display_startup_errors = Off
        log_errors = On
        log_errors_max_len = 1024
        ignore_repeated_errors = Off
        ignore_repeated_source = Off
        report_memleaks = On
        track_errors = Off
        html_errors = On
        variables_order = "GPCS"
        request_order = "GP"
        register_argc_argv = Off
        auto_globals_jit = On
        post_max_size = 8M
        auto_prepend_file =
        auto_append_file =
        default_mimetype = "text/html"
        default_charset = "UTF-8"
        doc_root =
        user_dir =
        enable_dl = Off
        cgi.fix_pathinfo=0
        file_uploads = On
        upload_max_filesize = 25M
        max_file_uploads = 20
        allow_url_fopen = On
        allow_url_include = Off
        default_socket_timeout = 60
        [CLI Server]
        cli_server.color = On
        [Date]
        [filter]
        [iconv]
        [intl]
        [sqlite]
        [sqlite3]
        [Pcre]
        [Pdo]
        [Pdo_mysql]
        pdo_mysql.cache_size = 2000
        pdo_mysql.default_socket=
        [Phar]
        [mail function]
        SMTP = localhost
        smtp_port = 25
        mail.add_x_header = On
        [SQL]
        sql.safe_mode = Off
        [ODBC]
        odbc.allow_persistent = On
        odbc.check_persistent = On
        odbc.max_persistent = -1
        odbc.max_links = -1
        odbc.defaultlrl = 4096
        odbc.defaultbinmode = 1
        [Interbase]
        ibase.allow_persistent = 1
        ibase.max_persistent = -1
        ibase.max_links = -1
        ibase.timestampformat = "%Y-%m-%d %H:%M:%S"
        ibase.dateformat = "%Y-%m-%d"
        ibase.timeformat = "%H:%M:%S"
        [MySQL]
        mysql.allow_local_infile = On
        mysql.allow_persistent = On
        mysql.cache_size = 2000
        mysql.max_persistent = -1
        mysql.max_links = -1
        mysql.default_port =
        mysql.default_socket =
        mysql.default_host =
        mysql.default_user =
        mysql.default_password =
        mysql.connect_timeout = 60
        mysql.trace_mode = Off
        [MySQLi]
        mysqli.max_persistent = -1
        mysqli.allow_persistent = On
        mysqli.max_links = -1
        mysqli.cache_size = 2000
        mysqli.default_port = 3306
        mysqli.default_socket =
        mysqli.default_host =
        mysqli.default_user =
        mysqli.default_pw =
        mysqli.reconnect = Off
        [mysqlnd]
        mysqlnd.collect_statistics = On
        mysqlnd.collect_memory_statistics = Off
        [OCI8]
        [PostgreSQL]
        pgsql.allow_persistent = On
        pgsql.auto_reset_persistent = Off
        pgsql.max_persistent = -1
        pgsql.max_links = -1
        pgsql.ignore_notice = 0
        pgsql.log_notice = 0
        [Sybase-CT]
        sybct.allow_persistent = On
        sybct.max_persistent = -1
        sybct.max_links = -1
        sybct.min_server_severity = 10
        sybct.min_client_severity = 10
        [bcmath]
        bcmath.scale = 0
        [browscap]
        [Session]
        session.save_handler = files
        session.use_strict_mode = 0
        session.use_cookies = 1
        session.use_only_cookies = 1
        session.name = PHPSESSID
        session.auto_start = 0
        session.cookie_lifetime = 0
        session.cookie_path = /
        session.cookie_domain =
        session.cookie_httponly =
        session.serialize_handler = php
        session.gc_probability = 1
        session.gc_divisor = 1000
        session.gc_maxlifetime = 1440
        session.referer_check =
        session.cache_limiter = nocache
        session.cache_expire = 180
        session.use_trans_sid = 0
        session.hash_function = 0
        session.hash_bits_per_character = 5
        url_rewriter.tags = "a=href,area=href,frame=src,input=src,form=fakeentry"
        [MSSQL]
        mssql.allow_persistent = On
        mssql.max_persistent = -1
        mssql.max_links = -1
        mssql.min_error_severity = 10
        mssql.min_message_severity = 10
        mssql.compatibility_mode = Off
        mssql.secure_connection = Off
        [Assertion]
        [COM]
        [mbstring]
        [gd]
        [exif]
        [Tidy]
        tidy.clean_output = Off
        [soap]
        soap.wsdl_cache_enabled=1
        soap.wsdl_cache_dir="/tmp"
        soap.wsdl_cache_ttl=86400
        soap.wsdl_cache_limit = 5
        [sysvshm]
        [ldap]
        ldap.max_links = -1
        [dba]
        [opcache]
        [curl]
        [openssl]
        ```
8. `cgi.fix_pathinfo=0` - **Make sure this is set!**