autofilter (version 2.0)
========================

Automatically mitigate layer 7 DDoS attacks.

Installation
------------

- ``cd /opt``
- ``git clone https://github.com/makhomed/autofilter.git autofilter``

Also you need to install python3, unbound, dnspython and netaddr:

.. code-block:: none

    # yum install python3
    # yum install unbound
    # vim /etc/unbound/unbound.conf
            interface: 127.0.0.1
            do-ip6: no
    # systemctl enable unbound
    # systemctl start unbound

    # pip3 install dnspython
    # pip3 install netaddr

Upgrade
-------

- ``cd /opt/autofilter``
- ``git pull``

Configuration
-------------

Configuration file ``/opt/autofilter/autofilter.conf`` is optional.

If no config file provided - default built-in config is used:

.. code-block:: none

    block ALL 24h

    limit ALL 128 32

Configuration file allow comments, from symbol ``#`` to end of line.

Configuration file has only two directives: ``limit`` and ``block``.

``limit`` directive has syntax: ``limit <entity> <total_request_count_threshold> <one_uri_request_count_threshold>``.

``<entity>`` can be ``ALL``, or country code, for example, ``UA`` or ``RU`` or ``CN``.
Also ``<entity>`` can be IP address, ipv4 or ipv6 or IP network in CIDR notation.

``<total_request_count_threshold>`` is total request count after which specific ip address will be blocked.

``<one_uri_request_count_threshold>`` is request count to only one uri after which specific ip address will be blocked.

Each threshold is integer number or special value ``none``.

Request count measured as one-minute sum of request weight for each request from each ip.
One request to static resource or one nginx-level redirect measured as weight 0.1,
all other requests considered as requests to backend, and has weight 1.0.

If some specific ip generates load above threshold - this ip will be blocked as bot.

Search engine bots from Google, Yandex and Bing are detected automatically and will be never blocked.

By default ``limit ALL 128 32`` if other value not specified in config.

``block`` directive has syntax: ``block <entity> <time>``.

``<entity>`` can be ``ALL``, or country code, for example, ``UA`` or ``RU`` or ``CN``.
Also ``<entity>`` can be IP address, ipv4 or ipv6 or IP network in CIDR notation.

``<time>`` is block time in hours or in days, suffix ``h`` or ``d`` is required.
For example: ``24h`` or ``3d``.

By default ``block ALL 24h`` if other value not specified in config.

nginx configuration
~~~~~~~~~~~~~~~~~~~

nginx global configuration context:

.. code-block:: none

    worker_shutdown_timeout 60s;

nginx configuration in context http for standalone server:

.. code-block:: none

    geo $geoip_country_code {
        default XX;
        include /etc/nginx/geo/geoip_country_code.conf;
    }

    geo $bot {
        default 0;
        include /opt/autofilter/var/bot.conf;
    }

    map $bot $loggable {
        0 1;
        1 0;
    }

    map $time_iso8601 $time {
        "~([0-9-]+)T([0-9:]+)" "$1 $2";
        volatile;
    }

    log_format standalone '$time\t$geoip_country_code\t$remote_addr\t$upstream_cache_status'
                       '\t$upstream_response_time\t$status\t$scheme\t$host\t$request_method'
                         '\t$request_uri\t$body_bytes_sent\t$http_referer\t$http_user_agent';

    access_log /var/log/nginx/access.log standalone if=$loggable;

File ``/etc/nginx/geo/geoip_country_code.conf`` can be generated with `nginx-geo <https://github.com/makhomed/nginx-geo>`_

nginx configuration in context http for nginx frontend server:

.. code-block:: none

    geo $geoip_country_code {
        default XX;
        include /etc/nginx/geo/geoip_country_code.conf;
    }

    geo $bot {
        default 0;
        include /opt/autofilter/var/bot.conf;
    }

    map $bot $loggable {
        0 1;
        1 0;
    }

    map $time_iso8601 $time {
        "~([0-9-]+)T([0-9:]+)" "$1 $2";
        volatile;
    }

    log_format frontend '$time\t$geoip_country_code\t$remote_addr\t$upstream_http_x_cache_status'
                     '\t$upstream_http_x_response_time\t$status\t$scheme\t$host\t$request_method'
                              '\t$request_uri\t$body_bytes_sent\t$http_referer\t$http_user_agent';

    access_log /var/log/nginx/access.log frontend if=$loggable;

    proxy_set_header  Host $host;
    proxy_set_header  X-Real-IP $remote_addr;
    proxy_set_header  X-Forwarded-Https $https;
    proxy_set_header  X-Forwarded-Proto $scheme;
    proxy_set_header  X-GeoIP-Country-Code $geoip_country_code;

nginx configuration in context http for nginx backend server:

.. code-block:: none

    set_real_ip_from   {{ nginx frontend ip }};
    real_ip_header     X-Real-IP;

    add_header X-Cache-Status $upstream_cache_status always;
    add_header X-Response-Time $upstream_header_time always;

    map $time_iso8601 $time {
        "~([0-9-]+)T([0-9:]+)" "$1 $2";
        volatile;
    }

    log_format backend  '$time\t$http_x_geoip_country_code\t$remote_addr\t$upstream_cache_status'
            '\t$upstream_response_time\t$status\t$http_x_forwarded_proto\t$host\t$request_method'
                              '\t$request_uri\t$body_bytes_sent\t$http_referer\t$http_user_agent';

    access_log /var/log/nginx/access.log backend;


nginx configuration in context server for standalone and nginx frontend servers:

.. code-block:: none

    if ( $bot ) { return 429; }

| **Warning!!!** If this line not included in server context - server will be unprotected from DDoS.

| **Warning!!!** If nginx ``log_format`` changed - you need to rotate nginx access.log file.


Automation via systemd service
------------------------------

Create configuration file ``/opt/autofilter/autofilter.conf`` and define limits.
After it create systemd service, for example, in file ``/etc/systemd/system/autofilter.service``:

.. code-block:: none

    [Unit]
    Description=autofilter
    After=unbound.service

    [Service]
    ExecStart=/opt/autofilter/autofilter
    Restart=always

    [Install]
    WantedBy=multi-user.target

After this you need to start service:

- ``systemctl daemon-reload``
- ``systemctl enable autofilter``
- ``systemctl start autofilter``
- ``systemctl status autofilter``

If all ok you will see what service is enabled and running.

