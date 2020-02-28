 # Docker Swarm Management for Heavy Forwarders
 
 ## Introduction
 
 ### Problem
 
 Managing "do everything" heavy forwarders is tedious: dependencies, deployment servers, conflicting configurations,
 failover.
 
 This can be easier.
 
 ### Solution
 
 Containers allow us to independently manage one input, or group a handful of similar inputs without additional
 hardware, real or virtual.  They can also be shipped to another infrastructure without worrying about the other admins
 changing anything.
 
 Docker Swarm enables scaling and failover for these containers.  Sprinkle in some persistent storage and your heavy
 forwarders are now significantly more resilient than before.
 
 ## Basic Process
 
 These playbooks provide you with this path:
 
 * Define your heavy forwarder image
   * Operating System
   * Splunk Version
   * TAs (or other configs) to install via local file, templates, git repositories, etc.
   * Healthcheck index/sourcetype and expected event time period
   * Volumes for persistent storage
   
* Build your image, along with some convenient variants

   * Primary image
      * Intended to be deployed to swarm
      * Disables indexes, splunkweb
      * Configures license master
      
   * Standalone image
      * Eases development efforts prior to deploying to swarm
      * Leaves splunkweb enabled
      * Allows simple index creation
   
* Push your image to a registry

* Deploy a service using your image

## Examples

### Simplest image definition

This image doesn't actually do anything valuable, but shows the required variables that must be set to perform the
build.  The full inventory file is in [all-in-one/inventory.yml](examples/all-in-one/inventory.yml).  The below shows
the host variables only.

    # every image must have a version
    version: 1.0.0

    # define where to get the splunk tarball
    # this can be http://, ftp://, file://, etc.
    # or more specifically, any URL wget will handle
    splunk_tgz_url: <url to splunk tarball>

    # set the splunk admin user/password for user-seed.conf
    splunk_admin_user: admin
    splunk_admin_password: pleasechangeme
    # you could use splunk_admin_password_hash for a pre-hashed password
    # splunk_admin_password_hash = <your password hash>

    # a license master uri is required
    # though forwarders don't index, and therefore don't consume license, they sometimes need to run searches
    # against populated lookups to forward results to the indexing tier.  the forwarder license doesn't enable
    # this type of usage.
    license_master_uri: https://lm.example.org:8089
    
### Splunk metrics.log healthchecking image

#### But first, let's organize with group_vars

Because our image definitions are represented as Ansible hosts, they can make use of Ansible's standard variable
precedence.  For our examples we're going to place the common variables from the above example in group_vars. This
inventory is named [organized-environment](examples/organized-environment).

Here we have an [inventory.yml](examples/organized-environment/inventory.yml), just as above.  But it only keeps the
host-specific variables, not the common ones, as they have been moved to
[organized-environment/group_vars/all.yml](examples/organized-environment/group_vars/all.yml).

Our simplest image definition is still included in this [inventory.yml](examples/organized-environment/inventory.yml),
but now it only has one variable (`version`), since the rest of its configuration was generic and applicable to almost
any host in our environment.

Now that our inventory is a bit easier to manage for multiple images, we'll move to the healthchecking image.

#### internal_healthcheck_forwarder's image-specific config

In [organized-environment/inventory.yml](examples/organized-environment/inventory.yml) under
`internal_healthcheck_forwarder`:

    # remember every image must have a version number
    version: 1.0.0
    
    # perform the check every 30 seconds
    healthcheck_interval_seconds: 30

    # in splunk's metrics.log
    # for the per_index_thruput group
    healthcheck_metrics_group: per_index_thruput
    # for the _internal series
    healthcheck_metrics_series: _internal
    # expect to see non-zero thruput no older than 60 seconds
    healthcheck_allowed_age_seconds: 60

This configuration enables Docker's builtin healthcheck functionality, using Splunk's metrics.log to identify if the
container has processed events for the configured group/series.

### Staging components into your Docker image

For any of this to have any value, you need a way to add custom content to the built image.  This is done through
"stage_items".  You are given two standard variables to use: `common_stage_items` (intended for `group_vars`) and
`host_stage_items` (intended for `host_vars`).

These two variables allow you to deploy a common set of components to all of your images, while still allowing you to
tailor individual images for their specific purposes.

An example of `host_stage_items` use is to include the specific Splunk TA and `inputs.conf` to enable data collection.

