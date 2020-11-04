## Apache (httpd, easy standard, uWSGI built-in for Flask)

To do basically everything in this tutorial you __must__ have root privileges.  No exception.

If on DigitalOcean droplet, you may need to install `httpd` service.  This is a rather straightforward operation, `sudo yum install httpd`.

Next, run `sudo service httpd start`.  Make sure it works.

You need to then edit a `.conf` file for your site.  Pretty critical. 

The mywebsite site `.conf` looks like this (in all places, replace "mywebsite" with your application name, and the domain name with your domain name):

```
ServerName mywebsite.org

<VirtualHost *:80>
    ServerName mywebsite.org
    ServerAlias www.mywebsite.org
    Redirect / https://mywebsite.org
</VirtualHost>


<VirtualHost *:443>
        DocumentRoot "/var/www/html/mywebsite.org"
        ServerName mywebsite.org
        ServerAlias www.mywebsite.org

                SSLEngine on


                SSLProtocol all -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
                SSLHonorCipherOrder on
                SSLCipherSuite "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4"


                RewriteOptions inherit


                WSGIScriptAlias /dquest /data/html/mywebsite/app.wsgi

        DefaultType text/plain
        ErrorLog logs/mywebsite-error_log
        CustomLog logs/mywebsite-access_log common
        <Directory /data/html/mywebsite/FlaskApp/>
                Order allow,deny
                Allow from all
        </Directory>
        <Directory "/data/html/mywebsite/cgi-bin">
                Options +ExecCGI
                Order allow,deny
                Allow from all
        </Directory>

        ProxyPass / http://localhost:5005/
        ProxyPassReverse / http://localhost:5005/

        Include /etc/letsencrypt/options-ssl-apache.conf

        SSLCertificateFile /etc/letsencrypt/live/mywebsite.org/cert.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/mywebsite.org/privkey.pem
        SSLCertificateChainFile /etc/letsencrypt/live/mywebsite.org/chain.pem

</VirtualHost>
```

You can also replace the port, `5005` with whatever port is relevant.  Default is usually `5000` for Flask applications.  Do __not__ use 22, 80, or 443.

Then `sudo systemctl restart httpd`.

## Setting up HTTPS and cron tab

To create the `.pem` key files for HTTPS that are in the bottom of the `.conf` file, we need to use Let's Encrypt with `certbot`.
So install it:

`sudo yum install certbot python2-certbot-apache mod_wsgi`
(can probably also use python3, but CentOS 7 uses Python 2 by default)

Run `certbot certonly --standalone -d mywebsite.org` to get the .pem cert if you set up the name on DigitalOcean Control Panel.

On the DigitalOcean Control Panel you have to do something like this:
![docp](https://user-images.githubusercontent.com/6568964/93425366-d372a200-f887-11ea-9f41-5ae592e61a57.png)

Then run `sudo certbot -d mywebsite.org`.  This will ask for your domain name that __must__ be registered to the droplet's unique IP.  Don't forget to select 1 for Apache.

If successful (if not, you may need to debug your ports and use `sudo yum install fuser` to kill port 80/443 tcp listeners then restart `httpd` service again) you are almost done.

The DigitalOcean droplets (h/t to Kai for this) use SELinux (Security Enhanced Linux) so you can use `ufw` or other firewall software to choose what traffic is allowed out of ports normally, but to keep things simple we can just run this command to open up http ports:

```
setsebool -P httpd_can_network_connect 1
```

And that's about it.

Now, finally we want to make sure the certbot renews our SSL certificates for HTTPS periodically so they do not expire and so we use `crontab`:

```
echo "0 0,12 * * * root python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew -q --apache" | sudo tee -a /etc/crontab > /dev/null
```

This will renew the certificate every 12 hours (0,12).

## Nginx + uWSGI (optional alternative)

To set up with nginx for those interested (incomplete):

Settings for site in `/etc/nginx/conf.d/default.conf`.

For the 157.245.2.226 digital ocean droplet, I used the example `.conf` file here:

https://medium.com/@mightywomble/how-to-set-up-nginx-reverse-proxy-with-lets-encrypt-8ef3fd6b79e5

Then I had to install `fuser` on the droplet and kill the tcp in DO droplet:

https://www.digitalocean.com/community/questions/nginx-is-unable-to-bind-to-443

Then, because apparently NGINX doesn't support uWSGI by default, and I want NGINX's speed I installed uWSGI:

https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uswgi-and-nginx-on-ubuntu-18-04

Had to install pip as root for python 3.6 and `yum install uwsgi-plugin-python36`

To test, create virtualenv with flask and software packages necessary:

`sudo uwsgi --plugin python36 --socket 0.0.0.0:5005 --protocol=http -w wsgi:app -H /home/havrillaj/flask/`

You need to enable outgoing traffic to that port, using firewall tools like `ufw`.


