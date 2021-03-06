.. -*- rst -*-
.. _Buildslaves:

.. bb:cfg:: slaves

Buildslaves
-----------

The :bb:cfg:`slaves` configuration key specifies a list of known buildslaves. In
the common case, each buildslave is defined by an instance of the
:class:`BuildSlave` class.  It represents a standard, manually started machine
that will try to connect to the buildbot master as a slave.  Contrast these
with the "on-demand" latent buildslaves, such as the Amazon Web Service Elastic
Compute Cloud latent buildslave discussed below.

The :class:`BuildSlave` class is instantiated with two values: (``slavename``,
``slavepassword``). These are the same two values that need to be provided to the
buildslave administrator when they create the buildslave.

The slavenames must be unique, of course. The password exists to
prevent evildoers from interfering with the buildbot by inserting
their own (broken) buildslaves into the system and thus displacing the
real ones.

Buildslaves with an unrecognized slavename or a non-matching password
will be rejected when they attempt to connect, and a message
describing the problem will be put in the log file (see :ref:`Logfiles`). ::

    from buildbot.buildslave import BuildSlave
    c['slaves'] = [BuildSlave('bot-solaris', 'solarispasswd'),
                   BuildSlave('bot-bsd', 'bsdpasswd'),
                  ]

:class:`BuildSlave` objects can also be created with an optional
``properties`` argument, a dictionary specifying properties that
will be available to any builds performed on this slave.  For example::

    from buildbot.buildslave import BuildSlave
    c['slaves'] = [BuildSlave('bot-solaris', 'solarispasswd',
                        properties=@{'os':'solaris'@}),
                  ]

The :class:`BuildSlave` constructor can also take an optional
``max_builds`` parameter to limit the number of builds that it
will execute simultaneously::

    from buildbot.buildslave import BuildSlave
    c['slaves'] = [BuildSlave("bot-linux", "linuxpassword", max_builds=2)]

.. _Master-slave-TCP-Keepalive:

Master-slave TCP keepalive
~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, the buildmaster sends a simple, non-blocking message to each slave
every hour.  These keepalives ensure that traffic is flowing over the
underlying TCP connection, allowing the system's network stack to detect any
problems before a build is started.

The interval can be modified by specifying the interval in seconds using the
``keepalive_interval`` parameter of BuildSlave::

    c['slaves'] = [BuildSlave('bot-linux', 'linuxpasswd',
                              keepalive_interval=3600),
                  ]

The interval can be set to ``None`` to disable this functionality
altogether.

.. _When-Buildslaves-Go-Missing:

When Buildslaves Go Missing
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes, the buildslaves go away. One very common reason for this is
when the buildslave process is started once (manually) and left
running, but then later the machine reboots and the process is not
automatically restarted.

If you'd like to have the administrator of the buildslave (or other
people) be notified by email when the buildslave has been missing for
too long, just add the ``notify_on_missing=`` argument to the
:class:`BuildSlave` definition::

    c['slaves'] = [BuildSlave('bot-solaris', 'solarispasswd',
                              notify_on_missing="bob@@example.com"),
                  ]

By default, this will send email when the buildslave has been
disconnected for more than one hour. Only one email per
connection-loss event will be sent. To change the timeout, use
``missing_timeout=`` and give it a number of seconds (the default
is 3600).

You can have the buildmaster send email to multiple recipients: just
provide a list of addresses instead of a single one::

    c['slaves'] = [BuildSlave('bot-solaris', 'solarispasswd',
                              notify_on_missing=["bob@@example.com",
                                                 "alice@@example.org"],
                              missing_timeout=300, # notify after 5 minutes
                              ),
                  ]

The email sent this way will use a :class:`MailNotifier` (:ref:`MailNotifier`)
status target, if one is configured. This provides a way for you to
control the *from* address of the email, as well as the relayhost
(aka *smarthost*) to use as an SMTP server. If no :class:`MailNotifier` is
configured on this buildmaster, the buildslave-missing emails will be
sent using a default configuration.

Note that if you want to have a :class:`MailNotifier` for buildslave-missing
emails but not for regular build emails, just create one with
``builders=[]``, as follows::

    from buildbot.status import mail
    m = mail.MailNotifier(fromaddr="buildbot@@localhost", builders=[],
                          relayhost="smtp.example.org")
    c['status'].append(m)
    c['slaves'] = [BuildSlave('bot-solaris', 'solarispasswd',
                              notify_on_missing="bob@@example.com"),
                  ]

.. _Latent-Buildslaves:
                  
Latent Buildslaves
~~~~~~~~~~~~~~~~~~

The standard buildbot model has slaves started manually.  The previous section
described how to configure the master for this approach.

