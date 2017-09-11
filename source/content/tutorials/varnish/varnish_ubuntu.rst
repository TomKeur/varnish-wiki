.. _varnish_ubuntu:

Installing and configuring Varnish
==================================

The following text discusses how to configure your web server to use Varnish.
Note that the installation is different for *systemv* and *systemd*.
The following guide is for systemd as many linux distributions are now adapting to
the systemd init system.

Audience:
.........

This reference has been prepared for web developers to get them started with
their Varnish installation and configuration.

Step 1 : Installing Varnish on Ubuntu/UNIX:
-------------------------------------------

It is recommended that you install the Varnish package from its repository.

- Start by grabbing the repository

- Add the repository to the source list and save

.. literalinclude:: /content/examples/varnish_install_repo
	:language: bash

- Run update and install

.. literalinclude:: /content/examples/update_install_varnish
	:language: bash
For more information visit the `packagecloud`_ page.

.. _`packagecloud`: https://packagecloud.io/varnishcache/varnish41

Step 2: Configure Varnish
--------------------------

Varnish comes with two configuration files:

**One with the starter parameter:**

.. code-block:: bash

	/etc/default/varnish

This file contains all the starter parameters.

**The other is the default VCL file:**

For systemd, the VCL file is directed in a different manner.
It will be located in:

.. code-block:: bash

	/etc/systemd/system/varnish.service

which will point to:

.. code-block:: bash

	/etc/varnish/default.vcl

This default.vcl contains the *default* policies that the *user includes*.
It also tells Varnish where to find the web content.
However, there is a `builtin.vcl`_ that is always appended to the VCL you define/specify
in this default.vcl.

1. Modify **Varnish** config file

- Open ``/etc/default/varnish`` in a text editor
- You will see a code like the one below

.. literalinclude:: /content/examples/default_varnish_1.vcl
	:language: VCL

**Description:**

 -T : refers to which port manages this.

 -f : refers to the other configuration file containing all the default policies.
 If you plan to change the name of the default policy file, 
 be sure to come here and change the default.vcl to the correct name.

 -S : refers to the file containing private information, such as passwords, etc. 
 also known as the shared-secret file.

 -s : refers to the space Varnish Cache is allocated. 256m”
 is decided based on the current server's RAM of 1GB.

- Set the Varnish listen port to 80

- Replace the ``-f`` line with ``-b 95.85.10.242:8080`` as shown below

.. literalinclude:: /content/examples/default_varnish_2.vcl
	:language: VCL

These are all the configuration changes required in this file.

2. Copy the default file named **varnish.service**

.. code-block:: bash

	cp /lib/systemd/system/varnish.service /etc/systemd/system/


- Edit ``/etc/systemd/system/varnish.service``
 
- Locate the line containing port 80 and change it to 8080

.. code-block:: bash
 
 	ExecStart=/usr/sbin/varnishd -a :80 -T localhost:6082 -f /etc/varnish/default.vcl
 	-S /etc/varnish/secret -s malloc,256m

Note that this file points to the /etc/varnish/default.vcl file.

3. Now modify the **default.vcl** file

This file contains configuration that points to the content. This is by default
set to serve at 8080 and points to host as localhost as shown below.

To minimally configure Varnish:

- Back up default.vcl

.. code-block:: bash

	cp /etc/varnish/default.vcl /etc/varnish/default.vcl.bak

- Open ``/etc/varnish/default.vcl`` in a text editor.

- Locate the following piece of code:

.. literalinclude:: /content/examples/default_vcl.vcl
	:language: VCL

The value of .host is localhost by default. It should be replaced with the fully
qualified host name or IP address (typically a web server) and listen port of the
Varnish backend or origin server; that is, the server providing the content
Varnish will accelerate.

The value of .port should be replaced with the web server's listening port, for example
8080 as shown below.

.. literalinclude:: /content/examples/default_vcl_2.vcl
	:language: VCL

It is recommended that if changes are made to these files, they should be copied
and renamed, because when Varnish updates, it will replace any changes made
with the new default.vcl and Varnish files.

Varnish is now serving the client at port 80 and listening to the backend at
port 8080.

This is when you can add ready-made VCL templates or recommended plugins for
your web application (WordPress, Drupal, Magento2)

But before you add any new code you must understand VCL.

Step 3: Configure Apache2 to work with Varnish
----------------------------------------------

Configure your web server to listen on a port other than the default port 80 because
Varnish responds directly to incoming HTTP requests from the client on this port.

Varnish will communicate on a different port with your backend web servers. In
the example above, it is port 8080.

In the sections that follow, we use port 8080 as an example, as shown above.
If you have more than one backend, you can put them on another port, such as port
8081, 8082, etc.

To change the Apache listen port:

 - Open ``/etc/apache2/ports.conf`` in a text editor

 - Locate the listen directive
 
 - Change the value of the listen port to 8080 (you can use any available listen port)
 
 - Save your changes to ports.conf and exit the text editor
 
 - Also edit ``/etc/apache2/sites-available/000-default.conf``
 
 - Change the VirtualHost port to 8080:
 
.. code-block:: bash

	<VirtualHost 127.0.0.1:8080>


Setting up multiple backends (skip this section if you have one backend)
------------------------------------------------------------------------

If you have more than one backend server, you need to add all these backends into
the default.vcl file as backends and also define how and when each of them
should be accessed.

Here is a simple example:

.. literalinclude:: /content/examples/vclex_multiple_backend.vcl
	:language: VCL


Step 4: Restart
---------------

It is always required that you restart all services once changes are made in configuration files.

.. literalinclude:: /content/examples/snippet2_restart_systemv
	:language: VCL


Step 5: Testing
---------------

Run

.. code-block:: bash

	http -p Hh localhost

You can also use varnishtest to test your backend as shown below.

.. literalinclude:: /content/examples/vtc/vtc_apacheBackend.vtc
	:language: VCL

Replace your backend ip-address and port number.


Step 6: Troubleshooting
-----------------------

If Varnish fails to start, try running it from the command line as follows:

.. code-block:: bash

	varnishd ~d ~f /etc/varnish/default.vcl

This should display any error messages.


Step 7: The management interface
--------------------------------

Varnish has a command line interface (CLI) to control any Varnish instance. It can be used to:

- Reload VCL without restarting

- Start/stop cache process

- Change configuration parameters without restarting

- View up-to-date documentation for parameters, etc.

- Implement a list of management commands in the `varnishadm` (`varnishadm` establishes a connection to the Varnish deamon `varnishd`)


.. _`builtin\.vcl`: https://github.com/varnishcache/varnish-cache/blob/master/bin/varnishd/builtin.vcl
