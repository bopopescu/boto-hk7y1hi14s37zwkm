.. _ec2_tut:

=======================================
An Introduction to boto's EC2 interface
=======================================

This tutorial focuses on the boto interface to the Elastic Compute Cloud
from Amazon Web Services.  This tutorial assumes that you have already
downloaded and installed boto.

Creating a Connection
---------------------

The first step in accessing EC2 is to create a connection to the service.
The recommended way of doing this in boto is::

    >>> import boto.ec2
    >>> conn = boto.ec2.connect_to_region("us-east-1",
    ...    aws_access_key_id='<aws access key>',
    ...    aws_secret_access_key='<aws secret key>')

At this point the variable ``conn`` will point to an EC2Connection object.  In
this example, the AWS access key and AWS secret key are passed in to the method
explicitly.  Alternatively, you can set the boto config environment variables
and then simply specify which region you want as follows::

    >>> conn = boto.ec2.connect_to_region("us-east-1")

In either case, conn will point to an EC2Connection object which we will
use throughout the remainder of this tutorial.

Launching Instances
------------------
Possibly, the most important and common task you'll use EC2 for is to launch, stop and terminate instances.
In its most primitive form, you can launch an instance as follows::

    >>> conn.run_instances('<ami-image-id>')

This will launch an instance in the specified region with the default parameters.

Now, let's say that you already have a key pair, want a specific type of instance, and
you have your security group all setup. In this case we can use the keyword arguments to accomplish that::

    >>> conn.run_instances('<ami-image-id>',key_name='myKey', instance_type='c1.xlarge', security_groups=['public-facing'])

The main caveat with the above call is that it is possible to request an instance type that is not compatible with the 
provided AMI (for example, the instance was created for a 64-bit instance and you choose a m1.small instance_type).
For more details on the plethora of possible keyword parameters, be sure to check out boto's EC2 API documentation_.

.. _documentation: http://boto.cloudhackers.com/en/latest/ref/ec2.html

Stopping Instances
------------------
Once you have your instances up and running, you might wish to shut them down if they're not in use. Please note that this will only de-allocate
virtual hardware resources (as well as instance store drives), but won't destroy your EBS volumes -- this means you'll pay nominal provisioned EBS storage fees 
even if your instance is stopped. To do this, you can do so as follows::

    >>> conn.stop_instances(instance_ids=['instance-id-1','instance-id-2', ...])

This will request a 'graceful' stop of each of the specified instances. If you wish to request the equivalent of unplugging your instance(s),
simply add force=True keyword argument to the call above. Please note that stop instance is not allowed with Spot instances.

Terminating Instances
---------------------
Once you are completely done with your instance and wish to surrender both virtual hardware, root EBS volume and all other underlying components 
you can request instance termination. To do so you can use the call bellow::

    >>> conn.terminate_instances(instance_ids=['instance-id-1','instance-id-2', ...])

Please use with care since once you request termination for an instance there is not turning back.

Checking What Instances Are Running
-----------------------------------
You can also get information on your currently running instances::

    >>> reservations = conn.get_all_instances()
    >>> reservations
    [Reservation:r-00000000]

A reservation corresponds to a command to start instances. You can see what
instances are associated with a reservation::

    >>> instances = reservations[0].instances
    >>> instances
    [Instance:i-00000000]

An instance object allows you get more meta-data available about the instance::

    >>> inst = instances[0]
    >>> inst.instance_type
    u'c1.xlarge'
    >>> inst.placement
    u'us-east-1a'

In this case, we can see that our instance is a c1.xlarge instance in the
`us-east-1a` availability zone.

=================================
Using Elastic Block Storage (EBS)
=================================


EBS Basics
----------

EBS can be used by EC2 instances for permanent storage. Note that EBS volumes
must be in the same availability zone as the EC2 instance you wish to attach it
to.

To actually create a volume you will need to specify a few details. The
following example will create a 50GB EBS in one of the `us-east-1a` availability
zones::

   >>> vol = conn.create_volume(50, "us-east-1a")
   >>> vol
   Volume:vol-00000000

You can check that the volume is now ready and available::

   >>> curr_vol = conn.get_all_volumes([vol.id])[0]
   >>> curr_vol.status
   u'available'
   >>> curr_vol.zone
   u'us-east-1a'

We can now attach this volume to the EC2 instance we created earlier, making it
available as a new device::

   >>> conn.attach_volume (vol.id, inst.id, "/dev/sdx")
   u'attaching'

You will now have a new volume attached to your instance. Note that with some
Linux kernels, `/dev/sdx` may get translated to `/dev/xvdx`. This device can
now be used as a normal block device within Linux.

Working With Snapshots
----------------------

Snapshots allow you to make point-in-time snapshots of an EBS volume for future
recovery. Snapshots allow you to create incremental backups, and can also be
used to instantiate multiple new volumes. Snapshots can also be used to move
EBS volumes across availability zones or making backups to S3.

Creating a snapshot is easy::

   >>> snapshot = conn.create_snapshot(vol.id, 'My snapshot')
   >>> snapshot
   Snapshot:snap-00000000

Once you have a snapshot, you can create a new volume from it. Volumes are
created lazily from snapshots, which means you can start using such a volume
straight away::

   >>> new_vol = snapshot.create_volume('us-east-1a')
   >>> conn.attach_volume (new_vol.id, inst.id, "/dev/sdy")
   u'attaching'

If you no longer need a snapshot, you can also easily delete it::

   >>> conn.delete_snapshot(snapshot.id)
   True


