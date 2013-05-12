Installing hushfile
===================

The hushfile server is not dependent on any specific client. However most people will probably want to run the server and the webclient on the same webserver, like on https://hushfile.it. The following commands will check out both the webclient and the server to a given webserver.

    git clone git://github.com/hushfile/hushfile-web.git /usr/local/www/hushfile
    git clone git://github.com/hushfile/hushfile-server.git /usr/local/www/hushfile/api

If you only want to run the server you just run the second command.

nginx configuration
-------------------
To make nginx route the requests properly, something like the following config is needed:

    root                    /usr/local/www/dev.hushfile.it;
    client_max_body_size    1000M;

    location / {
        index                   hushfile.html;
        try_files               $uri /hushfile.html;
    }

    location /api/ {
        try_files               $uri /api/hushfile.php;
    }

    location ~ /api/hushfile.php$ {
        try_files $uri =404;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PHP_ADMIN_VALUE "open_basedir=$document_root/..";
        fastcgi_pass unix:/var/php/php-fpm.socket;
    }

This obviously needs to be adapted to match your servers way of handling PHP. If you have problems getting the server working, open an issue https://github.com/hushfile/hushfile-server/issues or join the IRC channel #hushfile on Freenode.
