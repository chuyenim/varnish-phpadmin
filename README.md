# Varnish PHP Admin
## A PHP-based Varnish Administration Tool

This tool will allow you to administer your varnish server via a web interface. It supports:

 - Purging by hostname and URI.
 - Banning with URI regex or full ban queries.
 - Display of top line stats from `varnishstat` with 3 different verbosity levels.
 - Display of stat medians over 10, 30 and 60 minutes
 - Display of top URIs, missed URIs and user agents.
 - Password protection.

## Installation

### Requirements

Before installing, ensure you have the following:

 - An Apache web server (at least version 2).
 - PHP 5.6.
 - `sudo` or root access to your webserver.
 - An installation of Varnish (At least 4.0) with `varnishstat` and `varnishtop`.
 - An active, working Cron.
 - Some knowledge of Varnish and VCL.

### Before you begin

Varnish PHP Admin is built in such a way to to minimise its footprint and requirements set.
There are no external requirements beyond the minimum listed above. The PHP file is almost totally
self-contained and uses the Bootstrap CDN for styling and interaction.

The exception being the `settings.php` file. This is a small file which returns an array for use
by Varnish PHP Admin to define its environment variables. It is written in PHP to ensure that
the settings cannot be accidentally viewed via the webserver, as well as allowing it to sit next
to the main `index.php` file.

There is also a shell script which Varnish PHP Admin relies on to create the data with which it
displays statistics. Without this, the PHP cannot directly access the data generated by `varnishstat`
or `varnishtop` as it is rarely accessible by non eleveated users such as apache. It would also be
unable to provide historical data.

### 1) Generating the settings file.

For obvious reasons, the settings file does not come pre-populated. You must fill it in
yourself. Thankfully, it's pretty straightforwards and a template is below:

```php
<?php
/* This is a sample configuration file for Varnish PHP Admin
 * Please place this file next to the index.php file wherever it is installed
 * and ensure it is named "settings.php".
 */
return array(
	'password' => '',
	'varnish_data_path' => '/var/varnish/data',
	'apache_hosts_path' => '/path/to/apache/hosts',
	'varnish_socket_ip' => '127.0.0.1',
	'varnish_socket_port' => '80',
	'timezone' => 'Europe/London'
);
```

