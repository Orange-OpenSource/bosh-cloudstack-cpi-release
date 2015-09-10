[![Build Status](https://travis-ci.org/cloudfoundry-community/bosh-cloudstack-cpi-release.png)](https://travis-ci.org/cloudfoundry-community/bosh-cloudstack-cpi-release)



# bosh-cloudstack-cpi-release

This is work in progress / feedbacks welcome (use issues).


## Design :
* this CPI is a Bosh external CPI i.e a dedicated bosh release wich must be deployed alongside a bosh release OR configured with bosh release in a bosh-init yml manifest
*  CPI business logic is in a dedicated spring boot app, included in bosh-cloudstack-cpi-release, called cpi-core. This java component has a dedicated template / monit service.
* leverages Apache Jclouds support for CloudStack, nice java adapter for Cloudstack API
* supports Cloudstack advanced zone
* secondary / ephemeral implemented as cloudstack volume (no ephemeral disk concept in CloudStack).
* Uses a webdav http server to enable stemcell / template loading by cloudstack (other option was create volume and transform to template, required bosh / bosh-init to be hosted in the target cloud / tenant). This webdav server is started from the spring boot cpi core jvm

* Out of Scope : security groups provisioning / CS Basic Zones

## Current Status:



### Global status
* micro-bosh creation (with an externally launched cpi-core process)
* compilation vms ok, blobstore ok
* bosh-master creation from micro-bosh
* concourse creation from bosh-master
	
### Issues
* local storage issues (persistent disks happen to not be on the same host as vm when reconfiguring a deployment)
* ip conflicts when recreating vm, due to cloudstack expunge delay (orig vm is destroyed but ip not yet releases)
* disk / mount issues (probably related to vm expunge delay)
* stemcell agent.json user data url is hardcoded. must find a way to find the correct cloudstack userdata url on the fly
* stemcell : cant get keys from cloudstack metadata (requires stemcell code change to match cloudstack)
* no support for vip / floating ip yet	


## TODO

* run Director BATS against the cpi
	https://github.com/cloudfoundry/bosh/blob/master/docs/running_tests.md

* expunge vm
	required with static ip vms (ip not freed until vm is expunged)
	default value expunge.delay = 300 (5 mins)
	option 1
		reduce to 30s as cloudstack admin => need to restart management server 
		wait 30s in CPI after vm delete
	option2
		expunge with cloudstack API. option not avail in jclouds 1.9, use escaping mechanism to add the expunge flag?
* check template publication
	from cpi-core webdav, only public template possible ?
	validate publication state (instant OK for registering, need to wait for the ssvm (secondary storage vm) to copy the template
	connectivity issue when cloudstack tries asynchronously to copy template : vlan opened, dedicated chunk encoding compatible spring boot endpoint
	(workaround: mock template mode in cpi. copies an existing template and use it as a stemcell image)
	
	add a garbarge collection mechanism to the webdav server (remove old templates when consumed by cloudstack)	

* use isolated / dedicated networks for bosh vms
	http://www.shapeblue.com/using-the-api-for-advanced-network-management/
	static API assigned through API deployVirtualMachine	

* persist bosh registry.
	now hsqldb
	set persistent file in /var/vcap/store/cpi
	TBC : use bosh postgres db (add postgres jdbc driver + dialect config + bosh *db credentials injection)


* globals
	harden Exception mangement
	map spring boot /error to an intelligible CPI rest payload / stdout
	update json reference files for unit tests

	

* provision ssh keys
	generate keypair with cloudstack API (no support on portail)
	use keypair name + private key in bosh.yml
	see http://cloudstack-administration.readthedocs.org/en/latest/virtual_machines.html?highlight=ssh%20keypair
	see http://chriskleban-internet.blogspot.fr/2012/03/build-cloud-cloudstack-instance.html




## Typical bosh.yml configuration to activate the CloudStack external CPI

```yml

# add the cpi bosh release
releases:
- {name: bosh, version: "185"}
- {name:  bosh-cloudstack-cpi, version: latest}

# add the template for cpi-core rest server
jobs:
- name: bosh_apidata
  templates:
  - {name: nats, release: bosh}
  - {name: redis, release: bosh}
  - {name: postgres, release: bosh}
  - {name: blobstore, release: bosh}
  - {name: director, release: bosh}
  - {name: health_monitor, release: bosh}
  - {name: powerdns, release: bosh}
  - {name: cpi, release: bosh-cloudstack-cpi} # <--


# activate external cpi
properties:
  director:
    cpi_job: cpi  

# set external cpi credentials
   cloudstack:
      endpoint: <your_cloudstack api end_point_url> # Ask for your administrator
      api_key: <your_api_key> # You can find at your user page
      secret_access_key: <your_secret_access_key> # Same as above
      default_key_name: <default_keypair_name> # Your keypair name (see the next section)
      private_key: <path_to_your_private_key> # The path to the private key file of your key pair
      state_timeout: 600
      state_timeout_volume: 1200
      stemcell_public_visibility: true
      default_zone: <default_zone_name> # Zone name
      proxy_host: <proxy to cloudstack api> #proxy acces active if set
      proxy_port: <proxy port>
      proxy_user: <proxy user>
      proxy_password: <proxy password>
      
	  cpi:
	    webdav_host: *bosh_static_ip
	    mock_create_stemcell: true
	    existing_template_name: "bosh-stemcell-3033-po10.vhd.bz2"
	
	    registry:
	      endpoint: http://<bosh_ip>:8080
	
	    blobstore:
	        address: *bosh_static_ip
	        port: 25251
	        provider: dav
	        agent: {user: agent, password: agent-password}
	    agent:
	      mbus: "nats://nats:nats-password@<bosh_ip>:4222"
	    ntp: ""
       
      
      
# define disk_pool, refering to cloudstack disk offering      
disk_pools:
- name: disks
  disk_size: 10000
  cloud_properties:
    disk_offering: "DO2 - Medium STD" #<--- Replace with your disk offering name for persistent disk
      
# define vm pool, refering to cloudstack compute offering      
resource_pools:
- name: vms
  stemcell:
    name: bosh-cloudstack-xeb-ubuntu-trusty-go_agent-raw
    version: latest
  network: private
  size: 1
  cloud_properties:
    compute_offering: "CO1 - Small STD" #<--- Replace with your compute offering name
    disk: 20000
    ephemeral_disk_offering : "DO2 - Medium STD" #<--- Replace with the disk offering u want for ephemeral disk
      
      
```


## Typical bosh-int configuration to create a micro-bosh with CloudStack external CPI

bosh-init configuration : micro-bosh.yml

* necessary configuration to activate on the TARGET bosh (micro-bosh) ie : CPI used from micro-bosh to create other deployments
* bosh init must use the cpi external release, and have the necessary configuration to create the microbosh vm. 


cpi-core config: application.yml
* exposes a webdav server (used by cloudstack iaas to get the template extracted from the stemcell)
* exposes a registry to the microbosh vm for bootstrapping purpose. cpi-core will generate the correct json setting. Bosh agents will copy it to /var/vcap/bosh/settings.json 
* generates adequate vm userdata when creating the vm, giving registry endpoint

The cpi-core must be launched separately, see the inception directory which provides a sample script and application.yml configuration.


As the bosh-init bootstrapping is quite cumbersome, its recommended to provision a bosh deployment from the microbosh and operate from that full managed bosh (best practice anyway).



```yml


---



```





## Contributing

In the spirit of [free software](http://www.fsf.org/licensing/essays/free-sw.html), **everyone** is encouraged to help
improve this project.

Here are some ways *you* can contribute:

* by using alpha, beta, and prerelease versions
* by reporting bugs
* by suggesting new features
* by writing or editing documentation
* by writing specifications
* by writing code (**no patch is too small**: fix typos, add comments, clean up inconsistent whitespace)
* by refactoring code
* by closing [issues](https://github.com/cf-platform-eng/cf-containers-broker/issues)
* by reviewing patches


### Submitting an Issue

We use the [GitHub issue tracker](https://github.com/cf-platform-eng/cf-containers-broker/issues) to track bugs and
features. Before submitting a bug report or feature request, check to make sure it hasn't already been submitted. You
can indicate support for an existing issue by voting it up. When submitting a bug report, please include a
[Gist](http://gist.github.com/) that includes a stack trace and any details that may be necessary to reproduce the bug,
including your gem version, Ruby version, and operating system. Ideally, a bug report should include a pull request
with failing specs.

### Submitting a Pull Request

1. Fork the project.
2. Create a topic branch.
3. Implement your feature or bug fix.
4. Commit and push your changes.
5. Submit a pull request.
