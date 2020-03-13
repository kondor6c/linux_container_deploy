container-deploy
****************
This simple concept, deploying a container monitored by systemd. Uses a lot of templating around a map/dict (container_chalk) to accmoplish the task. Yes, kubernetes is better, and there are many features that this role can't do, but some organization or individuals, might (or should) have a container running a simple service like ``task-warrior`` it listens on a TCP port and is often used personally, but could be used at a small startup to track todo's. This could facilitate that while not adding a lot of dependencies or staff/knowledge requirements.
**This is just ansible pushing out systemd to monitor the container running.**

systemd can monitor a path, and execute another unit file if that path has changed. This is very cool and could improve deployments in non-kubernetes environments. 

This was designed with docker in mind, but could handle others (designed for the unforeseen future). 

Inspirations include the podman generate feature which creates a systemd unit file for the container.

Current Potential pitfalls
==========================
In this new design, there were compromises that were made in order to ship this quickly or the complexity to support it might introduce. Here are a few of them:
All of these are areas for improvement, and the author is attempting to be critical of his own design

 - sidecars would be nice, but it might be difficult to manage right now
 - non-tcp or non-listening containers. If you have a container that is periodically calling a service, like a database, and it is not listening for connections... You'll need to expose a port. This sucks, sorry. But the port is a form of "uid". UDP might work, but has not been tested. Let's face it the vast majority of our services are TCP (often HTTP).

Requirements
============
none

Role Variables
=============

container_chalk dict/map
^^^^^^^^^^^^^^^^^^^^^^^^

+-----------------------+-----------------------------+------------------------+
| variable              | type and (status)           | function               |
+=======================+=============================+========================+
| container_port        | string (required)           | this is the port that the container is listening on    |
+-----------------------+-----------------------------+------------------------+
| name                  | string (required) | the primary name of this container service (not what will be pulled), this will be used on the system |
+-----------------------+-----------------------------+------------------------+
| alias_name            | string (not implemented) | the alternative vanitry name, think of this as a CNAME that points to "name" |
+-----------------------+-----------------------------+------------------------+
| host_port                    | integer (required)       | this port is used as a unique identifier AND to open a port on the host to the container   |
+-----------------------+-----------------------------+------------------------+
| project                    | string (required)       | the path of the container, if the image is from docker hub, use 'library' |
+-----------------------+-----------------------------+------------------------+
| registry_host         | string (required) | the host that has the container image, if docker hub, use 'docker.io'   |
+-----------------------+-----------------------------+------------------------+
| container_image_name         | string (required)      | the name of the image |
+-----------------------+-----------------------------+------------------------+
| container_block_device  | string (optional) | the block device to use for overlay2 |
+-----------------------+-----------------------------+------------------------+
| container_engine  | string (required) | install docker or podman |
+-----------------------+-----------------------------+------------------------+
| container_config_dir  | string (required) | root tree location where environment files, label files and other config files |
+-----------------------+-----------------------------+------------------------+
| container_image_version_tag  | string (optional:latest) | the version of the container to pull     |
+-----------------------+-----------------------------+------------------------+
| option_map | map (required)  | allows for more options, if no options are desired, don't set anything inside the map, but it must be set          |
+-----------------------+-----------------------------+------------------------+
| option_map[addional_optoins] | map (optional) | literal strings to append to ``docker run``           |
+-----------------------+-----------------------------+------------------------+
| option_map[publish] | list[string] (optional) | allows for more options          |
+-----------------------+-----------------------------+------------------------+
| option_map[volumes] | list[string] (optional) | allows for more options          |
+-----------------------+-----------------------------+------------------------+
| option_map[cap_add] | list (optional) | allows for more options          |
+-----------------------+-----------------------------+------------------------+
| option_map[cap_del] | key/value (optional) | allows for more options          |
+-----------------------+-----------------------------+------------------------+
| option_map[] | key/value (optional) | allows for more options          |
+-----------------------+-----------------------------+------------------------+
| option_map[any_option] | key/value (optional) | *any CLI option* and its corresponding value. See more below |
+-----------------------+-----------------------------+------------------------+

example execution
-----------------
Let's take a look at an example and some yaml that would deploy the example. If we took the mysql image on docker hub, and deployed it, the systemd unit file would have this as the ExecStart, and deployed it would look like the output:

