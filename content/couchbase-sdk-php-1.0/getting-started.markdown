# Getting Started

We will use PHP and access the Couchbase server as part of this exercise; the
Couchbase Server can be part of a cluster or it can be stand-alone. This
exercise will show you how to:

 * Download, install and configure the Couchbase PHP SDK

 * How to use Couchbase and PHP to generate a general page access counter and
   last-accessed date indicator script

This section assumes that you have the following items set up for your
development environment:

 * PHP 5 and above installed. For more information, see [PHP
   Manual](http://php.net/manual/en/install.php),

 * Web server installed and configured to serve PHP pages,

 * Couchbase Server 1.8 or above installed, and available from your development
   environment. See [Couchbase Downloads](http://www.couchbase.com/download)

This section also assumes you have downloaded and set up a compatible version of
Couchbase Server and have at least one instance of Couchbase Server and one data
bucket established. If you need to set up these items, you can do with the
Couchbase Administrative Console, or Couchbase Command-Line Interface (CLI), or
the Couchbase REST-API. For information and instructions, see:

 * [Using the Couchbase Web
   Console](http://www.couchbase.com/docs/couchbase-manual-1.8/couchbase-introduction.html),
   for information on using the Couchbase Administrative Console,

 * [Couchbase
   CLI](http://www.couchbase.com/docs/couchbase-manual-1.8/couchbase-admin-web-console.html),
   for the command line interface,

 * [Couchbase REST
   API](http://www.couchbase.com/docs/couchbase-manual-1.8/couchbase-admin-restapi.html),
   for creating and managing Couchbase resources.

The TCP/IP port allocation on Windows by default includes a restricted number of
ports available for client communication. For more information on this issue,
including information on how to adjust the configuration and increase the
available ports, see [MSDN: Avoiding TCP/IP Port
Exhaustion](http://msdn.microsoft.com/en-us/library/aa560610(v=bts.20).aspx).

After you have your Couchbase Server set up and you have installed the Couchbase
SDK, you can compile and run the following basic program.

<a id="download"></a>

## Downloading and Installation

Follow these steps to install and set up the PHP SDK:

 1. Download the SDK for your system [from the SDK
    page](http://www.couchbase.com/develop/php/current).

 1. Unpack the archive containing the SDK:

     ```
     shell> tar xzf php-ext-couchbase-$system-$arch.tar.gz
     ```

    The resulting directory includes a file couchbase.so which is the PHP extension
    for the SDK. Note the path to this file.

 1. Find the `php.ini` file for your PHP interpreter. Try:

     ```
     shell> phpi -i | grep ini
     ```

    The output provided will have configuration file that is loaded for PHP, for
    instance:

     ```
     Loaded Configuration File => /private/etc/php.ini
     ```

 1. Open the php.ini file and add the path to your couchbase.so file. For example:

     ```
     extension=/path/to/couchbase.so
     ```

<a id="installation-verification"></a>

### Verify the SDK Install

To verify you have set up the PHP SDK correctly follow these steps:

 1. Create a new file named test.php andplace the following test code it:

     ```
     <?php
         $cb = new Couchbase("127.0.0.1:8091", "username", "password", "default");
         $cb->set("a", 1);
         var_dump($cb->get("a"));
     ?>
     ```

 1. Save the new script as test.php.

 1. Start your Couchbase Server. Note the ip address and port for Couchbase Server
    and any username and password.

 1. If needed, update the IP address, username and password in test.php with your
    Couchbase Server information.

 1. Run your test script:

     ```
     shell> php test.php
     ```

    Couchbase Server returns 1.

<a id="accessing-php"></a>

## Accessing Couchbase from PHP

To demonstrate how to use Couchbase PHP SDK with Couchbase Server we will built
a web page counter. The counter will record the last access date for a web page.
we'll implement those features in PHP using the Couchbase library.

To store data in Couchbase Server, you provide information in a way that is
similar to a PHP Array. You can store a value with a key, then retrieve the
value by the key. In this exercise, we will store and retrieve integer and
string data.

In the first part of our PHP script, we create a connection with Couchbase
Server:


```
<?php
#Create the Couchbase object
$cb_obj = new Couchbase("127.0.0.1:8091", "user", "pass", "default");
```

In the next section of our script, we find out the name of the script currently
running on the web server. Typically the name would be for the counter PHP
script itself:


```
#determine the name of the script currently running
$script_name=$_SERVER["SCRIPT_NAME"];
```

In this case we set the `$script_name` variable to be the script name that is
stored in a PHP environment variable, `SCRIPT_NAME`. Then we try to retrieve an
existing record for the key `$script_name` from Couchbase Server. This will tell
us if the key already exists or not and wheter we need to create a new record
with the key:


```
#if the script name doesn't exist as a Couchbase key then add

$script_access_count=$cb_obj->get($script_name);

if($cb_obj->getResultCode() == COUCHBASE_KEY_ENOENT){
    #the add will fail if it has already been added
   $cb_obj->add($script_name,0);
}
```

Then we use one of the Couchbase SDKs increment methods to add one to the page
access count. We provide the key, which is the `$script_name`, to increase the
counter by one. We also output the access count in a string:


```
#increment the integer associated with the script name
$access_count = $cb_obj->increment($script_name);

#print the current access count
echo "this page ($script_name) accessed $access_count times<br>";
```

In this final part of our access counter script, we retrieve the last date and
time that the script had been accessed. Again, we use the key as a parameter to
get the value from Couchbase Server:


```
#retrieve the last access date/time of the script.
#the key name is is the script name prefixed with DATE::
$last_access_date=$cb_obj->get("DATE::" . $script_name);

#handle the case where this is the first access to the script
#and that key doesn't yet exist
if($last_access_date == NULL){
     $last_access_date = "never";
}
echo "this page last accessed: " . $last_access_date;

#save the current access date/time in a script
$cb_obj->set("DATE::" . $script_name,date("F j, Y, g:i:s a "));
?>
```

You can save this sample can be saved into a PHP file in a directory on your web
server, such as test.php

When you load test.php page you'll see something like this on the first load:


```
this page (/test.php) accessed 1 times
this page last accessed: never
```

And like this on subsequent loads, with the access count incrementing and the
date/time increasing:


```
this page (/test.php) accessed 2 times
this page last accessed: June 1, 2011, 10:06:04 am
```

Reload the script a few times in rapid succession and you'll see a spike in
traffic in the Couchbase Web UI. So, what exactly is our script doing to achieve
this?

The first line of the script creates a Couchbase object ( `$cb_obj` ). The
parameters define a Couchbase server hostname and port as well as user
credentials and the default bucket name to the server pool that we'll be
accessing.

If you are running Couchbase on a single node, you specify its hostname and
port. If you are running a multi-node cluster, you only need to point to a
single node in a cluster as the PHP extension will transparently determine any
cluster topology and route requests to the right servers as well as react on
topolgy changes.


```
$cb_obj = new Couchbase("localhost", "user", "pass", "default");
```

We then get the `"SCRIPT_NAME"` value from the `$_SERVER` Array, which tells us
in which script this PHP code is currently running.


```
$script_name=$_SERVER["SCRIPT_NAME"];
```

Now we connect to Couchbase to retrieve the integer access count associated with
this script using the script name (determined above) as the key.


```
$script_access_count=$cb_obj->get($script_name);
```

If there is no such key in Couchbase (because the page has not previously been
accessed), the request returns a `NULL` value, and the result code of the call
to the get method is the Couchbase constant `COUCHBASE_KEY_ENOENT` (resource not
found). In this case we use the `add` method to set the value to zero. If,
between the failed `get` and `add` method calls, the key has already been added
by another process, the `add` will fail and we will not overwrite the added
value. This sort of attention to concurrency issues is overkill for the current
application, as a missed counting of a single script access is unlikely to be
critical; however, it illustrates how to write Couchbase code to avoid
unintentionally overwriting keys:


```
if($cb_obj->getResultCode() == COUCHBASE_KEY_ENOENT){
       $cb_obj->add($script_name,0);
}
```

The key associated with the script name count is then incremented by one for the
current script access, and that incremented value is returned and printed out.
The increment method is atomic, consisting of both an increment and retrieval of
the resultant value. If the Couchbase server receives two such requests, they
will be queued for action in the order they were received and return the correct
count to each requesting process.


```
$access_count = $cb_obj->increment($script_name);
echo "this page ($script_name) accessed $access_count times<br>";
```

To retrieve and update the last access date of the script we're doing something
similar to what we did for the script access count, however, instead of adding
and incrementing an integer, we're adding and updating a string.

As a key for the date any given script was accessed, we've prepended the string
`"DATE::"` to the beginning of the script name; thus the key for the last access
date of our test.php script is `"DATE::test.php"`. We first try to get the date
of last access and assign it to the `$last_access_date` variable. If this fails
(the `get` method returns a Boolean `NULL` value and the subsequent
`getResultCode` call returns `COUCHBASE_KEY_ENOENT` ) that means it hasn't yet
been set, so the script has not previously been accessed. In that case we'll set
`$last_access_date` to "never". We then print out the value of
`$last_access_date`.


```
$last_access_date=$cb_obj->get("DATE::" . $script_name);
 if($cb_obj->getResultCode() == COUCHBASE_KEY_ENOENT){
    $last_access_date = "never";
}
echo "this page last accessed: " . $last_access_date;
```

We now set the access date for this access of the script for retrieval on the
next access.


```
$cb_obj->set("DATE::" . $script_name,date("F j, Y, g:i:s a "));
```

And that's the end of the script. There is no method in the Couchbase library to
close the connection to the server, however it will automatically be closed at
the end of script execution.

<a id="conclusion"></a>

## Conclusion

This has been a brief outline of some basic features of the Couchbase API in
PHP. We've used the `add`, `get`, `set`, `increment` and `getResultCode` methods
of the Couchbase class to store and retrieve integers and strings. There are a
number of other methods that allow for scalable development in a multiprocess
and multi-server environment.

<a id="tutorial"></a>
