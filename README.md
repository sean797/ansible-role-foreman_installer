# foreman_installer

Role to interact with foreman-installer

## Requirements

N/A

## Role Variables

```yaml
vars:
  foreman_installer:
    foreman_installer_pkg:          # foreman installer package. You probably want either "foreman-installer" or "foreman-installer-katello".
    foreman_installer_verbose:      # Run the installe with -v option
    foreman_installer_scenario:     # Scenario. Required
    installer_scenarios_answers:    # Dict of custom answers that for your scenario. This is merged with your scenarios default answers in the {{ scenario }}-answers.yml file.
    foreman_installer_options: []   # Array of extra options to pass to whenever the installer is ran
    katello_ca:                     # String containing the custom CA cert. Katello Only.
    katello_cert:                   # String containing the custom cert. Katello Only.
    katello_key:                    # String containing the custom key. Katello Only.
    katello_csr:                    # String containing the custom csr. Katello Only.
    katello_certs_dir:              # Directory to store the certificates
    update_certs: False             # Set to True to force Certificate Update.
    update_certs_tar: False         # Set to True to force new Proxy Certificates tar to be generated & applied.

    # Advanced Options
    foreman_installer_patches:      # Array of Dicts allowing patches against the installer files. See defaults/main.yml for an example.
    foreman_encryption_key:         # Encryption key that is put into /etc/foreman/encryption_key.rb. Must be the same across a Foreman cluster.
    katello_cluster_group:          # Name of the inventory group will all Katello servers in. Requires http://projects.theforeman.org/issues/20021
```

## Example Playbook

### Basic Foreman scenario:

```yaml
    - hosts:
      - foreman.example.com
      roles:
        - role: foreman_installer
          foreman_installer_scenario: foreman
          installer_scenarios_answers:
            foreman:
              admin_password: changeme
```

### Katello scenario with custom certificates:

```yaml
    - hosts:
      - katello.example.com
    var_files:
      - group_vars/vault_certs.yml
    roles:
       - role: foreman_installer
         foreman_installer_pkg: foreman-installer-katello
         foreman_installer_scenario: katello
         installer_scenarios_answers:
           foreman:
             admin_password: changeme
         katello_ca: "{{ vault_katello_ca }}"
         katello_cert: "{{ vault_katello_cert }}"
         katello_key: "{{ vault_katello_key }}"
         katello_csr: "{{ vault_katello_csr }}"
```

### Katello Proxy scenario with supplied certificates tar:

```yaml
    - hosts:
      - foreman-proxy.example.com
    roles:
       - role: foreman_installer
         foreman_installer_pkg: foreman-proxy-content
         foreman_installer_scenario: foreman-proxy-content
         installer_scenarios_answers:
           foreman_proxy_content:
             certs_tar: /root/foreman-proxy.example.com-certs.tar #This must already be on-disk
             pulp_oauth_secret: <outputted when generating the certifcates tar>
           foreman_proxy:
             oauth_consumer_key: <outputted when generating the certifcates tar>
             oauth_consumer_secret: <outputted when generating the certifcates tar>
```

### Katello Proxy scenario without supplied certificates tar:

```yaml
    - hosts:
      - foreman-proxy.example.com
    roles:
       - role: foreman_installer
         foreman_installer_pkg: foreman-proxy-content
         foreman_installer_scenario: foreman-proxy-content
         generate_proxy_certs_from: katello.example.com
         katello_proxy_cert: "{{ vault_proxy1_cert }}"
         katello_proxy_key: "{{ vault_proxy1_key }}"
         katello_proxy_csr: "{{ vault_proxy1_csr }}"
         katello_ca: "{{ vault_katello_ca }}"
```

### Katello cluster with custom certificates: 

Couple of things to note:
 - All the key, secret & password answers are there as these must be the same across the cluster. Please don't use the values in this example.
 - The `foreman_installer_patches` options was only used to backport http://projects.theforeman.org/issues/20021 to my Katello version.

```yaml
    - hosts:
      - katello1.example.com
      - katello2.example.com
    var_files:
      - group_vars/vault_certs.yml
    roles:
       - role: foreman_installer
         foreman_installer_pkg: foreman-installer-katello
         foreman_installer_scenario: katello
         installer_scenarios_answers:
           foreman_proxy_content:
             pulp_oauth_secret: uC2qfoQfPVhdFTBEbS89ykZWQz6BVpcu
           foreman:
             db_password: KmVzXiWuVWCUZrn4kWF8PRsFG4H4ecqo
             initial_location: Global
             initial_organization: AMCE
             admin_password: changeme
             servername: katello.example.com
             foreman_url: https://katello.example.com
             oauth_consumer_key: xmi95B9qNQoX6owdg4MT8WMCBNhgudYy
             oauth_consumer_secret: x6TheD8Z9ZBtgdgBUrqSbPR2rh6k7UQE
           foreman_proxy:
             registered_name: katello.example.com
             registered_proxy_url: https://katello.example.com:9090
             oauth_consumer_key: xmi95B9qNQoX6owdg4MT8WMCBNhgudYy
             oauth_consumer_secret: x6TheD8Z9ZBtgdgBUrqSbPR2rh6k7UQE
             foreman_base_url: https://katello.example.com
             trusted_hosts:
               - katello.example.com
               - katello1.example.com
               - katello2.example.com
           katello:
             oauth_secret: uC2qfoQfPVhdFTBEbS89ykZWQz6BVpcu
         foreman_installer_custom_hiera:
           candlepin::db_password: L45DkebcvWdgXG9ryzWkfavSvQ23dw8U
         foreman_encryption_key: dfc6799e4d722a4e86c786cb0fc96cbbae0151f6
         katello_cluster_group: katello-servers
         katello_ca: "{{ vault_katello_ca }}"
         katello_cert: "{{ vault_katello_cert }}" # Certificate must use dns-alt-names with all cluster Hostnames and VIP hostname.
         katello_key: "{{ vault_katello_key }}"
         katello_csr: "{{ vault_katello_csr }}"
         foreman_installer_patches:
           - { src: files/katello_certs_tools.patch, basedir: /usr/lib/python2.7/site-packages/ }
           - { src: files/puppet-certs.patch, basedir: /usr/share/katello-installer-base/modules/certs/ }
```

### Foreman proxy cluster that connected to a Katello cluster with custom certificates: 

Each proxy is there own proxy in Foreman, but a client can use a VIP address to connect to either of them for packages.

```yaml
    - hosts:
      - foreman-proxy1.example.com
      - foreman-proxy2.example.com
    roles:
       - role: foreman_installer
         foreman_installer_pkg: foreman-proxy-content
         foreman_installer_scenario: foreman-proxy-content
         generate_proxy_certs_from: katello1.example.com
         katello_proxy_cert: "{{ vault_proxy1_cert }}" # Certificate must use dns-alt-names with all cluster Hostnames and VIP hostname.
         katello_proxy_key: "{{ vault_proxy1_key }}"
         katello_proxy_csr: "{{ vault_proxy1_csr }}"
         katello_ca: "{{ vault_katello_ca }}"
         installer_scenarios_answers:
           foreman_proxy_content:
             parent_fqdn: katello.example.com
           foreman_proxy:
             foreman_base_url: https://katello.example.com
             trusted_hosts:
               - katello1.example.com
               - katello2.example.com
               - katello.example.com
```
