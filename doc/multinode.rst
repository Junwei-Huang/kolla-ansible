.. _multinode:

=============================
Multinode Deployment of Kolla
=============================

.. _deploy_a_registry:

Deploy a registry
=================

A Docker registry is a locally hosted registry that replaces the need to pull
from the Docker Hub to get images. Kolla can function with or without a local
registry, however for a multinode deployment some type of registry is
mandatory.  Only one registry must be deployed, although HA features exist for
registry services.

The Docker registry prior to version 2.3 has extremely bad performance because
all container data is pushed for every image rather than taking advantage of
Docker layering to optimize push operations. For more information reference
`pokey registry <https://github.com/docker/docker/issues/14018>`__.

Edit the ``/etc/kolla/globals.yml`` and add the following where 192.168.1.100
is the IP address of the machine and 5000 is the port where the registry is
currently running:

::

    docker_registry = 192.168.1.100:5000

The Kolla community recommends using registry 2.3 or later. To deploy registry
with version 2.3 or later, do the following:

::

    cd kolla
    tools/start-registry

The Docker registry can be configured as a pull through cache to proxy the
official Kolla images hosted in Docker Hub. In order to configure the local
registry as a pull through cache, in the host machine set the environment
variable ``REGISTRY_PROXY_REMOTEURL`` to the URL for the repository on
Docker Hub.

::

    export REGISTRY_PROXY_REMOTEURL=https://registry-1.docker.io

.. note::

    Pushing to a registry configured as a pull-through cache is unsupported.
    For more information, Reference the `Docker Documentation
    <https://docs.docker.com/registry/configuration/>`__.

.. _configure_docker_all_nodes:

Configure Docker on all nodes
=============================

.. note:: As the subtitle for this section implies, these steps should be
          applied to all nodes, not just the deployment node.

After starting the registry, it is necessary to instruct Docker that it will
be communicating with an insecure registry. To enable insecure registry
communication on CentOS, modify the ``/etc/sysconfig/docker`` file to contain
the following where 192.168.1.100 is the IP address of the machine where the
registry is currently running:

::

    # CentOS
    INSECURE_REGISTRY="--insecure-registry 192.168.1.100:5000"

For Ubuntu, check whether its using upstart or systemd.

::

    # stat /proc/1/exe
    File: '/proc/1/exe' -> '/lib/systemd/systemd'

Edit ``/etc/default/docker`` and add:

::

    # Ubuntu
    DOCKER_OPTS="--insecure-registry 192.168.1.100:5000"

If Ubuntu is using systemd, additional settings needs to be configured.
Copy Docker's systemd unit file to ``/etc/systemd/system/`` directory:

::

    cp /lib/systemd/system/docker.service /etc/systemd/system/docker.service

Next, modify ``/etc/systemd/system/docker.service``, add ``environmentFile``
variable and add ``$DOCKER_OPTS`` to the end of ExecStart in ``[Service]``
section:

::

    # CentOS
    [Service]
    MountFlags=shared
    EnvironmentFile=/etc/sysconfig/docker
    ExecStart=
    ExecStart=/usr/bin/docker daemon $INSECURE_REGISTRY

    # Ubuntu
    [Service]
    MountFlags=shared
    EnvironmentFile=-/etc/default/docker
    ExecStart=
    ExecStart=/usr/bin/docker daemon -H fd:// $DOCKER_OPTS

Restart Docker by executing the following commands:

::

    # CentOS or Ubuntu with systemd
    systemctl daemon-reload
    systemctl restart docker

    # Ubuntu with upstart or sysvinit
    sudo service docker restart

.. _edit-inventory:

Edit the Inventory File
=======================

The ansible inventory file contains all the information needed to determine
what services will land on which hosts. Edit the inventory file in the
Kolla-Ansible directory ``ansible/inventory/multinode``. If Kolla-Ansible
was installed with pip, it can be found in ``/usr/share/kolla-ansible``.

Add the IP addresses or hostnames to a group and the services associated with
that group will land on that host. IP addresses or hostnames must be added to
the groups control, network, compute, monitoring and storage. Also, define
additional behavioral inventory parameters such as ``ansible_ssh_user``,
``ansible_become`` and ``ansible_private_key_file/ansible_ssh_pass`` which
controls how ansible interacts with remote hosts.

.. note::

   Ansible uses SSH to connect the deployment host and target hosts. For more
   information about SSH authentication please reference
   `Ansible documentation <http://docs.ansible.com/ansible/intro_inventory.html>`__.

::

   # These initial groups are the only groups required to be modified. The
   # additional groups are for more control of the environment.
   [control]
   # These hostname must be resolvable from your deployment host
   control01      ansible_ssh_user=<ssh-username> ansible_become=True ansible_private_key_file=<path/to/private-key-file>
   192.168.122.24 ansible_ssh_user=<ssh-username> ansible_become=True ansible_private_key_file=<path/to/private-key-file>

.. note::

   Additional inventory parameters might be required according to your
   environment setup. Reference `Ansible Documentation
   <http://docs.ansible.com/ansible/intro_inventory.html>`__ for more
   information.


For more advanced roles, the operator can edit which services will be
associated in with each group. Keep in mind that some services have to be
grouped together and changing these around can break your deployment:

::

   [kibana:children]
   control

   [elasticsearch:children]
   control

   [haproxy:children]
   network

Deploying Kolla
===============

.. note::

    If there are multiple keepalived clusters running within the same layer 2
    network, edit the file ``/etc/kolla/globals.yml`` and specify a
    ``keepalived_virtual_router_id``. The ``keepalived_virtual_router_id`` should
    be unique and belong to the range 0 to 255.

First, check that the deployment targets are in a state where Kolla may deploy
to them:

::

    kolla-ansible prechecks -i <path/to/multinode/inventory/file>

Run the deployment:

::

    kolla-ansible deploy -i <path/to/multinode/inventory/file>

.. _Building Container Images: https://docs.openstack.org/kolla/latest/image-building.html