Another approach is to let the buildbot master start slaves when builds are
ready, on-demand.  Thanks to services such as Amazon Web Services' Elastic
Compute Cloud ("AWS EC2"), this is relatively easy to set up, and can be
very useful for some situations.

The buildslaves that are started on-demand are called "latent" buildslaves.
As of this writing, buildbot ships with an abstract base class for building
latent buildslaves, and a concrete implementation for AWS EC2.

.. _AWS-EC2:

.. index::
   AWS EC2
   BuildSlaves; AWS EC2

Amazon Web Services Elastic Compute Cloud ("AWS EC2")
+++++++++++++++++++++++++++++++++++++++++++++++++++++

`AWS EC2 <http://aws.amazon.com/ec2/>`_ is a web service that allows you to
start virtual machines in an Amazon data center. Please see their website for
details, incuding costs. Using the AWS EC2 latent buildslaves involves getting
an EC2 account with AWS and setting up payment; customizing one or more EC2
machine images ("AMIs") on your desired operating system(s) and publishing
them (privately if needed); and configuring the buildbot master to know how to
start your customized images for "substantiating" your latent slaves.

Get an AWS EC2 Account
######################

To start off, to use the AWS EC2 latent buildslave, you need to get an AWS
developer account and sign up for EC2. These instructions may help you get
started:

  1. Go to http://aws.amazon.com/ and click to "Sign Up Now" for an AWS account.

  2. Once you are logged into your account, you need to sign up for EC2.
     Instructions for how to do this have changed over time because Amazon changes
     their website, so the best advice is to hunt for it. After signing up for EC2,
     it may say it wants you to upload an x.509 cert. You will need this to create
     images (see below) but it is not technically necessary for the buildbot master
     configuration.

  3. You must enter a valid credit card before you will be able to use EC2. Do that
     under 'Payment Method'.

  4. Make sure you're signed up for EC2 by going to 'Your Account'->'Account
     Activity' and verifying EC2 is listed.

Create an AMI
#############

Now you need to create an AMI and configure the master.  You may need to
run through this cycle a few times to get it working, but these instructions
should get you started.

Creating an AMI is out of the scope of this document.  The
`EC2 Getting Started Guide <http://docs.amazonwebservices.com/AWSEC2/latest/GettingStartedGuide/>`_
is a good resource for this task.  Here are a few additional hints.

  * When an instance of the image starts, it needs to automatically start a
    buildbot slave that connects to your master (to create a buildbot slave,
    :ref:`Creating-a-buildslave`; to make a daemon,
    :ref:`Launching-the-daemons`).

  * You may want to make an instance of the buildbot slave, configure it as a
    standard buildslave in the master (i.e., not as a latent slave), and test and
    debug it that way before you turn it into an AMI and convert to a latent
    slave in the master.

Configure the Master with an EC2LatentBuildSlave
################################################

Now let's assume you have an AMI that should work with the
EC2LatentBuildSlave.  It's now time to set up your buildbot master
configuration.

