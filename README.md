# How to setup apache2 for SSO using Kerberos

## Introduction
I've had no success in finding tutorial that shows how to secure a server by using a reverse proxy (in this case apache2) with SSO and Kerberos.

## Pre Requisites
- Windows server with an active directory domain dontroller installed and set up
- Ubuntu 20.04 (newer will probably work just as well) server

Both of those machines will need to be able to talk to each other over the network.

## AD configuration / setup
It's advised to create a service user which will handle the Kerberos authentication. In my case, I just created an ad-user with the name "apache2".

Next step is to create a service principal (spn) for this user and the webserver. Open powershell on the windows server and execute the following:

```
setspn -A HTTP/<dns_of_webserver> <service_user>
```
Example:
```
setspn -A HTTP/webserver.local.test apache2
```
> Hint: local.test should be your domain

## DNS entries
We will be using the dns server on the domain-controller. To add an entry, go to the server manager > Tools > DNS.
Here, go to your forward lookup zone and with right-click > New Host (A or AAAA) create a DNS-Name for your webserver if it is not public and intended for internal use.

## Ubuntu setup
To be up to date, install the missing updates by entering:
```
sudo apt update && sudo apt dist-upgrade -y
```

Now install apache2:
```
sudo apt install apache2 -y
```

And krb5:
```
sudo apt install krb5-user -y
```

During the installation you will be asked to enter the Realm/KDC/Admin server. Enter the correct values but after you can always edit/change what you entered.

### krb5 config
The krb5 configuration file can be found under `/etc/krb5.conf`.
Edit the file to look something like this:
```
[libdefaults]
	default_realm = LOCAL.TEST

	kdc_timesync = 1
	ccache_type = 4
	forwardable = true
	proxiable = true
	fcc-mit-ticketflags = true

[realms]
	LOCAL:TEST = {
		kdc = DOMAIN-CONTROLLER.LOCAL.TEST
		admin_server = DOMAIN-CONTROLLER.LOCAL.TEST
	}
[domain_realm]
    domain-controller.local.test = LOCAL.TEST DOMAIN-CONTROLLER.LOCAL.TEST = LOCAL.TEST local.test = LOCAL.TEST .local.test = LOCAL.TEST
```

### Generating a kerberos keytab file
Now you need to generate a keytab file which will be used by apache in order to know where and how to authenticate the users trying to connect to the webserver.
For this we will use the `ktutil` tool which was installed by the `krb5-user` package.

Enter `ktutil` to start using it.
Now add an entry by executing:
```
add_entry -password -p http/<webserver name>@<realm> -k1 -f
```

For example:
```
add_entry -password -p http/webserver.local.test@LOCAL.TEST -k1 -f
```

by typing `list` you will see the created entry from before.
Now to save this to a keytab file enter:
```
wkt /<directory_to_save_keytab_file>/kerberos.keytab
```

Copy the keytab file to `/etc/apache2/kerberos.keytab`:
```
sudo cp /<directory_to_save_keytab_file>/kerberos.keytab /etc/apache2/kerberos.keytab
```

Now set the permissions for this file (use the user which runs the apache2 process, usually `www-data`)
```
sudo chown www-data:www-data /etc/apache2/kerberos.keytab && sudo chmod 400 www-data:www-data /etc/apache2/kerberos.keytab
```

### Install mod_auth_gssapi apache2 module
You will need to install the mod_auth_gssapi apache2 module to setup the kerberos authentication:

```
sudo apt install libapache2-mod-auth-gssapi -y
```

## Apache config
Apache2 by default creates a default config file which we will use `/etc/apache2/sites-available/000-default.conf`.
Open the file and change the content to the following:

```
LoadModule headers_module /usr/lib/apache2/modules/mod_headers.so
LoadModule rewrite_module /usr/lib/apache2/modules/mod_rewrite.so
LoadModule proxy_module /usr/lib/apache2/modules/mod_proxy.so
LoadModule proxy_http_module /usr/lib/apache2/modules/mod_proxy_http.so
LoadModule proxy_ajp_module /usr/lib/apache2/modules/mod_proxy_ajp.so

<VirtualHost *:80>
	<Proxy *>
		Order deny,allow
		Allow from all
	</Proxy>
	ProxyPass / <webserver where content will get pulled from>/
	<Location />
		AuthType GSSAPI
		AuthName "Kerberos authenticated intranet with mod_auht_gssapi"

		GssapiCredStore keytab:/etc/apache2/kerberos.keytab

		# We don't want to fallback to Basic Auth
		GssapiBasicAuth Off

		# Resolve remote's user into REMOTE_USER variable. Proper setting of [realms].auth_to_local in /etc/krb5.conf is required
		GssapiLocalName On

		require valid-user

		RequestHeader unset userid
    RewriteEngine On
    RewriteCond   %{LA-U:REMOTE_USER} (.+)
    RewriteRule   /.* - [E=RU:%1,L,NS]
    RequestHeader set userid %{RU}e

	</Location>

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Some Application/Scenarios require the proxy to set the username as a request header.
> The RequestHeader userid will set the username as the header without the domain, for example: max.muster instead of max.muster@local.test

Now you can restart the apache2 service:
```
sudo service apache2 restart
```
