Deployment tips for puppet-tripleo
----------------------------------

This section will describe different ways of debugging puppet-tripleo changes.

Deploying puppet-tripleo using gerrit patches or source code repositories
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When adding new features to THT, in some cases dependencies should be merged
first in order to test newer patches, with the following procedure, the user
will be able to create the overcloud images using WorkInProgress patches from
gerrit code review without having them merged (for CI testing purposes).

If using third party repos included in the overcloud image, like i.e. the
puppet-tripleo repository, your changes will not be available by default in the
overcloud until you write them in the overcloud image (by default is:
overcloud-full.qcow2)

In order to make quick changes to the overcloud image for testing purposes, you
can:


Create the overcloud image by following (:doc:`in_progress_review`) :
::

  export DIB_INSTALLTYPE_puppet_tripleo=source

  export DIB_REPOLOCATION_puppet_tripleo=https://review.openstack.org/openstack/puppet-tripleo

  export DIB_REPOREF_puppet_tripleo=refs/changes/25/310725/14


In order to avoid noise on IRC, it is possible to clone puppet-tripleo and
apply the changes from your github account. In some cases this is particularly
useful as there is no need to update the patchset number.
::

  export DIB_INSTALLTYPE_puppet_tripleo=source

  export DIB_REPOLOCATION_puppet_tripleo=https://github.com/<usergoeshere>/puppet-tripleo

Remove previously created images from glance and from the user home folder by:
::

  rm -rf overcloud-full.*

  glance image-delete overcloud-full

  glance image-delete overcloud-full-initrd

  glance image-delete overcloud-full-vmlinuz

After this step the images can be recreated by executing:
::

  ./tripleo-ci/scripts/tripleo.sh --overcloud-images

Pushing changes directly without re-creating the overcloud images
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

On the undercloud as the stack user, clone the openstack modules using:
::

  #!/bin/bash
  set -eu
  cd ~
  if [ ! -e ~/puppet-modules ]; then
    mkdir ~/puppet-modules
    pushd ~/puppet-modules
      curl -O https://raw.githubusercontent.com/openstack/tripleo-puppet-elements/master/elements/puppet-modules/source-repository-puppet-modules
      mapfile -t REPO_CLONES < <(cat source-repository-puppet-modules | awk '{ print $4 " " $3 }' | sed -e 's|/opt/stack/puppet-modules/||')
      for CLONE in "${REPO_CLONES[@]}"; do
        git clone $CLONE
      done
    popd
  fi

Then, using the following script, upload the cloned puppet modules to swift
::

  #!/bin/bash
  set -eu
  cd ~
  tar --exclude-vcs --show-transformed-names --transform 's|puppet-modules/|opt/stack/puppet-modules/|' -czvf puppet-modules.tar.gz puppet-modules
  upload-swift-artifacts -f puppet-modules.tar.gz --environment puppet-modules.yaml

This will generates the puppet-modules.yaml environment file, which can be
added to the deployment command ("-e puppet-modules.yaml") and use cloned
puppet modules

After every manual change to any of the puppet modules it is possible to re-run
the upload to swift, and re-run `overcloud deploy` with the new env file. This
process will speed up the changes in the ovecloud images.

Debugging puppet-tripleo from overcloud images
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For debugging purposes, it is possible to mount the overcloud .qcow2 file:
::

  #Install the libguest tool:
  sudo yum install -y libguestfs-tools

  #Create a temp folder to mount the overcloud-full image:
  mkdir /tmp/overcloud-full

  #Mount the image:
  guestmount -a overcloud-full.qcow2 -i --rw /tmp/overcloud-full

  #Check and validate all the changes to your overcloud image, go to /tmp/overcloud-full:
  #  For example, in this step you can go to /opt/puppet-modules/tripleo,

  #Umount the image
  sudo umount /tmp/overcloud-full


From the mounted image file it is also possible to run, for testing purposes,
the puppet manifests by using ``puppet apply`` and including your manifests:
::

  sudo puppet apply -v --debug --modulepath=/tmp/overcloud-full/opt/stack/puppet-modules -e "include ::tripleo::services::time::ntp"

