---
# Variables for ansible role: Spectrum Scale bridge for grafana 


```yaml
scale_grafana_bridge_scale_cluster_name: "scale-test-os"   #Name of the cluster, is used to add content to separate configur
scale_grafana_bridge_scale_node_name: "scale-test-os-1"   #IP or hostname to the Spectrum Scale GUI/PMCollector node.
```

Section for ibm-spectrum-scale-bridge-for-grafana github repo.
- Download of ibm-spectrum-scale-bridge-for-grafana
```yaml
scale_grafana_bridge_repo_url: https://github.com/IBM/ibm-spectrum-scale-bridge-for-grafana.git
scale_grafana_bridge_repo_dest_path: /var/tmp/ibm-spectrum-scale-bridge-for-grafana
scale_grafana_bridge_git_version: v7.0.4  # HEAD should take the newest version.
```

Section for copying over mmsdrfs file from Spectrum scale node to node with podman.
```yaml
scale_grafana_bridge_scale_gpfsconfig_src_path: /var/mmfs/gen/mmsdrfs #location of the mmsdrfs on scale node.
scale_grafana_bridge_gpfsconfig_dest_path: "{{ scale_grafana_bridge_repo_dest_path }}/source/gpfsConfig"
scale_grafana_bridge_gpfsconfig_tmp_src_path: "{{ scale_grafana_bridge_gpfsconfig_tmp_dest_path }}/mmsdrfs"
scale_grafana_bridge_gpfsconfig_tmp_dest_path: temp/  # This will be located in you ansible role temp folder. roles/ibm-spectrum-scale-install-infra/temp/mmsdrfs
```


This will clean up the Grafana Dashboard, Datasource, API Key, (if not expired) PODMAN containers, systemd, folders(github,logs,binary), and Scale Perf apikey,
```yaml
scale_grafana_bridge_cleanup: false
```

If you want preserve the Container image that is built
```yaml
scale_grafana_bridge_cleanup_image: "{{ scale_grafana_bridge_cleanup }}"
```

-------

### Section for Spectrum Scale Bridge Container in Podman.

Run the Container as non-root user, # todo: this for future function, running container with user.
```yaml
scale_grafana_bridge_rights_user: root
scale_grafana_bridge_rights_group: root
scale_grafana_bridge_rights_mode_file: "0644" #rights on file
scale_grafana_bridge_rights_mode_folder: "755" # right on folder
```

Grafana bridge build the image and set name.
```yaml
scale_grafana_bridge_build_image_name: "localhost/{{ scale_grafana_bridge_scale_cluster_name }}-grafana-bridge-image"
scale_grafana_bridge_build_image_tag: 7.0.4 #default 7.0.4
```

-----


### Variables for Spectrum Scale Bridge Container in Podman.


IP or hostname #where we should point the Grafana datasource to
```yaml
scale_grafana_bridge_hostname: "https://10.33.3.105" 
```
Network port that is used for the datasource and container export
```yaml
scale_grafana_bridge_port: 8443 # Default 8443
```

Variables for PODMAN container.
```yaml
scale_grafana_bridge_container_name: "scale-grafana-bridge-{{ scale_grafana_bridge_scale_cluster_name }}"
scale_grafana_bridge_container_image: "{{ scale_grafana_bridge_build_image_name }}"
scale_grafana_bridge_container_state: started 
scale_grafana_bridge_container_detach: true
scale_grafana_bridge_container_log_level: info
scale_grafana_bridge_container_exposed_ports: "{{ scale_grafana_bridge_port }}"
scale_grafana_bridge_container_ports: "{{ scale_grafana_bridge_port }}:{{ scale_grafana_bridge_port }}"
scale_grafana_bridge_container_systemd_container_prefix: container 
scale_grafana_bridge_container_systemd_path: /etc/systemd/system/
```

- Volumes that will be mapped into container.
  - First mapping before : is locally then next is inside the container
  - First line: Is for logging, log name is zserver.log ++
  - Second line: Is for ssl certificate
  - Third line: Is for the Scale performance apikey
```yaml
scale_grafana_bridge_container_volumes:
  - "{{ scale_grafana_bridge_bridge_container_log.local_path }}:/var/log/ibm_bridge_for_grafana"
  - "{{ scale_grafana_bridge_scale_perf_ssl_cert.tlskeypath }}:/etc/ibm_scale_grafana_bridge/certs"
  - "{{ scale_grafana_bridge_scale_perf_apikey.path }}:/etc/ibm_scale_grafana_bridge/apikey/"
```