You will need some information from your AWS account: the `Access Key Id` and
the `Secret Access Key`.  If you've built the AMI yourself, you probably
already are familiar with these values.  If you have not, and someone has
given you access to an AMI, these hints may help you find the necessary
values:

  * While logged into your AWS account, find the "Access Identifiers" link (either
    on the left, or via "Your Account" -> "Access Identifiers".

  * On the page, you'll see alphanumeric values for "Your Access Key Id:" and
    "Your Secret Access Key:". Make a note of these. Later on, we'll call the
    first one your ``identifier`` and the second one your ``secret_identifier``\.

When creating an EC2LatentBuildSlave in the buildbot master configuration,
the first three arguments are required.  The name and password are the first
two arguments, and work the same as with normal buildslaves.  The next
argument specifies the type of the EC2 virtual machine (available options as
of this writing include ``m1.small``, ``m1.large``, ``m1.xlarge``, ``c1.medium``,
and ``c1.xlarge``; see the EC2 documentation for descriptions of these
machines).

Here is the simplest example of configuring an EC2 latent buildslave. It
specifies all necessary remaining values explicitly in the instantiation. ::

    from buildbot.ec2buildslave import EC2LatentBuildSlave
    c['slaves'] = [EC2LatentBuildSlave('bot1', 'sekrit', 'm1.large',
                                       ami='ami-12345',
                                       identifier='publickey',
                                       secret_identifier='privatekey'
                                       )]

The ``ami`` argument specifies the AMI that the master should start.  The
``identifier`` argument specifies the AWS `Access Key Id`, and the
``secret_identifier`` specifies the AWS `Secret Access Key.` Both the AMI and
the account information can be specified in alternate ways.

.. note:: Whoever has your ``identifier`` and ``secret_identifier`` values can request
   AWS work charged to your account, so these values need to be carefully
   protected. Another way to specify these access keys is to put them in a
   separate file. You can then make the access privileges stricter for this
   separate file, and potentially let more people read your main configuration
   file.

By default, you can make an :file:`.ec2` directory in the home folder of the user
running the buildbot master. In that directory, create a file called :file:`aws_id`.
The first line of that file should be your access key id; the second line
should be your secret access key id. Then you can instantiate the build slave
as follows. ::

    from buildbot.ec2buildslave import EC2LatentBuildSlave
    c['slaves'] = [EC2LatentBuildSlave('bot1', 'sekrit', 'm1.large',
                                       ami='ami-12345')]

If you want to put the key information in another file, use the
``aws_id_file_path`` initialization argument.

Previous examples used a particular AMI.  If the Buildbot master will be
deployed in a process-controlled environment, it may be convenient to
specify the AMI more flexibly.  Rather than specifying an individual AMI,
specify one or two AMI filters.

In all cases, the AMI that sorts last by its location (the S3 bucket and
manifest name) will be preferred.

One available filter is to specify the acceptable AMI owners, by AWS account
number (the 12 digit number, usually rendered in AWS with hyphens like
"1234-5678-9012", should be entered as in integer). ::

    from buildbot.ec2buildslave import EC2LatentBuildSlave
    bot1 = EC2LatentBuildSlave('bot1', 'sekrit', 'm1.large',
                               valid_ami_owners=[11111111111,
                                                 22222222222],
                               identifier='publickey',
                               secret_identifier='privatekey'
                               )

The other available filter is to provide a regular expression string that
will be matched against each AMI's location (the S3 bucket and manifest name). ::

    from buildbot.ec2buildslave import EC2LatentBuildSlave
    bot1 = EC2LatentBuildSlave(
        'bot1', 'sekrit', 'm1.large',
        valid_ami_location_regex=r'buildbot\-.*/image.manifest.xml',
        identifier='publickey', secret_identifier='privatekey')

The regular expression can specify a group, which will be preferred for the
sorting.  Only the first group is used; subsequent groups are ignored. ::

    from buildbot.ec2buildslave import EC2LatentBuildSlave
    bot1 = EC2LatentBuildSlave(
        'bot1', 'sekrit', 'm1.large',
        valid_ami_location_regex=r'buildbot\-.*\-(.*)/image.manifest.xml',
        identifier='publickey', secret_identifier='privatekey')

If the group can be cast to an integer, it will be.  This allows 10 to sort
after 1, for instance. ::

    from buildbot.ec2buildslave import EC2LatentBuildSlave
    bot1 = EC2LatentBuildSlave(
        'bot1', 'sekrit', 'm1.large',
        valid_ami_location_regex=r'buildbot\-.*\-(\d+)/image.manifest.xml',
        identifier='publickey', secret_identifier='privatekey')

In addition to using the password as a handshake between the master and the
slave, you may want to use a firewall to assert that only machines from a
specific IP can connect as slaves.  This is possible with AWS EC2 by using
the Elastic IP feature.  To configure, generate a Elastic IP in AWS, and then
specify it in your configuration using the ``elastic_ip`` argument. ::

    from buildbot.ec2buildslave import EC2LatentBuildSlave
    c['slaves'] = [EC2LatentBuildSlave('bot1', 'sekrit', 'm1.large',
                                       'ami-12345',
                                       identifier='publickey',
                                       secret_identifier='privatekey',
                                       elastic_ip='208.77.188.166'
                                       )]

The :class:`EC2LatentBuildSlave` supports all other configuration from the standard
:class:`BuildSlave`.  The ``missing_timeout`` and ``notify_on_missing`` specify how long
to wait for an EC2 instance to attach before considering the attempt to have
failed, and email addresses to alert, respectively.  ``missing_timeout``
defaults to 20 minutes.

The ``build_wait_timeout`` allows you to specify how long an :class:`EC2LatentBuildSlave`
should wait after a build for another build before it shuts down the EC2
instance.  It defaults to 10 minutes.

``keypair_name`` and ``security_name`` allow you to specify different names for
these AWS EC2 values.  They both default to ``latent_buildbot_slave``.

.. _BuildSlave-Libvirt:

.. index::
   libvirt
   BuildSlaves; libvirt

Libvirt
+++++++

`libvirt <http://www.libvirt.org/>`_ is a virtualization API for interacting
with the virtualization capabilities of recent versions of Linux and other OSes.
It is LGPL and comes with a stable C API, and python bindings.

This means we know have an API which when tied to buildbot allows us to have slaves
that run under Xen, QEMU, KVM, LXC, OpenVZ, User Mode Linux, VirtualBox and VMWare.

The libvirt code in Buildbot was developed against libvirt 0.7.5 on Ubuntu Lucid. It
is used with KVM to test python code on Karmic VM's, but obviously isn't limited to that.
Each build is run on a new VM, images are temporary and thrown away after each build.

Setting up libvirt
##################

We won't show you how to set up libvirt as it is quite different on each platform,
but there are a few things you should keep in mind.

  * If you are running on Ubuntu, your master should run Lucid. Libvirt and apparmor are
    buggy on Karmic.

  * If you are using the system libvirt, your buildbot master user will need to be in the
    libvirtd group.

  * If you are using KVM, your buildbot master user will need to be in the KVM group.

  * You need to think carefully about your virtual network *first*. Will NAT be enough?
    What IP will my VM's need to connect to for connecting to the master?

Configuring your base image
###########################

You need to create a base image for your builds that has everything needed to build
your software. You need to configure the base image with a buildbot slave that is configured
to connect to the master on boot.

Because this image may need updating a lot, we strongly suggest scripting its creation.

If you want to have multiple slaves using the same base image it can be annoying to duplicate
the image just to change the buildbot credentials. One option is to use libvirt's DHCP
server to allocate an identity to the slave: DHCP sets a hostname, and the slave takes its
identity from that.

Doing all this is really beyond the scope of the manual, but there is a :file:`vmbuilder` script
and a :file:`network.xml` file to create such a DHCP server in
:file:`contrib/` (:ref:`Contrib-Scripts`)
that should get you started:

.. code-block:: bash

    sudo apt-get install ubuntu-vm-builder
    sudo contrib/libvirt/vmbuilder

Should create an :file:`ubuntu/` folder with a suitable image in it.

.. code-block:: none

    virsh net-define contrib/libvirt/network.xml
    virsh net-start buildbot-network

Should set up a KVM compatible libvirt network for your buildbot VM's to run on.

Configuring your Master
#######################

If you want to add a simple on demand VM to your setup, you only need the following. We
set the username to ``minion1``, the password to ``sekrit``. The base image is called ``base_image``
and a copy of it will be made for the duration of the VM's life. That copy will be thrown
away every time a build is complete. ::

    from buildbot.libvirtbuildslave import LibVirtBuildSlave
    c['slaves'] = [LibVirtBuildSlave('minion1', 'sekrit',
                                       '/home/buildbot/images/minion1', '/home/buildbot/images/base_image')]

You can use virt-manager to define ``minion1`` with the correct hardware. If you don't, buildbot
won't be able to find a VM to start.

:class:`LibVirtBuildSlave` accepts the following arguments:

``name``
    Both a buildbot username and the name of the virtual machine

``password``
    A password for the buildbot to login to the master with

``hd_image``
    The path to a libvirt disk image, normally in qcow2 format when using KVM.

``base_image``
    If given a base image, buildbot will clone it every time it starts a VM.
    This means you always have a clean environment to do your build in.

``xml``
    If a VM isn't predefined in virt-manager, then you can instead provide XML
    like that used with ``virsh define``. The VM will be created
    automatically when needed, and destroyed when not needed any longer.

.. _Dangers-with-Latent-Buildslaves:

Dangers with Latent Buildslaves
+++++++++++++++++++++++++++++++

Any latent build slave that interacts with a for-fee service, such as the
EC2LatentBuildSlave, brings significant risks. As already identified, the
configuration will need access to account information that, if obtained by a
criminal, can be used to charge services to your account. Also, bugs in the
buildbot software may lead to unnecessary charges. In particular, if the
master neglects to shut down an instance for some reason, a virtual machine
may be running unnecessarily, charging against your account. Manual and/or
automatic (e.g. nagios with a plugin using a library like boto)
double-checking may be appropriate.

A comparatively trivial note is that currently if two instances try to attach
to the same latent buildslave, it is likely that the system will become
confused.  This should not occur, unless, for instance, you configure a normal
build slave to connect with the authentication of a latent buildbot.  If this
situation does occurs, stop all attached instances and restart the master.

.. _Writing-New-Latent-Buildslaves:

Writing New Latent Buildslaves
++++++++++++++++++++++++++++++

Writing a new latent buildslave should only require subclassing
:class:`buildbot.buildslave.AbstractLatentBuildSlave` and implementing
:meth:`start_instance` and :meth:`stop_instance`. ::

    def start_instance(self):
        # responsible for starting instance that will try to connect with this
        # master. Should return deferred. Problems should use an errback. The
        # callback value can be None, or can be an iterable of short strings to
        # include in the "substantiate success" status message, such as
        # identifying the instance that started.
        raise NotImplementedError
    
    def stop_instance(self, fast=False):
        # responsible for shutting down instance. Return a deferred. If `fast`,
        # we're trying to shut the master down, so callback as soon as is safe.
        # Callback value is ignored.
        raise NotImplementedError

See :class:`buildbot.ec2buildslave.EC2LatentBuildSlave` for an example, or see the
test example :class:`buildbot.test_slaves.FakeLatentBuildSlave`.

