Debian Package Build from PPA HOWTO
===================================

First you need to install some dependencies::

  $ sudo apt-get install devscripts build-essential
  $ sudo apt-get build-dep af-alg

Next, add the PPA source url to a new file under sources.list.d, and then uncomment the *deb-src* lines in your parent sources.list file.

I currently maintain two PPA's, a crypto PPA and a desktop PPA, so we'll need to use the first one at https://launchpad.net/~nerdboy/+archive/ubuntu/ppa-crypto

1. On the page of the PPA, look for the heading that reads "Adding this PPA to your system" and click the "Technical details about this PPA" link.

2. Use the Display sources.list entries drop-down box, and look for the line starting with **deb-src**

3. You'll see that the text-box directly below reads something like this::

  deb-src http://ppa.launchpad.net/nerdboy/ppa-crypto/ubuntu trusty main

4. Now you need to add this line to your list of repos; the following will create a new config file for apt. As root:

  # echo "deb-src http://ppa.launchpad.net/nerdboy/ppa-crypto/ubuntu trusty main" > /etc/apt/sources.list.d/crypto-ppa.list

5. Now look for the repository signing key, located on the same page. Here it should read::

  Signing key:
  1024R/7774ED19 (What is this?) 

Add the 7774ED19 part in the middle, which is the key ID, to your list of trusted apted sources:

  $ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 7774ED19

6. Now you can install the package source and build the package as a normal user (note you may want to do all this inside a deb-src or tmp directory).  Next run:

  $ sudo apt-get update

7. If your distro is Ubuntu, you're all set, just run the command in step 8 with the `--build` flag (the ppa build config is already done).  If on Debian, you'll need to download the source first, then edit the debian/control file in the source tree.  Use your favorite editor and change the Ubuntu versions to Debian-compatible ones.  Open the debian/control file and change this line::

  Build-Depends: debhelper (>= 9), libssl-dev (>= 1.0.1f-1ubuntu2), pkg-config (>= 0.26-1ubuntu4)

to something like this::

  Build-Depends: debhelper (>= 9), libssl-dev (>= 1.0.1t-1+deb8u2), pkg-config (>= 0.26-1)

8. For Debian, fetch the package source and make the change above.  Run the following command to get the source:

  $ apt-get source af-alg

9. After editing the control file, build the binary package:

  $ debuild -b -uc -us

10. If all goes well, you can install the resulting binary package.  You should have a package in the current directory called something like af-alg_0.0.1c-1_mips.deb which you can install with the following command:

  $ sudo dpkg -i af-alg_0.0.1c-1_mips.deb