**Read this section carefully before change:**
- This populates parameters to PODMAN bridge container `container:env`
  - So if you want to change one or more variables, you can do this by changing the `scale_grafana_bridge_container_environment:`
  - If you want to add more parameters/variables to the container, then you can use the `scale_grafana_bridge_container_env` that will import all the parameters you describe/set


```yaml
scale_grafana_bridge_container_environment:
  protocol: "https"
  server: "{{ scale_grafana_bridge_scale_node_name }}"
  apikeyvalue: "{{ scale_grafana_bridge_scale_perf_apikey.path}}{{ scale_grafana_bridge_scale_perf_apikey.name}}" #/etc/bridge_ssl/api/*
  apikeyname: "{{ scale_grafana_bridge_scale_perf_apikey.name}}"
  tlskeypath: "{{ scale_grafana_bridge_scale_perf_ssl_cert.tlskeypath }}"
  tlskeyfile: "{{ scale_grafana_bridge_scale_perf_ssl_cert.tlskeyfile }}"
  tlscertfile: "{{ scale_grafana_bridge_scale_perf_ssl_cert.tlscertfile }}"
  port: "{{ scale_grafana_bridge_container_exposed_ports }}"
```


- This passes the whole section underneath to the ENV section in PODMAN, so that we can add parameters easily.

```yaml
scale_grafana_bridge_container_env:
  PROTOCOL: "{{ scale_grafana_bridge_container_environment.protocol }}"
  SERVER: "{{ scale_grafana_bridge_container_environment.server }}"
  APIKEYVALUE: "{{ scale_grafana_bridge_container_environment.apikeyvalue }}"
  APIKEYNAME: "{{ scale_grafana_bridge_container_environment.apikeyname }}"
  TLSKEYPATH: "{{ scale_grafana_bridge_container_environment.tlskeypath }}"
  TLSKEYFILE: "{{ scale_grafana_bridge_container_environment.tlskeyfile }}"
  TLSCERTFILE: "{{ scale_grafana_bridge_container_environment.tlscertfile }}"
  PORT: "{{ scale_grafana_bridge_container_environment.port }}"
```


- The path is used outside the container:
```yaml
scale_grafana_bridge_bridge_container_log:
  local_path: "/var/log/ibm_scale_grafana_bridge"
```


- The name and location of the Spectrum Scale APIKEY.
  - The paths are only used outside the container, the files name are used inside the same container.
  - The scale apikey name needs to be with underscore if dividers used. 

```yaml
scale_grafana_bridge_scale_perf_apikey:
  name: "scale_grafana_bridge_api"
  path: "/etc/ibm_scale_grafana_bridge/apikey/" # path to where the apikey should be saved.
```

### Section for TLS Certificate

- This is used when generating Self Signed Certificates. 

- Common name is used when generating the TLS KEY.
  - **Change the common name to your server,** this just a alias into `scale_grafana_bridge_scale_perf_ssl_cert`
```yaml
scale_grafana_bridge_scale_perf_ssl_cert_common_name: "grafana"
```

- This is used when generating Self Signed Certificates.
```yaml
scale_grafana_bridge_scale_perf_ssl_cert:
  tlskeypath: "/etc/ibm_scale_grafana_bridge/certs/"
  tlskeyfile: "privkey.pem"
  tlscertfile: "cert.pem"
  tlscsrfile: "cert.csr"
  tlsprivatesize: "2048"
  tlsprivatekeytype: "RSA"
  tlsvalid: "+365d"
  tlsprovider: "selfsigned"
  entrust_not_after: "+365d"
  common_name: "{{ scale_grafana_bridge_scale_perf_ssl_cert_common_name }}"
```



### This Section is for Creating DataSource and Importing Dashboard to Grafana Instance/Server

- Point this to your Grafana instance. (Dashboard)
```yaml
scale_grafana_bridge_grafana_url: "http://10.33.3.105:3000"
```

- If the grafana User and Password is set, it will connect up and create a short-lived apikey for import.
```yaml
scale_grafana_bridge_grafana_user: admin
scale_grafana_bridge_grafana_password: "jkn-cwu6umt@MPZ2fkd"
```

- Create an API key manually in the grafana instance/server if you dont want to use username and password. 
  - If the `scale_grafana_bridge_grafana_api_key` is set then it will not use the grafana user.
