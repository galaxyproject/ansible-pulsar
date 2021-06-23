Pulsar
======

An [Ansible][ansible] role for installing and managing [Pulsar][pulsar]
servers.

[ansible]: http://www.ansible.com/
[pulsar]: https://github.com/galaxyproject/pulsar/

Requirements
------------

This role has the same dependencies as the `git` module, namely, [git][git]. In addition, [Python virtualenv][venv] is
required (as is [pip][pip], but pip will automatically installed with virtualenv). These can easily be installed via a
pre-task in the same play as this role:

```yaml
- hosts: pulsarservers
  pre_tasks:
    - name: Install dependencies
      package:
        name:
          - git
          - virtualenv
          - python3
      become: true
      when: ansible_os_family == 'Debian'
    - name: Install dependencies
      package:
        name:
          - git
          - python36-virtualenv
          - python3
        state: present
      become: true
      when: ansible_os_family == 'RedHat'
  roles:
    - galaxyproject.pulsar
```

If your `virtualenv` executable is not on `$PATH`, you can specify its location with the `pip_virtualenv_command`
variable.

[git]: http://git-scm.com/
[venv]: http://virtualenv.readthedocs.org/
[pip]: http://pip.readthedocs.org/

Breaking Changes
----------------

As of 1.0.0, installation directly from source using `git clone` is no longer supported. However, `pulsar_package_name`
and `pulsar_package_version` can be used to install using pip's `git+https` method.

Role Variables
--------------

### Required variables ###

- `pulsar_root`: Filesystem path under which Pulsar virtualenv, configs, etc. will be installed. Formerly
  `pulsar_server_dir`

### Optional variables ###

You can control various things about where you get Pulsar from, what version you use, and where its configuration files
will be placed:

- `pulsar_yaml_config`: a YAML dictionary whose contents will be used to create Pulsar's app.yml
- `pulsar_venv_dir` (default: `<pulsar_root>/venv`): The role will create a [virtualenv][virtualenv] from which Pulsar
  will run, this controls where the virtualenv will be placed.
- `pulsar_config_dir` (default: `<pulsar_root>/config`): Directory that will be used for Pulsar configuration files.
- `pulsar_optional_dependencies` (default: None): List of optional dependency modules to install. Whether or not you
  need these depends on what features you are enabling.
- `pulsar_dependencies_dir` (default: `<pulsar_root>/deps`): The value of the `tool_dependency_dir` option in the Pulsar
  configuration, this directory will be created by the role if it does not exist.
- `pulsar_persistence_dir` (default: `<pulsar_root>/files/persisted_data`): The value of the `persistence_directory`
  option in the Pulsar configuration, this directory will be created by the role if it does not exist.
- `pulsar_staging_dir` (default: `<pulsar_root>/files/staging`): The value of the `staging_directory` option in the
  Pulsar configuration, this directory will be created by the role if it does not exist.
- `pulsar_job_metrics_plugins`: Contents of the `job_metrics_conf.yml` Pulsar configuration file, see Galaxy
  documentation for details on this file and its syntax.

For details about the Pulsar application configuration options, consult the [Pulsar documentation][pulsardocs] and
`app.yml.sample`.

**User management and privilege separation**

- `pulsar_separate_privileges` (default: `no`): Enable privilege separation mode.
- `pulsar_user` (default: user running ansible): The name of the system user under which pulsar runs. (or,
  `{name: pulsar, shell: /bin/bash}`)
- `pulsar_privsep_user` (default: `root`): The name of the system user that owns the pulsar code, config files, and
  virtualenv (and dependencies therein).
- `pulsar_group`: Common group between the pulsar user and privilege separation user. If set directories containing
  potentially sensitive information such as the pulsar config file will be created group- but not world-readable.
  Otherwise, directories are created world-readable.
- `pulsar_create_user` (default: `no`): Create the pulsar user. Running as a dedicated user is a best practice, but most
  production pulsar instances submitting jobs to a cluster will manage users in a directory service (e.g.  LDAP). This
  option is useful for standalone servers. Requires superuser privileges.

**systemd support**

By default, this role does not configure Pulsar to start and stop automatically, but if the system you are installing
Pulsar on to uses [systemd][systemd] and you have root privileges on that system (required to install the systemd
service unit that controls Pulsar; Pulsar does not run as root), the role can automatically start Pulsar and restart it
as needed.

- `pulsar_systemd` (default: `false`): Set to `true` to enable configuration and management of systemd by this role.
- `pulsar_systemd_enabled` (default: `true`): Set to `false` to configure Pulsar in systemd but disable automatic
  starting on boot.
- `pulsar_systemd_service_name` (default: `pulsar`): systemd service name for the Pulsar service. If you want to run
  multiple Pulsar servers on the same system, you can use this variable to prevent collision of the systemd services and
  service unit file names.
- `pulsar_systemd_state` (default: `started`): State for Pulsar service to be in after running the playbook. Valid
  values are Ansible service module states. Can be useful for testing.
- `pulsar_systemd_memory_limit` (default: `6`): Size (in GB) of memory limit. If Pulsar uses more than this limit, the
  system will kill (and attempt to restart) it.
- `pulsar_systemd_runner` (default: `paste`): Whether to start Pulsar with a web server and, if so, what web server. If
  using Pulsar in AMQP "message mode", set this to `webless`. Valid values are `paste`, `webless`, `uwsgi`
