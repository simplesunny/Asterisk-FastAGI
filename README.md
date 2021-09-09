# FastAGI: AGI as a TCP Server
##### FastAGI with PHPAGI and xinetd
Similar to AGI, FastAGI provides the ability to pass variables directly from the dialplan to the FastAGI server. 
## Some housekeeping before we begin
First of all install the phpagi library (that can be found on sourceforge at http://phpagi.sourceforge.net/) and had a look at the code.
Copy the agi files to `/var/lib/asterisk/agi-bin/` and give `777` permission.

```shell
chmod -R 777 /var/lib/asterisk/agi-bin/
```

Also install few modules.
```shell
yum install xinetd
yum install ngrep
yum install php-posix
yum install php-process
```
## Configuring xinetd for FastAGI and PHPAGI
##### Configuring xinetd
As indicated previously, in order to turn our AGI script into a FastAGI server, we must first configure xinetd. 
```shell
vi /etc/xinetd.d/fastagi
```

The following is an example of what the xinetd configuration may look like. Paste the below configuration in `/etc/xinetd.d/fastagi`.
```shell
# default: off
# description: fastagi is a remote AGI interface
service fastagi
{
	socket_type  = stream
	user         = root
	group        = nobody
	server       = /var/lib/asterisk/agi-bin/phpagi-fastagi.php	
	wait         = no
	protocol     = tcp
	bind         = 127.0.0.1
	disable      = no
}
```


Make sure you set the path to the phpagi-fastagi.php script.  Set the user and group to a non-root user if none of your scripts need root access. You might consider using posix_setuid and friends to reduce privileges. Change the bind address to your outbound IP address or to 0.0.0.0 to allow anyone to connect. Be sure to read up about xinetd and take advantage of security features it provides.  Fast AGI doesn't provide authentication.  It's up to you to keep unwanted visitors from extracting information from your AGI implementation.

Once we have configured our xinetd server, we need to notify your Linux server of the new TCP service. In order to do this, add the following line to your  `/etc/services` file:
```shell
vi /etc/services
```

Paste the following at the end of `/etc/services`
```shell
fastagi         4573/tcp                # Asterisk AGI
```

The first parameter indicates the name of the new service to be rendered by xinetd. The second parameter indicates the port number and the type of IP protocol to listen to (UDP or TCP). 

Once you have finished your configuration, you may run the following command: 
`service xinetd restart`

If everything has been performed correctly, using the following command should emit an output, search for following:
```shell
netstat -ap | less
```
```shell
tcp        0      0 localhost:fastagi       0.0.0.0:*               LISTEN      53871/xinetd
```

Or

```shell
netstat -apn | less
```
```shell
tcp        0      0 127.0.0.1:4573          0.0.0.0:*               LISTEN      53871/xinetd
```

Similar to AGI, FastAGI provides the ability to pass variables directly from the dialplan to the FastAGI server. 
# Configuring PHPAGI for FastAGI
##### phpagi.conf
Before we start handling the actual FastAGI bootstrap, we must first configure the FastAGI environment of our PHPAGI class. By editing the phpagi.conf file, verify that the following code appears in it or if file does not exists create one with following content.

```shell
vi /etc/asterisk/phpagi.conf
```

```shell
[phpagi]
debug=true                              ; enable debuging
error_handler=true                      ; use internal error handler
#admin=errors@mydomain.com              ; mail errors to
#hostname=sip.mydomain.com              ; host name of this server
tempdir=/var/spool/asterisk/tmp/        ; temporary directory for storing temporary output

[asmanager]
server=localhost                        ; server to connect to
port=5038                               ; default manager port
username=sachin                         ; username for login
secret=sachin                           ; password for login

[fastagi]
setuid=true                             ; drop privileges to owner of script
basedir=/var/lib/asterisk/agi-bin/      ; path to script folder

#[festival]                             ; text to speech engine
#text2wave=/usr/bin/text2wave           ; path to text2wave binary

#[cepstral]                             ; alternate text to speech engine
#swift=/opt/swift/bin/swift             ; path to switft binary
#voice=David                            ; default voice
```

##### Your AGI file
```shell
vi /var/lib/asterisk/agi-bin/test-agi.php
```

```php
<?php
	$fastagi->verbose('cool, the FastAGI server has been called!');
	$fastagi->set_variable("test", "1111");
?>
```

##### extensions.conf
```shell
vi /etc/asterisk/extensions.conf
```

```shell
[default]
exten => _X.,1,AGI(agi://127.0.0.1/test-agi.php,12345)
exten => _X.,n,NoOp(${test})
exten => _X.,n,NoOp(${test2})
exten => _X.,n,hangup()
```

TEST by command
telnet 127.0.0.1 PORT

AGI FILE - test-agi.php
```shell
$request = $fastagi->request;
$number=$request['agi_arg_1'];
$fastagi->verbose('cool, the FastAGI server has been called!');
$fastagi->set_variable("test", "1111");
$fastagi->set_variable("test2", $number);
```

**Remove debug log from command line**
Set debug and error_handler to false in /etc/asterisk/phpagi.conf

vi /etc/asterisk/phpagi.conf`
[phpagi]
debug=false                              ; enable debuging
error_handler=false                      ; use internal error handler
After this, set verbose to false in /var/lib/asterisk/agi-bin/phpagi-fastagi.php

vi /var/lib/asterisk/agi-bin/phpagi-fastagi.php`
#replace
$fastagi->verbose(print_r($fastagi, true)); #or comment it
#with
$fastagi->verbose(print_r($fastagi, false));