```yaml
- scale_grafana_bridge_grafana_api_key: 'eyJrIjoiSmd0SFFkZ1Q1TFFuZjRYVDlBcGlPcnZrRWlJa3JpZTYiLCJuIjoidGVzdGFwaSIsImlkIjoxfQ=='
```

- The API key to create on Grafana Instance if `scale_grafana_bridge_grafana_api_key` is not set.
  - This is a short-lived api and will be deleted and created again if not saved locally.
  - **PS**: Have not succeeded to search for Expired apikey (Something with the restapi calls), so if you get error that the user is in use, change the name or delete the API key in grafana manually
  
```yaml
scale_grafana_bridge_grafana_api_keys:
- name: "scale-apikey"
  role: "Admin"
  secondsToLive: 5000 # this is in seconds
```


- This will save the Grafana APIKey on remote host,
  - if this set, it will check if key exist in the `scale_grafana_bridge_grafana_api_keys_dir:` and use that.
```yaml
scale_grafana_bridge_grafana_api_key_save: false
```
- The location where the Grafana API key should be stored locally. (Normally the first host in you inventory)
```yaml
scale_grafana_bridge_grafana_api_keys_dir: "/tmp/grafana/keys"
```


- Will give out extra task with debug level. not in use for all task.
  - TODO add Verbosity instead
```yaml
scale_grafana_bridge_debug: false #default: false 
```

- To run the task with no_log for not output keys and API keys, (not valid on all task when set_facts.)
```yaml
scale_grafana_bridge_no_log: true  #default: true
```

- Name of the grafana folder to create and where it should be created.
```yaml
scale_grafana_bridge_grafana_folder: "{{ scale_grafana_bridge_scale_cluster_name }}"
```
- When set, it creates the grafana folder that is used to import dashboard, if not you need to create it manually.
```yaml
scale_grafana_bridge_grafana_folder_create: true #default true
```

- Set this to use the same grafana_folder on all Dashboard imports
```yaml
scale_grafana_bridge_grafana_folder_global: "{{ scale_grafana_bridge_grafana_folder }}"
```

- Variables for the Grafana task to validate certs against Grafana instance.
```yaml
scale_grafana_bridge_grafana_validate_certs: "no"
```

- Variables for the Grafana Folder task with state. valid variables: `absent`, `present`,
  - absent will remove what's created
```yaml
scale_grafana_bridge_grafana_folder_state: "present"
```

- If you don't have `owerwrite` when re-run (playbook) import grafana dashboards, it will fail with error: "Unable to update the dashboard xxx : The dashboard has been changed by someone else (HTTP: 412)"
  - (have not found anyway to map this to a existing version, so for each ansible play, it wil create a new version of the dashboard in grafana.)
```yaml
scale_grafana_bridge_grafana_dashboards_overwrite: yes
```

- Variables for the Grafana Dashboard task with state. Valid variables: `absent`, `present`
  - `absent` will remove what's created
```yaml
scale_grafana_bridge_grafana_dashboard_state: "present"
```

- Variables to set the same datasource for all dashboards.
```yaml
scale_grafana_bridge_grafana_datasource_name_global: "ds-{{ scale_grafana_bridge_scale_cluster_name }}"
```

- List of all dashboards to download change the datasource on and import them to grafana.
  - We could also have imported the dashboards from the local gitclone, but then we might not have the same possibility to set names etc.
  - And with dashboard provisioner, we need access to the grafana instance. not just the restapi. (trying to do this without too much access) also for grafana instance in Openshift.
  - To change the dashboards you want, add, change the dashboards list below with a example from the ibm-spectrum-scale-bridge-for-grafana: https://github.com/IBM/ibm-spectrum-scale-bridge-for-grafana/tree/master/examples

```yaml
scale_grafana_bridge_grafana_dashboards:
 - name: FileSystemCapacityView-1596986743960.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/File%20System%20Capacity%20View-1596986743960.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
   grafana_folder: "{{ scale_grafana_bridge_grafana_folder_global }}"
 - name: FileSystemsView-1596986639199.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/File%20Systems%20View-1596986639199.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
 - name: FilesetQuotas-1596986773193.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/Fileset%20Quotas-1596986773193.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
 - name: InodesCapacityView-1596986797679.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/Inodes%20Capacity%20View-1596986797679.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
 - name: MainDashbord-1596986550321.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/Main%20Dashbord-1596986550321.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
 - name: NSDServer-1596986691522.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/NSD%20Server-1596986691522.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
 - name: Network-1596986669168.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/Network-1596986669168.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
 - name: PoolCapacityView-1596989290996.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/Pool%20Capacity%20View-1596989290996.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
 - name: Protocols-1596986715438.json
   url_path: https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-bridge-for-grafana/master/examples/grafana_dashboards/Example_Dashboards_bundle/Predefined%20Basic%20Dashboards/Protocols%20(NOT%20SUPPORTED%20NOW)-1596986715438.json
   datasource_name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
```

