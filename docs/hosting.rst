Hosting
=======

* Use `Digital Ocean <https://www.digitalocean.com/>`_

Server Setup
------------

1. Create a Ubuntu droplet on the latest LTS version
2. `Login <https://www.digitalocean.com/docs/droplets/how-to/connect-with-ssh/>`_
3. `Add a non-root user <https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-18-04>`_
4. Configure the non-root user in your local ``~/.ssh/config``

    .. code:: bash

        Host droplet
          Hostname <ip>
          User <user>

5. `Configure firewall <https://www.digitalocean.com/docs/networking/firewalls/quickstart/>`_

    * Also add HTTP+HTTPS

6. `Configure nameservers <https://www.digitalocean.com/community/tutorials/how-to-point-to-digitalocean-nameservers-from-common-domain-registrars>`_  for each domain you want served by this droplet
7. `Configure DNS <https://www.digitalocean.com/docs/networking/dns/quickstart/>`_ for each domain you want served by this droplet
8. `Setup Nginx <https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04>`_

.. _static-website-hosting:

Static Website
--------------

Create a directory to hold the site's files:

.. code:: bash

    sudo mkdir -p /var/www/<hostname>/html
    sudo chown -R $USER:$USER /var/www/<hostname>/html

Create ``/etc/nginx/sites-available/<hostname>``:

.. code:: bash

    server {
        listen 80;
        listen [::]:80;

        root /var/www/<hostname>/html;
        index index.html index.htm index.nginx-debian.html;

        server_name <hostname> www.<hostname>;

        location / {
            try_files $uri /index.html;
        }
    }

.. code:: bash

    sudo ln -s /etc/nginx/sites-available/<hostname> /etc/nginx/sites-enabled/
    sudo systemctl restart nginx

Then `Configure SSL <https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-18-04>`_

.. _wsgi-app-hosting:

WSGI App
--------

* Use `uWSGI <https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uswgi-and-nginx-on-ubuntu-18-04>`_

Create the app entrypoint in ``<myapp-repo>/wsgi.py``:

.. code:: python

    from myapp import create_app

    app = create_app()

Configure uWSGI in ``<myapp-repo>/myapp.ini``:

.. code:: ini

    [uwsgi]
    module = wsgi:app

    master = true
    processes = 5

    socket = myapp.sock
    chmod-socket = 660
    vacuum = true

    die-on-term = true

    logto = /var/log/myapp/%n.log

Create directories for the application code and logs:

.. code:: bash

    sudo mkdir -p /var/app/myapp
    sudo chown -R $USER:$USER /var/app/myapp
    sudo mkdir -p /var/log/myapp
    sudo chown -R $USER:$USER /var/log/myapp

Create ``/etc/systemd/system/myapp.service`` to run the app under supervision:

.. code:: ini

    [Unit]
    Description=uWSGI instance to serve myapp
    After=network.target

    [Service]
    User=<non-root-user>
    Group=www-data
    WorkingDirectory=/var/app/myapp
    Environment="PATH=/var/app/myapp/venv/bin"
    ExecStart=/var/app/myapp/venv/bin/uwsgi --ini myapp.ini

    [Install]
    WantedBy=multi-user.target

Start the service and set it to run on boot:

.. code:: bash

    sudo systemctl start myapp
    sudo systemctl enable myapp

Create ``/etc/nginx/sites-available/<hostname>``:

.. code::

    server {
        listen 80;
        listen [::]:80;

        server_name <hostname>;

        location / {
            include uwsgi_params;
            uwsgi_pass unix:/var/app/myapp/myapp.sock;
        }
    }

.. code:: bash

    sudo ln -s /etc/nginx/sites-available/<hostname> /etc/nginx/sites-enabled/
    sudo systemctl restart nginx

Then `Configure SSL <https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-18-04>`_
