# Test definitions for freeIPA

Currently included are Jenkins job definitions for Fedora, in
jenkins-job-builder YAML format.

Contributions for other CI systems, platforms and setups are welcome.

To install jenkins-job-builder (by the OpenStack project).
See http://ci.openstack.org/jenkins-job-builder/index.html for docs.

## Sample usage

    install:
        source my-virtualenv/bin/activate
        pip install jenkins-job-builder
    test:
        mkdir -p /tmp/jenkins-jobs
        jenkins-jobs test ./jenkins-job-builder/ -o /tmp/jenkins-jobs
        (examine /tmp/jenkins-jobs)
    configuration file for talking to Jenkins (default: /etc/jenkins_jobs/jenkins_jobs.ini):
        [jenkins]
        user=USERNAME
        password=PASSWORD
        url=JENKINS_URL
    project configuration file:
        Copy 'project.yaml.example' to <my-project>.yaml and customize it to your needs
    commit to Jenkins:
        jenkins-jobs update -r ./jenkins-job-builder/:PATH/TO/<my-project>.yaml

Please change the recipient in the "mail-on-fail" publisher.


## Configuration

The configuration in this repository provides job descriptions.
Other required Jenkins configuration is listed below.

These environment variables need to be set on build nodes. They can be set globally via jenkins
Variables used in out-of-tree and webui jobs. TRUSTPASS and TRUSTDOMAIN are environment dependent.

    FREEIPACI_PASSWORD=12345678
    FREEIPACI_TRUSTPASS
    FREEIPACI_TRUSTDOMAIN

These variables are deployment dependent

    FREEIPACI_PREPARE_HOSTS_URL=<URL, see Controller scripts below>
    FREEIPACI_SHUTDOWN_HOSTS_URL=<URL, see Controller scripts below>

The HOSTS_* variables are used in the controller scripts used by the projects CI system.

    FREEIPACI_HOSTS_DOMAINS
    FREEIPACI_HOSTS_STATIC

# Network requirements

The current code in the repository expects a specific networking setup.
The resources used in the integration testing and DNS used for it must be
configured as follows.

A resource used as integration test host has a hostname starting by a prefix
'vm-'. For example host 192.168.42.15 on subnet 192.168.42.0/24 and domain testing.example.com
used for the testing would use hostname `vm-015.testing.example.com.` resolvable by the
used DNS server.

This DNS server is also required to delegate the domain `dom-015.testing.example.com`
to the hostname `vm-015.testing.example.com`.
If done for the whole IP range, any resource on the network can become a domain controller
and DNS discovery will work (as long as the delegation chain works).

If any other schema for domain delegation is present, the repository code needs to be amended.


## Jenkins configuration

### Required Jenkins nodes

- master:
    should have Publican, pip and sloccount installed
- freeipa-builder - builds RPMs from source repository
- freeipa-runner - runs acceptance tests against local IPA instance
- freeipa-webui - similar to acceptance tests, runs webui tests
- freeipa-controller  (see Controller scripts below)

The type of node is indicated as a label in jenkins configuration. The JJB uses following labels.

- ipa-ci-{os}-builder
- ipa-ci-{os}-controller{controller_suffix}
- ipa-ci-{os}-webui
- ipa-ci-{os}-xmlrpc-runner
- master

### Installed Jenkins plugins

The list of required jenkins plugins is in
the file [required-jenkins-plugins.md](/required-jenkins-plugins.md).

## Controller scripts

Jenkins only allows a single slave for each test. For multi-machine testing,
we use a "controller" machine that manages nodes.
The controller uses two scripts, which it downloads from URLs specified
by envoronment variables (see above).

prepare-hosts:
    Receives a test config JSON template on stdin,
    where each host entry only has 'name' and 'role' specified.
    The script brings up the mentioned machines and fills
    in 'external_hostname' and 'ip'.
    Also, fills in 'dns_forwarder', 'ntp_server', and either
    'root_ssh_key_filename' or 'root_password', and may change
    any other configuration.
    It handles the assigment of static test nodes. These are
    specified by using special role names, such as TRUST_IPA for IPA domain
    that AD is configured to establish trusts with, or trust_master for the
    actual IPA master. These fake values are substituted by the script.
    Output can be YAML or JSON.

shutdown-hosts
    Recieves a IPA test config file which the prepare-hosts script produced,
    and brings down all machines specified in it.