The settings file is comprised of four main options:

 - `password` - The password your Varnish PHP Admin will be protected with. This cannot be blank.
 - `varnish_data_path` - The root path defined by `varnishstat-json.sh`. More on this further down.
 - `apache_hosts_path` - The root path containing the `hostxyz.conf` files defining the available virtual hosts on your web server.
 - `varnish_socket_ip` - The IP address at which your Varnish server can be accessed. This is usually the same as your web server.
 - `varnish_socket_port` - The port number at which your varnish server can be accessed. This is usually set to `80` to replace apache.
 - `timezone` - Defaults to GMT, set to make local to your area. ([Full list of timezones](http://php.net/manual/en/timezones.php)).

#### More about `apache_hosts_path`

If you don't have a consistent folder for the apache virtual host files, don't worry - you can do one of two things:

 - Create a folder of symlinks, pointing to the individual vhost files, or
 - Leave the setting empty, in which case a plain text field will be provided for defining the host during PURGE or BAN operations.

Once you have a `settings.php` file, place next to the `index.php` file and it will provide options to the Varnish PHP Admin script.

### 2) Using `varnishstat-json.sh`

This is a simple shell script which provides two purposes:

 1. To run as a root user, generating file-based data from `varnishstat` and `varnishtop`.
 2. To provide historical data on a rolling 1 minute basis.

The idea behind this shell script is to run as the root user (or any user who can access the varnish administrative binaries as well as set permissions on the resulting files). It should run every 1 (one) minute from a crontab using a command similar to the following:

```
# assuming the script is placed in /var/varnish
  */1 *  *   *   *     /var/varnish/varnishstat-json.sh /var/varnish/data
```

The argument after the script filename defines where the data will be placed. The structure of the data is as follows:

```
/var/varnish/data/
 > top.json
 > top-misses.json
 > top-ua.json
 > stat/
	 > ##.json
```

Each of the files generated within `stat/` are based on the minute of the hour in which the script runs. They will roll over every 60 minutes. The Varnish PHP Admin script will use these stats to produce historical median data.

### 3) Set up the varnish VCL

If you already have Varnish set up, you may already have a VCL file. If so, great! Read on. Otherwise, you should look into [how VCL files work](https://www.varnish-cache.org/docs/index.html) first.

Before supporting PURGE and/or BAN requests, an [ACL](https://www.varnish-cache.org/docs/4.1/users-guide/purging.html) will need to be created. You can do this with the following code:

```
# Before the subroutines
acl purge {
	# ACL we'll use later to allow PURGE or BAN
	"localhost";
	"127.0.0.1";
	"your.servers.external.ip";
	"::1";
}
```

You'll need to support PURGE requests in the VCL. This can be done with the following logic (although feel free to use your own):

```
sub vcl_recv {
	if (req.method == "PURGE") {
		if (!client.ip ~ purge) { # purge is the ACL defined at the begining
			# Not from an allowed IP? Then die with an error.
			return (synth(405, "This IP is not allowed to send PURGE requests."));
		}

		# If you got this stage (and didn't error out above), purge the cached result
		return (purge);
	}
}
```

In addition to that, you can add support for the BAN requests with some custom logic:


```
sub vcl_recv {
	if (req.method == "BAN") {
		if (!client.ip ~ purge) { # purge is the ACL defined at the begining
			# Not from an allowed IP? Then die with an error.
			return (synth(405, "This IP is not allowed to send BAN requests."));
		}

		if (req.http.Ban-Query-Full) {
			# A "full" ban query (the result of checking "Full Ban Query")
			ban(req.http.Ban-Query-Full);
			return (synth(200, "Full ban added"));
		} else if (req.http.Ban-Query) {
			# A normal URI based ban query
			ban("req.http.host == " + req.http.host + " && req.url ~ " + req.http.Ban-Query);
			return (synth(200, "Ban added"));
		} else {
			return (synth(405, "Ban query sent without Ban-Query or Ban-Query-Full headers."));
		}
	}
}
```

The above two snippets define two `vcl_recv` subroutines - This is intentional and done so for clarity within the documentation, although in reality while it is not essential, you may want to combine the above logic to support PURGE and BAN requests in one go.

Altering your `.vcl` file is completely optional, but without it you might not be able to use the PURGE/BAN features of the actions form.

#### Important note about BANS in the Varnish PHP Admin

You have probably noticed above that in the example handling snippet there are references to headers named `Ban-Query` and `Ban-Query-Full`. These headers are sent by the Varnish PHP Admin whenever a BAN request is being performed. Due to the fact that the query could be any number of complex characters forming a posix-style regular expression or VCL logic, they are sent as URL-encoded headers in order to ensure they are transported to the Varnish interface correctly. You **must** add support for these headers if you want to use the BAN feature of the actions form on the Varnish PHP Admin page.

### 4) Finish up

Once you have the `index.php` and `settings.php` in a publicly accessible location on your web server, and the `varnishstat-json.sh` file running on a 1 minute interval, and the `.vcl` file set up within Varnish, everything should work as expected.

## Using the Varnish PHP Admin

When you first load the admin page, you'll be prompted to enter your password. This will only happen once per session. Once you're in, you should see a screen similar to the following:

![https://raw.githubusercontent.com/njpanderson/varnish-phpadmin/master/screenshot.png](Main page screenshot)

The interface is comprised of three main parts:

 1. The **actions** form, which allows you to perform PURGE/BAN requests on the Varnish enabled server.
 2. The **Stats** display, giving you a current (within ~1 minute) display of data as well as up to 60 minute median data.
 3. The **Top** display, which gives historical statistics on the top visited URIs, missed URIs and user agents.

### Purging

Purging is easily done with the **actions** form:

 1. Enter a host in the `Host` field either by selecting from the dropdown list or by keying in a hostname manually.
 2. Key in a URI into the `Query` field. This URI will be purged and the response will be shown below the form.

### Banning

While more powerful, banning is slightly more complicated.

#### Banning a URI

 1. As with PURGEs, enter or select a host with the `Host` field.
 2. Enter a URI or posix-style regex to ban within the `Query` field. There are some examples below.

#### Banning based on a VCL query

This can be done by checking the `Full ban query` checkbox (and heeding the warning).

In this state, the `Query` field becomes the raw argument which will be sent to the `ban` function within the Varnish VCL subroutines. This allows you to perform queries with multiple arguments and logic.

See [Varnish-CLI](https://www.varnish-cache.org/docs/4.1/reference/varnish-cli.html#varnish-cli-7) for more information on ban queries.

*Example ban URI regular expressions:*

 - **Everything:** `.*`
 - **Common inages:** `\.(gif|jpeg|jpg|png)(\?.*)?$`
 - **CSS and LESS fies:** `\.(s?css|less)(\?.*)?$`
 - **All JS files:** `\.(jsx?)(\?.*)?$`

### Displaying more information within Stats

If you want to see more information within the Stats panel, just choose one of the options from the 'Show' menu.

If a stat is not displayed even if 'Show' is set to 'All', then it is likely because the data is currently at zero, and it is being hidden.

## Troubleshooting

### Fatal Error! "The "settings.php" file does not exist. Have you created it?"

This one should be pretty clear already but if you're not sure, it's just a case of following step **1** of the installation to create the `settings.php` file which should be placed next to the `index.php` file wherever you've installed the Varnish PHP Admin page.

### Fatal Error! "Socket could not be opened to host."

When PURGE-ing or BAN-ing, Varnish PHP Admin will attempt to connect to your server using the IP and port you defined within `settings.php`. It also sends the `Host` header which you have defined within the actions form at the top.

If your `varnish_socket_ip` or `varnish_socket_port` options in the `settings.php` file are pointing to the wrong place, or the port defined is not opened by your firewall, the Varnish PHP Admin can have trouble creating a socket. Check that your server is accessible at the IP/port you have defined.

### Fatal Error! "Varnish stat path "/your/path" could not be found or could not be read."

The shell script `varnishstat-json.sh` attempts to ensure that the statistic files it creates are world-readable for you, but it likely needs to run as an elevated user in order to do so. This will also help ensure it can run the `varnishstat` and `varnishlog` commands as necessary.

Ensure you have the shell script set up as per step **2** of the installation instructions.

If everything is set up, you may also want to check the `varnish_data_path` option in the `settings.php` file is also pointing to the same path as the shell script command. For instance:

```
# in crontab:
/var/varnish/varnishstat-json.sh /some/path/to/my/data

# in settings.php:
	'varnish_data_path' => '/some/path/to/my/data',
```

### Fatal Error! "Password not defined. Please define a password before continuing!"

You must define a `password` option within the Varnish PHP Admin `settings.php` file. There are no restrictions on what this password can be, except that it cannot be empty.

### The 'hosts' field in the actions form is just a text field

This will happen if the `.conf` files defining your VHOST directives in Apache cannot be found or are not readable by the apache instance running the server. It's not a major problem, as you can type in the host name anyway, but otherwise, refer to "More about `apache_hosts_path`" above.

### PURGE requests don't work!

If your PURGE requests aren't giving a socket error but otherwise aren't working, check the following:

 1. That you are typing/selecting the correct host name to purge.
 2. That you have entered the correct URI (without the hostname, starting with a slash, e.g: `/` would be the homepage.
 3. That the `.vcl` file is set up to acccept PURGE requests and is accessible via the `varnish_socket_ip` and `varnish_socket_port` defined in your `settings.php` file.

### BAN requests don't work!

If your BAN requests aren't giving a socket error but otherwise aren't working, check the following:

 1. That you are typing/selecting the correct host name to ban.
 2. That you have entered the correct regular expression (without the hostname, starting with a slash, e.g: `.*\.(jpg|jpeg)` would ban all JPEG filenames.
 3. That the `.vcl` file is set up to acccept BAN requests and is accessible via the `varnish_socket_ip` and `varnish_socket_port` defined in your `settings.php` file.
 4. That your expression is not incorrectly formed in the case of sending a full ban request.

### Any other errors, issues or questions

Feel free to [start me an issue](https://github.com/njpanderson/varnish-phpadmin/issues) and I'll get back to you within a reasonable time. Keep in mind that while I will likely be happy to help, this is an open source project and I have a day job.