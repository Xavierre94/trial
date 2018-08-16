.. _setup

#########################################
Setup for xPAD Test Execution Environment
#########################################

This documentation applies for the setup of xTEE in August 2018, it changes
everyday and an easiest way of installation is under development, so make sure
to update your local xTEE repositories.

*************
Preconditions
*************
* Have internet access inside the Ubuntu Vm (this guide uses Ubuntu 16.04). For that, configure your proxy environment variables editing your /etc/environment file and include:

.. code-block:: console

      http_proxy=http://qxnumber:password@proxy.muc:8080/
      https_proxy= http://qxnumber:password@proxy.muc:8080/
      ftp_proxy= http://qxnumber:password@proxy.muc:8080/
      no_proxy=localhost,127.0.0.1,localaddress,.localdomain.com,asc-repo.bmwgroup.net
      ,asc.bmwgroup.net,b2b.bmwgroup.net,asc-int3.bmwgroup.net,
      asc-repo-int3.bmwgroup.net,.muc,bmwgroup.net,cc-artifactory.bmwgroup.net,
      cc-github.bmwgroup.net

      HTTP_PROXY= http://qxnumber:password@proxy.muc:8080/
      HTTPS_PROXY= http://qxnumber:password@proxy.muc:8080/
      FTP_PROXY= http://qxnumber:password@proxy.muc:8080/
      NO_PROXY=localhost,127.0.0.1,localaddress,.localdomain.com,asc-repo.bmwgroup.net
      ,asc.bmwgroup.net,b2b.bmwgroup.net,asc-int3.bmwgroup.net,
      asc-repo-int3.bmwgroup.net,.muc,bmwgroup.net,cc-artifactory.bmwgroup.net,
      cc-github.bmwgroup.net

* Install Docker and be able to run it without sudo. The idea of a docker container is having a "box" with everything we need inside it to run the tests against the target.
  * Refer to https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce

  .. code-block:: console

    sudo apt-get update

    sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add –

    Verify that you now have the key with the fingerprint 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88, by searching for the last 8 characters of the fingerprint.

    sudo apt-key fingerprint 0EBFCD88

    	pub   4096R/0EBFCD88 2017-02-22
          	Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
    	uid                  Docker Release (CE deb) <docker@docker.com>

    	sub   4096R/F273FCD8 2017-02-2

    Add stable repository ->
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs)  stable

  * Install Docker Community edition: sudo apt-get install docker-ce

  * Run without sudo and setup system-test container: the 0.1.3 version is the most stable one, HW will be available from 0.1.4.

  .. code-block:: console

      sudo docker login -–username $artifactory_username –-password $artifactory_password cc-artifactory.bmwgroup.net

      cat /etc/group | grep docker

      sudo groupadd docker

      sudo gpasswd -a $USER docker

      Logout and login again

      Download the system-test docker container and tag it with “latest”

      docker pull cc-artifactory.bmwgroup.net/xpad-docker-local/system-test:0.1.3

      docker tag cc-artifactory.bmwgroup.net/xpad-docker-local/system-test:0.1.3 system-test:latest

Workspace structure for xTEE
*****************************

  ::

    xpad
    |-- workspace
        |__ src
        |   |__ build
        |       |__ template
        |           |__ deploy
        |               |__images
        |               |   |__ hpad-a1-64-abl
        |               |__sdk
        |__ testing
            |__ test-framework
            |__ mtee_xpad
            |__ tools
                |__ mgu-can
                |__ mgu_dlt
                |__ reporting

* hpad-a1-64-abl is the folder where xTEE searches for the image to run with qemu:
    * Releases can be found in: ..https://cc-artifactory.bmwgroup.net/artifactory/xpad-releases-local/hpad/1.xx.10/images
    * Unzip the kernels folder.
    * Mount the image using the hpad_appl_image.tar.gz into an ext4 image and name it
          *hpad-a1-64-abl*.ext4
* sdk: folder for the hPAD sdk 8not compulsory int he current version.
    * SDK versions: https://cc-artifactory.bmwgroup.net/artifactory/xpad-releases-local/
    * Unzip the .tar.gz file of the sdk and compile it using:
            bazel build //...
    * Compiling will generate symlinks to the output folders, but Docker doesn't
      understand symlinks so far, so we need to replace the symlinks with the actual files.
      Do this with bazel-out and tools.

* test-framework: mtee testing framework.
    * Checkout here the mtee-testing repo.
    * Inside testing folder:

    .. code-block:: console

      git clone https://asc-repo.bmwgroup.net/gerrit/ascgit098.testing.mtee test-framework
* mtee_xpad: mtee_xpad plugin
    * Inside the testing folder:

    .. code-block:: console

      git clone git@cc-github.bmwgroup.net:xpad/mtee-xpad-plugin.git mtee_xpad
* mgu-can:
    * Inside the tools folder:

    .. code-block:: console

      git clone https://asc-repo.bmwgroup.net/gerrit/ascgit098.testing.mgu-can mgu-can
* mgu_dlt:
     * Inside the tools folder:

     .. code-block:: console

      git clone https://asc-repo.bmwgroup.net/gerrit/ascgit098.testing.mgu-dlt mgu-dlt
* Reporting:
    * Inside the tools folder:

    .. code-block:: console

       git clone https://asc-repo.bmwgroup.net/gerrit/ascgit098.testing.reporting

***********************
Final xTEE installation
***********************
Run the following commands inside the workspace folder to finish the installation and check that no errors occur.

.. code-block:: console

    docker run --rm -t --privileged -v $(pwd):/workspace system-test:latest \
    /bin/bash -c "install_test_tools.sh \
    && cd /workspace/testing/test-framework \
    && mkdir -p junit-reports \
    && nosetests tests/unittests"

    docker run --rm -t --privileged -v $(pwd):/workspace system-test:latest \
    /bin/bash -c "install_test_tools.sh \
    && cd /workspace/testing/test-framework \
    && pep8"

    docker run --rm -t --privileged -v $(pwd):/workspace system-test:latest \
    /bin/bash -c "install_test_tools.sh \
    && cd /workspace/testing/test-framework \
    && pylint --rcfile=setup.cfg mtee"