- Grafana Data Source, the `tls_client_cert_file` and `tls_client_key_file` will be generated automatically,

- They can be replaced by signed one, just place them on the container/PODMAN host and change variables. and set `scale_brige_generate_ssl_certificate: false`
```yaml
scale_grafana_brige_gen_ssl_certificate: true
```
- Then and populate the correct parameters:
```yaml
tls_client_cert_file: "/etc/bridge_ssl/certs/cert.pem"
tls_client_key_file: "/etc/bridge_ssl/certs/privkey.pem"
```

- Optional: They whole cert's and key kan be set as a string with backslash \ for line break
```yaml
tls_client_cert_file: "-----BEGIN CERTIFICATE-----\nMIIDiTCCAnGgAwIBAgIUEDtzuxAzIYL/DML05apPouTElwgwDQYJKoZIhvcNAQEL\nBQAwVDELMAkGA1UEBhMCWFgxFTATBgNVBAcMDERlZmF1bHQgQ2l0eTEcMBoGA1UE\nCgwTRGVmYXVsdCBDb21wYW55IEx0ZDEQMA4GA1UEAwwHZ3JhZmFuYTAeFw0yMTEy\nMTAxMzA4MDNaFw0yMjEyMTAxMzA4MDNaMFQxCzAJBgNVBAYTAlhYMRUwEwYDVQQH\nDAxEZWZhdWx0IENpdHkxHDAaBgNVBAoME0RlZmF1bHQgQ29tcGFueSBMdGQxEDAO\nBgNVBAMMB2dyYWZhbmEwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDr\npYvmlM+jzl0uAdCtu9DTM7NY1g1GYQp9e2jxIWATtlF08X+O/TM7VqmkgF2q7qA1\n/AG/KjW6VszUW4nbrfM9pbNfzd1pId6m34pkZerN93L59vUSIAM21iG6HMcWlZdU\nmgM/4IAGRe+DtzZxY3bf5gLtRx5cPzArXNqX7IiEHSUvyDTErJ5Dw/QbF2Xs7jqN\ngGYk/FLZWEv5EKwuniwWG4UImRFLAVzKQT/NklXcl95Q/czhjJh1rn+rGT5OdQRV\nG1tgxAhfUDQ10MJH+FqmoAD4JWbrQJtaM5qLqCFbmmZ5NkpbwZaV7oeBG23Sjzch\ng6kRQuvQ545IPlUmJJJHAgMBAAGjUzBRMB0GA1UdDgQWBBRTG6NwLshFOGinj4Lg\nPXEMZRW11TAfBgNVHSMEGDAWgBRTG6NwLshFOGinj4LgPXEMZRW11TAPBgNVHRMB\nAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUAA4IBAQA1p01+6SeTRdPo4t17YLxEaLzi\nxLutGV9OFx4BkVwtwkdXCED1P0IDVTCktdQoZbYvjMBPggDEaHh+wenGfeAhItv2\nh795JYxTVSWded8OPc9rVuuigNvGLjypt8V1sZnAG6nyZxyBi6bQSWqGj4MYz1Ie\nVvJ75KR6h9EbKnSnEstKklIMszyvKJQfFgev9ntlch4Y6/+rgkXT4inLAWs8H6da\nP0EzY6k4w1umzQGB5nHAkBPX1i11G2E+eMFC0c0dG5kwPticq6lElJPeYX8LZUWS\nBz394XvGIkGdbs6YNjxOJ1E/GSFCGiDiFYoW34uU/11Ed2tRL8P9uKzN6LVO\n-----END CERTIFICATE-----\n"
tls_client_key_file: "-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDrpYvmlM+jzl0u\nAdCtu9DTM7NY1g1GYQp9e2jxIWATtlF08X+O/TM7VqmkgF2q7qA1/AG/KjW6VszU\nW4nbrfM9pbNfzd1pId6m34pkZerN93L59vUSIAM21iG6HMcWlZdUmgM/4IAGRe+D\ntzZxY3bf5gLtRx5cPzArXNqX7IiEHSUvyDTErJ5Dw/QbF2Xs7jqNgGYk/FLZWEv5\nEKwuniwWG4UImRFLAVzKQT/NklXcl95Q/czhjJh1rn+rGT5OdQRVG1tgxAhfUDQ1\n0MJH+FqmoAD4JWbrQJtaM5qLqCFbmmZ5NkpbwZaV7oeBG23Sjzchg6kRQuvQ545I\nPlUmJJJHAgMBAAECggEARaeNjv710YmyaWMH+BLIS2XA4cWD7wXIQAc2ofAnoiwG\nL/ciqXWWqzeWtZVsGpamrM5tBcDIWOhHa44TVeg5OkO8ndkQVX85fUIeekbV/UPj\nrJefIVhtGsx487aF8tsM/Gj6BOurbC9H+Tsy0JmCDfTDcLfQ9ZuH9Ylg4/966vHR\nR8KNwoZJx6mCiT7b+oewMF3s9NkkxT1d08simgNf44Jx/I050g23c0E6Niob6LcN\nJ0MiFcZiEBAAVUDvNDY1893NlObiQrCyZWmZcUTlHcBRtN4vL9/qwt9jsDb+bLdZ\ni4xONDu4l0sXMBHQ8Jj2m1cAyzAs/U1BzmF9ixpiQQKBgQD8DwtGPUIKViFwzLcG\n9owXOcTDxQQU76VBNjaaswLPMy9eDPRCkTXAWBvuZJ9Ih1Y0gyGiuvtZh5cf0ZxT\n0U+t2tQB1bpaKCXKat56ybPLB95Db2rV0PdfKxmRaO31ScSwZsl/YjKGvd9XhS5K\nuob8waADSe5gfOKU/CVl2QpDrwKBgQDvVM6epMC8PAkIKMZSQfouAQhDdHOYYJsO\nVxxgLQf270LFDAps8dU2M8MjUwIkgExQ+BdZjcvellEXktm7UqC8oNEVUXej1Ywi\n/FFIkLzkKS15zhUP4tkVqkoasLnfesp5nWv6ENg1fkE2fzd/Wu1wSZ0vBs1TmC3L\nGkKJU6iI6QKBgC65xzhFINnzr41OldtXlw6zKdO00RXkevkEyMiSyMGKVoyT0DAK\n5TD75GmkA5cZZ5SifnjBOtkU9qHyZI1xLtkmyMhyS3JtINxORWHzxD2t/rj3jZGH\nhGQDBGFdV0dyXmDpHQ9dL8qkpiN+T9+QhneSmUwix2rhm8tMls4zluCHAoGAZI+c\n1bniJfWP0fbYBd4lEclrQHSg0Yjd/fOKP7sMGqzDwGnjw40FimXLe384aj/iUS89\nGGrlG5zLa/1PMU9xrHBiCfQWMifbXyPnv3bZd4D507FM1kT59Al+Y6KYJxfAFcOY\niBUl06w+GHjxx7hcBg9YVVclVRefPjTFelBFg2kCgYEAikg67jpXA7rPPnutXrb2\nEWvff8VU2LCae4qsaQDrS1RcUklch9G7tDApIvfcheboM3W8Flld9VpwAwe/O9l7\nqMmQbyGXK7sl738oFQOg9fqtDBS0mW2B44bDGtIHEA7vMXIFaDnHAIhlY963nxPx\n6gP7OSMsAjcDRseH/2IfVb8=\n-----END PRIVATE KEY-----\n"
```

- Variables for grafana DataSource creation, name need to match with the dashboards datasource.
```yaml
scale_grafana_bridge_grafana_datasources:
- name: "{{ scale_grafana_bridge_grafana_datasource_name_global }}"
  url: "{{ scale_grafana_bridge_hostname }}:{{ scale_grafana_bridge_port }}"
  ds_type: "opentsdb" #Default: opentsdb
  isDefault: no #Default: no
  state: present #Default: present
  tsdb_resolution: second #Default: second
  tsdb_version: 3 #Default: 3
  tls_skip_verify: yes #Default:yes
  editable: true #Default: true
  tls_client_cert_file: "{{ scale_grafana_bridge_scale_perf_ssl_cert.tlskeypath }}{{ scale_grafana_bridge_scale_perf_ssl_cert.tlscertfile }}"
  tls_client_key_file: "{{ scale_grafana_bridge_scale_perf_ssl_cert.tlskeypath }}{{ scale_grafana_bridge_scale_perf_ssl_cert.tlskeyfile }}"
```