ubuntu 16.04安装mesos
step1, 安装相关依赖
# Update the packages.
$ sudo apt-get update

# Install a few utility tools.
$ sudo apt-get install -y tar wget git

# Install the latest OpenJDK.
$ sudo apt-get install -y openjdk-8-jdk

# Install autotools (Only necessary if building from git repository).
$ sudo apt-get install -y autoconf libtool

# Install other Mesos dependencies.
$ sudo apt-get -y install build-essential python-dev python-virtualenv libcurl4-nss-dev libsasl2-dev libsasl2-modules maven libapr1-dev libsvn-dev zlib1g-dev
step2,从网上下载tar.gz包进行编译
# Change working directory.
$ cd mesos

# Bootstrap (Only required if building from git repository).
$ ./bootstrap

# Configure and build.
$ mkdir build
$ cd build
$ ../configure
$ make
In order to speed up the build and reduce verbosity of the logs, you can append -j <number of cores> V=0 to make.

# Run test suite.
$ make check

# Install (Optional).
$ make install
step3,测试
# Change into build directory.
$ cd build

# Start Mesos master (ensure work directory exists and has proper permissions).
$ ./bin/mesos-master.sh --ip=127.0.0.1 --work_dir=/var/lib/mesos

# Start Mesos agent (ensure work directory exists and has proper permissions).
$ ./bin/mesos-agent.sh --master=127.0.0.1:5050 --work_dir=/var/lib/mesos

# Visit the Mesos web page.
$ http://127.0.0.1:5050

# Run C++ framework (exits after successfully running some tasks).
$ ./src/test-framework --master=127.0.0.1:5050

# Run Java framework (exits after successfully running some tasks).
$ ./src/examples/java/test-framework 127.0.0.1:5050

# Run Python framework (exits after successfully running some tasks).
$ ./src/examples/python/test-framework 127.0.0.1:5050
