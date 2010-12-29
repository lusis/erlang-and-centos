# CentOS 5.5 Erlang Applications
After spending more time than I wanted attempting to deal with building RPMS of various erlang applications on CentOS 5, I decided to skip packaging all together. The premise is that these applications will be managed from some CM tool like puppet or chef which will handle creating application users and writing configuration files. This just gets workable binaries on the system.

## Why not a custom RPM?
So why did I do it instead of building a custom RPM? Look at this list of requirements:

	* Riak - Erlang R13B04 minimum
	* CouchDB 1.0.1 - Curl 7.2 minimum

Why are those a problem?
	* EPEL only ships with Erlang 12.
	* There are no Curl 7.2 packages for CentOS 5.5 because it breaks the OS packages.
	* Catch-22

There have been various efforts to resolve all of this but I wanted a unified stack. If I wanted to run CouchDB on the same box as Riak, I damn well should be able to do so. Is it likely? Actually, yes...for our developers to test.

For the record, RabbitMQ RPMS appear to work with the EPEL Erlang packages but again, unified stack. I WILL need to be running CouchDB and RabbitMQ on the same box for my Vogeler server.
I could have swapped to Ubuntu 10.04.1 LTS but I'm not ready to start spreading out my distro usage just yet.

So why not just use chef/puppet to build on the server directly? Idemopotence. That and spin up time. I'm building a system to autoscale itself here. I don't want to wait any longer than I have to for new instances.

# Setup build environment
*Please don't run these commands blindly. This is a log of what I did. I might have missed something.*
For this process, I spun up two VMs of CentOS 5.5 - i386 and x86_64. These are my build hosts and nothing more.

	mkdir /usr/src/build
	cd /usr/src/build
	wget http://download.fedora.redhat.com/pub/epel/5/i386/epel-release-5-4.noarch.rpm
	rpm -ivh epel-release-5.4.noarch.rpm
	yum install js-devel libicu-devel libtool gnutls-devel libidn-devel libssh2-devel nss-devel openssl-devel gcc gcc-c++ automake autoconf make mercurial openldap-devel python-simplejson libxslt xmlto -y

# Grab packages
These are the tarballs from each vendor for the applications I need.

	wget http://www.erlang.org/download/otp_src_R13B04.tar.gz
	wget http://downloads.basho.com/riak/riak-0.13/riak-0.13.0.tar.gz
	wget http://curl.haxx.se/download/curl-7.21.3.tar.gz
	wget http://mirror.its.uidaho.edu/pub/apache//couchdb/1.0.1/apache-couchdb-1.0.1.tar.gz
	wget http://www.rabbitmq.com/releases/rabbitmq-server/v2.2.0/rabbitmq-server-2.2.0.tar.gz

# Build it all

## Build erlang
Build us some erlang. These options are taken from the couchone repo on github with some small changes. I don't need termcap and I don't need java support.

	tar -zxvf otp_src_R13B04.tar.gz
	cd otp_src_R13B04
	CFLAGS="--no-strict-aliasing" ./configure --prefix=/opt/erlang --enable-shared-zlib --without-javac --without-termcap --enable-smp-support --enable-hybrid-heap --enable-threads --disable-hipe --enable-kernel-poll --enable-dynamic-ssl-libs
	make && make install
	export PATH=/opt/erlang/bin/:$PATH

## Build riak
Now that we have a workable erlang install, we can build the applications.

	cd /usr/src/build
	tar -zxvf riak-0.13.0.tar.gz
	cd riak-0.13.0
	make
	make test
	make rel
	cd rel
	cp -R riak /opt/

## Build Curl
Stupid CouchDB people. Guess you don't care about RHEL customers with the fucking stupid curl requirement? These command line options were taken from a nifty spec file for CouchDB. We're building a static libcurl to compile directly into Couch.

	cd /usr/src/build
	tar -zxvf apache-couchdb-1.0.1.tar.gz
	tar -zxvf curl-7.21.3.tar.gz
	cd curl-7.21.3
	mkdir /usr/src/build/apache-couchdb-1.0.1/result
	./configure --disable-dependency-tracking --disable-shared --enable-static --prefix=/usr/src/build/apache-couchdb-1.0.1/result --libdir=/usr/src/build/apache-couchdb-1.0.1/result/lib
	make && make test && make install