But all of your images will need to forward their collected data to your indexers.  `common_stage_items` lets you define
that configuration just once, in `group_vars`, to ensure that every container you run will properly send its events,
and not just index them locally to never see the light of day.

In our inventory, we have this in [group_vars/all.yml](examples/organized-environment/group_vars/all.yml):

    common_stage_items:
      - type: copy
        src: "{{ inventory_dir }}/files/docker_forwarder_outputs"
        # stage_items dest points to a staging_path
        dest: splunk
        # because dest is, effectively, /opt/splunk, have this directory copied to a subdirectory
        dest_sub_path: etc/apps/docker_forwarder_outputs
        
This tells the build process to copy
[files/docker_forwarder_outputs](examples/organized-environment/files/docker_forwarder_outputs) to the `splunk`
staging path, and place it `etc/apps/docker_forwarder_outputs` under that path.

Which brings us to...

#### Staging paths

A "staging path" is a directory that will be copied during the build process, via `Dockerfile`'s `COPY` command.  This
was implemented to minimize the number of image layers that will be created, since each `COPY` command creates a new
layer. It ends up simplifying most image build definitions, by allowing re-use of standard paths.

These playbooks ship with two default build paths:

* `build`
* `splunk`

`build` represents the directory where `Dockerfile` is created.  As such this path never has a COPY performed on it.

`splunk` represents `$SPLUNK_HOME`.  This is where most configurations end up needing to be placed.

Additional staging paths can be created, but that is an advanced topic for later.

#### Standalone images shouldn't forward to the indexers

But what about the standalone image that also gets created?  In my opinion it shouldn't forward events to the indexers,
because it's intended to be run in a more convenient form (with splunkweb still enabled, etc.) where I imagine most of
your troubleshooting would occur.

Stage items also give you the option of using a `condition`. This is a jinja expression that is evaluated while staging
items to build your image, and if true that particular item will be included for that image.  Our
`docker_forwarder_outputs` example actually looks like this in our
[group_vars/all.yml](examples/organized-environment/group_vars/all.yml):

    common_stage_items:
      - type: copy
        src: "{{ inventory_dir }}/files/docker_forwarder_outputs"
        # stage_items dest points to a staging_path
        dest: splunk
        # because dest is, effectively, /opt/splunk, have this directory copied to a subdirectory
        dest_sub_path: etc/apps/docker_forwarder_outputs
        # but only when building the primary (not standalone) build
        condition: "{{ build_vars.primary_build }}"
        
This `condition` makes use of a variable defined in our "build variations" configuration, `primary_build`, which has
a value of `True` when the non-standalone image is built.  Thus we have these image versions built:

* 1.0.0 (has `docker_forwarder_outputs`)
* 1.0.0-standalone (does not have any forwarding app installed)

#### Included, out of the box stage items

[templates/build](roles/docker_image_build/templates/build)
* Dockerfile

[templates/base](roles/docker_image_build/templates/base)
* Install user-seed.conf
* Configure server.conf, inputs.conf with servername
* Configure convenience indexed fields

[templates/kvstore_disable](roles/docker_image_build/templates/kvstore_disable)
* Disables KVStore on primary builds (but overrideable)

[templates/metrics_healthcheck](roles/docker_image_build/templates/metrics_healthcheck)
* Enables the metrics.log healthcheck functionality detailed above

[templates/primary_build](roles/docker_image_build/templates/primary_build)
* Deployed to primary builds only
* Disables indexes
* Disables splunkweb
* Sets license master

The set of included stage items attempts to get you on your feet with this process as soon as possible.  These are
configurations we consider standard and reasonable for almost all instances.

### Time to build!

Now that we've discussed the components of the image definition, let's actually build the sample inventory.

#### Prerequisites

TODO - Add them here

#### The build playbook

From the directory containing this set of playbooks, run:

`ansible-playbook -i examples/organized-environment/inventory.yml build.yml`

If all goes well, you should have four new docker images:

    % docker image ls
    REPOSITORY                                                TAG                 IMAGE ID            CREATED             SIZE
    hello_swarm_forwarder                                     1.0.0               71900b65657b        40 seconds ago      1.16GB
    hello_swarm_forwarder                                     1.0.0-standalone    f4b506189917        25 seconds ago      1.16GB
    internal_healthcheck_forwarder                            1.0.0               4092a1157016        53 seconds ago      1.16GB
    internal_healthcheck_forwarder                            1.0.0-standalone    b556f86c181d        25 seconds ago      1.16GB

### Time to push!

### Time to deploy!
