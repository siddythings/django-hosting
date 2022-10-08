# Django Hosting on Nginx with uWSGI


## 1. Update Your Server

It’s always a good idea to start by updating your server and installing .

```
sudo apt-get update 
sudo apt install python3-pip python3-dev libpq-dev nginx curl uwsgi
```

## 2. Use a Virtual Environment for Python
```
sudo -H pip3 install virtualenv
virtualenv venv
```

Now activate your virtual environment.

```
source venv/bin/activate
```

## 3. Clone/Create a Django Project

Create or clone your project.

```
cd testoin-project
```

If you have any requrements file the install 
```
pip install -r requrements.txt
```
Test your project 
```
python manage.py runserver 0.0.0.0:8000
```

## 4. Get Started With uWSGI

![nginx-uwsgi-django](/images/nginx-uwsgi-django-stack.webp)

Now we serve the Django project with uWSGI with the following command:

```
uwsgi --http :8000 --module application.wsgi
```

Note that this works because the path `application/wsgi.py` exists. Test it out in a browser.

## 5. Configure the Nginx Web Server

Let’s tell Nginx about our Django project by creating a configuration file at `/etc/nginx sites-available/testoin.conf`. Change the highlighted lines to suite your needs.

```
# the upstream component nginx needs to connect to
upstream django {
    server unix:///home/siddythings/testoin-project/testoin.sock;
}

# configuration of the server
server {
    listen      80;
    server_name <ip_address/domain>;
    charset     utf-8;

    # max upload size
    client_max_body_size 75M;

    # Django media and static files 
    location /media  {
        alias /home/siddythings/testoin-project/media;
    }
    location /static {
        alias /home/siddythings/testoin-project/static;
    }

    # Send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /home/siddythings/testoin-project/uwsgi_params;
    }
}
```

We need to create the `/home/udoms/microdomains/uwsgi_params` file defined in testoin.conf file.

```
uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;
uwsgi_param  REQUEST_URI        $request_uri;
uwsgi_param  PATH_INFO          $document_uri;
uwsgi_param  DOCUMENT_ROOT      $document_root;
uwsgi_param  SERVER_PROTOCOL    $server_protocol;
uwsgi_param  REQUEST_SCHEME     $scheme;
uwsgi_param  HTTPS              $https if_not_empty;
uwsgi_param  REMOTE_ADDR        $remote_addr;
uwsgi_param  REMOTE_PORT        $remote_port;
uwsgi_param  SERVER_PORT        $server_port;
uwsgi_param  SERVER_NAME        $server_name;
```

Next we can publish our changes by creating a symbolic link from sites-available to sites-enabled like so:

`sudo ln -s /etc/nginx/sites-available/testoin.conf /etc/nginx/sites-enabled/`

## 6. Get Nginx, uWSGI, and Django to Work Together


```
uwsgi --socket testoin.sock --module application.wsgi --chmod-socket=666
```

Visit your IP/Domain in a drowser, and this time you should see the default Django landing page

## 7. Configure uWSGI for Production

This time we don't pass arguments to uWSGI like above, we can put these options in a config file at the root of your Django project `testoin_uwsgi.ini`

```
[uwsgi]
# full path to Django project's root directory
chdir            = /home/siddythings/testoin-project/
# Django's wsgi file
module           = microdomains.wsgi
# full path to python virtual env
home             = /home/siddythings/venv
# enable uwsgi master process
master          = true
# maximum number of worker processes
processes       = 10
# the socket (use the full path to be safe
socket          = /home/siddythings/testoin-project/testoin.sock
# socket permissions
chmod-socket    = 666
# clear environment on exit
vacuum          = true
# daemonize uwsgi and write messages into given log
daemonize       = /home/siddythings/uwsgi-emperor.log

```

You can then proceed to start up uwsgi and specify the `ini` file:

```
uwsgi --ini testoin_uwsgi.ini
```

Visit your IP/Domain in a drowser, and this time you should see the default Django landing page


As one last configuration option for uWSGI, let’s run uWSGI in emperor mode. This will monitor the uWSGI config file directory for changes and will spawn vassals (i.e. instances) for each one it finds.

```
cd /home/siddythings/venv/
mkdir vassals
sudo ln -s /home/siddythings/testoin-project/testoin_uwsgi.ini /home/siddythings/testoin-project/vassals/
```

Now you can run uWSGI in emperor mode as a test.

```
uwsgi --emperor /home/siddythings/venv/vassals --uid www-data --gid www-data
```
Visit your IP/Domain in a drowser, and this time you should see the default Django landing page

Finally, we want to start up uWSGI when the system boots. Create a systemd service file at `/etc/systemd/system/emperor.uwsgi.service` with the following content:

```
[Unit]
Description=uwsgi emperor for micro domains website
After=network.target
[Service]
User=udoms
Restart=always
ExecStart=/home/siddythings/venv/bin/uwsgi --emperor /home/siddythings/venv/vassals --uid www-data --gid www-data
[Install]
WantedBy=multi-user.target
```

Enable the service to allow it to execute on system boot and start it so you can test it without a reboot.

```
systemctl enable emperor.uwsgi.service
systemctl start emperor.uwsgi.service
```

Visit your IP/Domain in a drowser, and this time you should see the default Django landing page