## Build CouchDB
Now we can build CouchDB. Options and exports are from the same spec file I mentioned above.

	cd /usr/src/build
	cd apache-couchdb-1.0.1/
	export PKG_CONFIG_PATH="result/lib/pkgconfig/:$PKG_CONFIG_PATH"
	export PATH="result/bin/:/opt/erlang/bin/:$PATH"
	CFLAGS="-I /opt/erlang/lib/erlang/usr/include -L/usr/src/build/apache-couchdb-1.0.1/result/lib" ./configure --disable-dependency-tracking --prefix=/opt/couchdb
	make && make check && make install

## Build rabbitmq
As I said, You don't HAVE to build RabbitMQ this way. It supports an RPM install with the EPEL packages of Erlang. I'm just going for consistency. We're also just installing a shitload of plugins off the bat for testing. You probably don't need them all and should manage them with your CM tool since plugins are simply a .ez file installed in the plugins directory.

	cd /usr/src/build
	tar -zxvf rabbitmq-server-2.2.0.tar.gz
	cd rabbitmq-server-2.2.0
	make && make install TARGET_DIR=/opt/rabbitmq SBIN_DIR=/opt/rabbitmq/bin MAN_DIR=/opt/rabbitmq/man
	cd /opt/rabbitmq/plugins
	for i in rabbit_stomp amqp_client rabbitmq-management-agent mochiweb webmachine rabbitmq-mochiweb rabbitmq-management rabbitmq-shovel; do wget http://www.rabbitmq.com/releases/plugins/v2.2.0/$i-2.2.0.ez; done

# Manual testing
Where it was provided, I ran unit tests while building the packages. I could not get erlang itself to run its tests.
These are some sanity checks you can do before bundling it all up for pushing around.

## Riak

	cd /opt/riak/bin
	./riak start
	curl -v http://127.0.0.1:8098/riak/test
	curl -v -X PUT -H "Content-Type: application/json" -d '{"props":{"n_val":5}}' http://127.0.0.1:8098/riak/test
	curl -v http://127.0.0.1:8098/riak/test
	riak stop

## CouchDB

	cd /opt/couchdb/etc/couchdb
	sed -ie 's/;bind_address.*/bind_address = 0\.0\.0\.0/' local.ini
	cd /opt/couchdb/bin
	./couchdb

Now in a browser point (using firefox - don't ask), point your browser to http://<ip of box>:5984/_utils/
From the right hand menu, click on the Test Suites and then run all of them. A few will occasionally fail but should work if you rerun the individual ones.

## RabbitMQ

	cd /opt/rabbitmq/bin
	./rabbitmq-server

You should now be able to point your browser to http://<ip of box>:55672/mgmt/ and log in with guest/guest

Additionally, you can use some language like Ruby or Python to talk to each of the apps respectively.

# "Packaginglol"
Now we "package" everything up.

	cd /opt
	tar -zcvf erlang-R13B04-<arch>-bin.tar.gz erlang
	tar -zcvf riak-0.13-<arch>-bin.tar.gz riak
	tar -zcvf couchdb-1.0.1-<arch>-bin.tar.gz couchdb
	tar -zcvf rabbitmq-2.2.0-<arch>-bin.tar.gz rabbitmq
	sha1sum *.tar.gz > checksums

With the exception of riak, you need to untar the erlang and application tarballs on a destination host of the same arch into /opt/. You'll also need to install some libraries on the boxes that the packages are linked against from building:

	yum install -y js libicu gnutls libidn libssh2 nss python-simplejson

You can repeat your manual tests against the copied tarballs.

# TODO
Write the cookbooks for chef to distribute these and configure them.
Consider how upgrades will work.
Write a cookbook to actually do all of this for you ;)
Create spec files for all of these? Would that be too meta?

