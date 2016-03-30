Aptly
========

Aptly is a brilliant software for creating both your own Debian repositories and official repository mirrors with git-like snapshot management. This role helps you to install aptly and setup your own repository with nginx as web server.
Role allows aptly to be configured to publish multiple Debian apt repositories. Each published repository could be a snapshot of another repository mirror or a merge of multiple snapshots from different mirrors.

Notes
=====

1. This role is tested on Ubuntu 14.04 (Trusty) and aptly 0.9.5 only (the latest as of 29 Mar 2016). Patches for other distributions and versions are welcome.


Requirements
------------

You need a GPG public/private key pair to sign your repository.

Use the following shell commands to obtain your own key pair:

    $ sudo apt-get install gnupg
    $ gpg --gen-key

       Please select what kind of key you want:
        (1) RSA and RSA (default)
        (2) DSA and Elgamal
        (3) DSA (sign only)
        (4) RSA (sign only)
        Your selection? 3

        ... answer questions about your name, key expiration period, etc. ...

        gpg: checking the trustdb
        gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
        gpg: depth: 0  valid:   4  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 4u
        gpg: next trustdb check due at 2019-05-23

        pub   2048D/61B1BA69 2014-05-24
        ~~~~~~~~~~~~^^^^^^^^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
              Key fingerprint = 6C50 B46A 3D75 0C14 041B  6C99 FBBA 0275 61B1 BA69
              uid                  Ivan Ivanov (repository key) <ivan@example.com>

Once you have generated the key pair, you should export it to the corresponding files. Replace key ID with yours (you can find it in the end of previous command's output. In this example, the key ID is 61B1BA69)

    $ mkdir -p secrets/aptly  # (assuming you are in the directory where inventory.ini and playbooks are located. Also see 'Optional Variables' section)
    $ gpg --export-secret-keys --armor 61B1BA69 > secrets/aptly/private.key
    $ gpg --export --armor 61B1BA69 > secrets/aptly/public.key


Role Variables
--------------
 
### Secrets
 A GPG key is used to sign the published apt lists. A generated GPG public and private keypair should exist in the 
 preconfigured directories. 
```
    aptly_secret_key_path: files/secret
    aptly_secret_key_file: private.key
    aptly_public_key_file: public.key
    aptly_secret_key_id: AAAFFFF
```

### Mirror-less deb file repositories
 This role can publish an apt repositories that contain `deb` files uploaded by host. The files need to be uploaded and to 
 which publish endpoint they should belong must be configured.
```
    # Define a list of repositories (these will be initialized empty)
    aptly_repositories:
      -
        name: custom-deb
        comment: Direct download deb files
        distribution: trusty
        component: main
        architectures: amd64
    # Path to folder which contains deb files that need to be uploaded to aptly server
    aptly_upload_path: files/deb 
    # Assign the uploaded debs to specific repositories via fileglobs.
    aptly_upload_definitions: 
        -
          name: custom-deb
          fileglob: "*.deb"
```

### Snapshot debian mirrors
 Define a list of mirrors that needs to be cloned, updated, and snapshot each playbook run. Make sure to have the repo
 keys downloaded on the host machine for uploading.
```
# List of mirrors (and snapshots)
aptly_mirrors:
  -
    name: trusty
    archive_url: http://us-east-1.ec2.archive.ubuntu.com/ubuntu/
    distribution: trusty
    component: main
    architectures: amd64
    key: files/ubuntu-archive.asc
  -
    name: trusty (universe)
    archive_url: http://us-east-1.ec2.archive.ubuntu.com/ubuntu/
    distribution: trusty
    component: universe
    architectures: amd64
    key: files/ubuntu-archive.asc 
  # ...
# Create a snapshot merging multiple mirrors
aptly_snapshots_merged:
  -
    name: merged-all
    snapshots:
      - mongodb (10gen)
      - postgresql
      - newrelic
      - trusty
      - trusty (universe)
      - trusty-updates
      - trusty-updates (universe)
      - trusty-security
      - trusty-security (universe)
# 
```

### Publishing apt repositories
 Publish endpoints must be defined via a configuration variable. Each endpoint will be based on a snapshot or a repo
 with the same name.
```
aptly_snapshot_published:
  - test
  - stage
  - prod
```
 All apt clients should insert the following entry per published end point in `/etc/apt/sources.list`.
```
deb http://aptly-server/endpoint-name/ trusty main
```

### Promoting repositories
 Published snapshots may be promoted each run of the playbook. This allows migrating snapshot of one repository to
 another. ( test -> stage -> prod ). The promotion rules must be predefined. 
```
aptly_snapshot_promote_definitions:
  - # 'merged-all' will be the new 'test'
    name: test
    base: merged-all
  - # original 'test' will be the new 'stage'
    name: stage
    base: test
  - # original 'stage' will be the new 'prod'
    name: prod
    base: stage
```
 * NOTE: Promotions should be explicitly specified each playbook run as they are destructive tasks.
```
# Only promote 'test'
$ ansible-playbook -i inventory/live setup.yml \
    --extra-vars='{"aptly_snapshot_promote": ["test"]}' 
# Promote 'test' and 'dev'
$ ansible-playbook -i inventory/live setup.yml  \
    --extra-vars='{"aptly_snapshot_promote": ["test", "dev"]}' 
```

### Update saved mirror snapshots
 To update remote sources and re-create mirrors, snapshots, and publish endpoints, run playbook with 
 `aptly_perform_update` set to `true`.
```
# Updates all mirrors, recreates snapshots and promotes the updated merged snapshot to 'test'
# publish end-point. ('--tags=update' is optional)
ansible-playbook-debugger -i inventory/live setup.yml \
 --extra-vars='{"aptly_snapshot_promote": ["test"], "aptly_perform_update": true}' --tags=update
```

### Other Vars
 * `aptly_nginx_enabled` - Install and configure nginx
 * `aptly_s3_enabled` - Configure publishing to s3; requires extra configuration.
 
 Check `defaults/main.yml` for a full list of configurable variables.
 
Dependencies
------------

Gnupg for key generation. None if you already have one.


Example Playbook
-------------------------

    -
        name: Install aptly
        gather_facts: no  # optional
        hosts:
            - repositories
        vars:
            aptly_secret_key_id: <GNUPG KEY ID>
            aptly_repositories:
                -
                    # With optional parameters
                    name: yourcompany-dev
                    comment: Developent packages
                    distribution: trusty
                    component: main
                    architectures: amd64,i386  # This is the default value. I recommend not to change it.
                -
                    # Using default settings
                    name: yourcompany-testing
                -
                    name: yourcompany-prod
        roles:
            - aptly


How to setup clients
--------------------

```shell
apt-key add secrets/aptly/public.key
echo 'deb http://$server_name/$repository_name trusty main' > /etc/apt/sources.list.d/$repository_name.list
```

How to upload new package
-------------------------
Here is an idea:

```shell
scp $package_file $server_name:/tmp/
ssh $server_name "sudo -u aptly -H aptly repo add $repository_name /tmp/$package_file"
ssh $server_name "sudo -u aptly -H aptly publish update main $repository_name"
```

AFAIK at this moment aptly doesn't support uploading signed .deb packages (.changes file)


License
-------

BSD

Author Information
------------------

http://github.com/alexey-sveshnikov (Modified by @rajiteh http://github.com/rajiteh)