.. code:: bash
  
   docker run --/usr/bin/docker run --env-file=/etc/devops/mysql-example/33100.container_environment --label-file=/etc/devops/mysql-example/33100.label \
          --publish=33100:9022 --mount type=tmpfs,tmpfs-size=32M,destination=/secrets \
          --name mysql-example-33100  docker.io/library/mysql:latest

   docker ps

   CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                          NAMES
   a8d93263c493        mysql:latest        "docker-entrypoint.s…"   7 days ago          Up 7 days           3306/tcp, 33060/tcp, 0.0.0.0:33100->9022/tcp   mysql-example--33100

The reason you are seeing 3306 and 33060, is that they are in the Dockerfile as "EXPOSE".  The host_port is the unique identifier, if there is another run of the systemd unit (whether its a deployment or a restart), and the container is still around (in ``docker container ls``), the systemd unit will fail and so will this ansible role. Ensuring that new deployments of the same $service--$host_port are *stopped* or better yet, *removed* is a good idea and might be implemented later, for now you can add ``option_map['additional_options']: "--rm"`` and that will remove the container when stopped. Keep in mind, if you are trying to troubleshoot the container, if --rm is passed the container will go away, making troubleshooting more difficult.

.. warning::
    Do not pass "--detach" or "-d" to options_map['additional_options']. This will cause docker or podman to fork and systemd won't monitor it (unless you modify the unit file) this is Not Advised. You have been warned

This is the corresponding YAML that would deploy the example above.

.. code:: yaml
 
    - name: mysql-example
      project: library
      host_port: 33100
      container_port: 9022
      registry_host: docker.io
      container_image_name: mysql
      container_image_version_tag: latest
      option_map:
        env:
          - MYSQL_RANDOM_ROOT_PASSWORD=true

This will be passed literally as key=value on the command line (systemd unit), this allows for expandability in the future. If docker or podman or rkt or any other container runner implements ``--cow-say=moooooo!`` which is totally not in the foreseeable future, this role can deploy that, by adding it like this ``option_map['cow-say']: mooooo!``. If the option is a single dash or has no equals, add it to ``option_map['additional_options']`` this is list of literal strings that will be added.
Even further... Let's say you are in production and you have a container running successfully with a ``CMD ["/usr/sbin/sshd","-f","/etc/ssh/sshd_config"]`` but you need it to run ``ssh-keygen`` first. You **could** deploy a fix file (a bash script to generate the key and run sshd) to the container config path, and pass an option like ``--entrypoint=/fix.sh`` to the option_map. This also allows for troubleshooting and bypassing a bad entrypoint.


Dependencies
------------


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

.. code:: yaml


    - vars:
        container_chalk:
          - name: rclone-fuse
            host_port: 33100
            project: tynor88
            container_image_version_tag: dev
            registry_host: docker.io
            container_image_name: rclone-mount
            container_port: 9022
            readiness_external_cmd: curl __self__/health
            option_map:
              cap-add:
                - sys_time
              cap-del:
                - setuid
                - setgid
              memory: 1024m
              block-weight: 500
              cpus: 1
              network: sidecar-net
              volume:
                - /mnt/containers/sshfs:/srv
              publish:
                - 5555:33501
            additional_options:
              - " -l fun_label=sure "
          
          - name: sshfs
            host_port: 33900
            registry_host: quay.io
            container_port: 9022
            project: nexway
            container_image_name: sshfs-server
            container_image_version_tag: latest
          - name: rclone-waf
            host_port: 33500
            container_port: 443
            project: zecure
            option_map:
              volume:
                - /mnt/containers/zecure:/srv
              rm: true
              publish:
                - 33501:8080
            container_image_name: shadowd
            registry_host: docker.io
            container_image_version_tag: latest
            
          - name: jenkins
            host_port: 33300
            project: jenkins
            container_port: 8080
            option_map:
              volume:
                - /mnt/containers/jenkins:/srv
            container_image_name: jenkins
            readiness_external_cmd: curl __self__/health
            registry_host: docker.io
            container_image_version_tag: lts
          - name: jenkins-waf
            host_port: 33400
            project: scollazo
            container_port: 443
            option_map:
              volume:
                - /mnt/containers/jenkins-waf:/srv
              rm: true
              publish:
                - 33401:8080
            container_image_version_tag: latest
            container_image_name: naxsi-waf-with-ui
            registry_host: docker.io
            additional_options:
              - " --env BACKEND_IP=192.168.122.37 "
    - hosts: workers
      roles:
         - { role: container-deploy, container_block_device: "/dev/vdb", container_engine: "podman" }

License
-------

BSD

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