- `pulsar_systemd_environment`: A list of `VAR=value` strings to be added as `Environment=VAR=val` to the systemd
  service unit

**Web server configuration**

Additional options from Pulsar's `server.ini` are configurable via the following variables (these options are explained
in the [Pulsar documentation][pulsardocs] and `server.ini.sample`):

- `pulsar_host` (default: `localhost`)
- `pulsar_port` (default: `8913`)
- `pulsar_uwsgi_socket` (default: if unset, uWSGI will be configured to listen for HTTP requests on `pulsar_host` port
  `pulsar_port`): If set, uWSGI will listen for uWSGI protocol connections on this socket.
- `pulsar_uwsgi_options` (default: empty hash): Hash (dictionary) of additional uWSGI options to place in the `[uwsgi]`
  section of `server.ini`

**Legacy options**

- `pulsar_drmaa_library_path`: The value of `$DRMAA_LIBRARY_PATH` set in Pulsar's `local_env.sh`. If using systemd, set
  `DRMAA_LIBRARY_PATH=/path/to/libdrmaa.so` in `pulsar_systemd_environment` instead.

All Pulsar application configuration (e.g. managers) should be done in `pulsar_yaml_config` except for the options
described above. Support for configuring via ini files has been dropped from this role.

[systemd]: https://www.freedesktop.org/wiki/Software/systemd/

### pulsar_optional_dependencies ###

Currently, the list of optional dependencies is:

```yaml
pulsar_optional_dependencies:
  - pyOpenSSL
  # For remote transfers initiated on the Pulsar end rather than the Galaxy end
  - pycurl
  # uwsgi used for more robust deployment than paste
  - uwsgi
  # drmaa required if connecting to an external DRM using it.
  - drmaa
  # kombu needed if using a message queue
  - kombu
  # requests and poster using Pulsar remote staging and pycurl is unavailable
  - requests
  - poster
  # psutil and pylockfile are optional dependencies but can make Pulsar
  # more robust in small ways.
  - psutil
```

Many of these dependencies have their own dependencies. A nice future
enhancement to this role would be to install the dependencies' dependencies via
the system package manager if desired.

[pulsardocs]: http://pulsar.readthedocs.org/

Dependencies
------------

None

Example Playbook
----------------

Install Pulsar on your local system with all the default options:

```yaml
- hosts: localhost
  connection: local
  vars:
    pulsar_root: /home/nate/pulsar
  roles:
    - role: galaxyproject.pulsar
```

Install Pulsar with directory separation and also install Galaxy for remote (Pulsar) metadata:

```yaml
- hosts: pulsarservers
  vars:
    pulsar_root: /opt/pulsar
    pulsar_config_dir: /etc/opt/pulsar
    pulsar_persistence_dir: /var/opt/pulsar/persisted_data
    pulsar_dependencies_dir: /var/opt/pulsar/deps
    pulsar_staging_dir: /hpc/pulsar/staging
    pulsar_optional_dependencies:
      - pyOpenSSL
      - pycurl
      - uwsgi
      - drmaa
      - kombu
      - requests
      - poster
      - psutil
    galaxy_server_dir: /opt/galaxy/server
    galaxy_config_dir: /etc/opt/galaxy
    galaxy_config_files:
      - name: files/galaxy/config/datatypes_conf.xml
        dest: "{{ galaxy_config_dir }}/datatypes_conf.xml"
  roles:
    - role: galaxyproject.pulsar
    # Install with:
    #   % ansible-galaxy install natefoo.postgresql_objects
    - role: galaxyproject.galaxy
      galaxy_manage_mutable_setup: no
      galaxy_manage_database: no
```
          
Install Pulsar into a CentOS 7 host with directory and privilege separation, systemd service configuration, webless
mode, communication via a message queue, multiple named job managers, and submission to an HTCondor cluster:

```yaml
- hosts: pulsarservers
  vars:
    pulsar_root: /opt/pulsar
    pulsar_config_dir: /etc/opt/pulsar
    pulsar_persistence_dir: /var/opt/pulsar/persisted_data
    pulsar_dependencies_dir: /var/opt/pulsar/deps
    pulsar_staging_dir: /hpc/pulsar/staging
    pulsar_optional_dependencies:
      - pycurl
      - kombu
      - psutil
    pulsar_systemd: true
    pulsar_systemd_runner: webless
    pulsar_separate_privileges: yes
    pulsar_privsep_user: centos
    pulsar_yaml_config:
      conda_auto_init: true
      conda_auto_install: true
      assign_ids: none
      message_queue_url: "amqp://user:pass@amqp.example.org:5671//vhost?ssl=1"
      min_polling_interval: 0.5
      managers:
        production:
          submit_universe: vanilla
          type: queued_condor
        test:
          submit_universe: vanilla
          type: queued_condor
  pre_tasks:
    - name: Install dependencies
      become: yes
      package:
        state: latest
        name:
          - git
          - python-virtualenv
          - python3
          - curl
          - libcurl-devel
  roles:
    - role: galaxyproject.pulsar
```

License
-------

[Academic Free License ("AFL") v. 3.0][afl]

[afl]: http://opensource.org/licenses/AFL-3.0

Author Information
------------------

[View contributors on Github](https://github.com/galaxyproject/ansible-pulsar/graphs/contributors)
