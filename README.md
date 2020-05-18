# OnionPerf

OnionPerf is a utility to track Tor and onion service performance.

OnionPerf uses multiple processes and threads to download random data
through Tor while tracking the performance of those downloads. The data is
served and fetched on localhost using two TGen (traffic generator)
processes, and is transferred through Tor using Tor client processes and
an ephemeral Tor Onion Service. Tor control information and TGen
performance statistics are logged to disk, analyzed once per day to
produce a json stats database and files that can feed into Torperf, and
can later be used to visualize changes in Tor client performance over time.

For more information, see https://git.torproject.org/onionperf

For a dockerized setup, see https://github.com/hiromipaw/onionperf-docker

## Quick deployment instructions

These are the quick deployment instructions for the current Debian stable distribution.

```
sudo apt install git cmake make build-essential gcc libigraph0-dev libglib2.0-dev python3-dev libxml2-dev python3-lxml python3-networkx python3-scipy python3-matplotlib python3-numpy libevent-dev libssl-dev python3-stem tor

git clone https://github.com/shadow/tgen.git
cd tgen
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/home/$USER/.local
make
sudo ln -s ~/tgen/build/tgen /usr/bin/tgen

git clone https://github.com/torproject/onionperf
cd onionperf
sudo python3 setup.py build
sudo python3 setup.py install
```

## Step-by-step installation instructions

Here you can find more detailed instructions for the current Debian stable distribution.

### Get OnionPerf

```
git clone https://git.torproject.org/onionperf.git
cd onionperf
```

### Install System Dependencies

  + **Tor** (>= v0.2.7.3-rc): libevent, openssl
  + **TGen** (Shadow >= v1.11.1): cmake, glib2, igraph
  + **OnionPerf**: python3

The easiest way to satisfy all system dependencies is to use a package manager. TGen is not currently packaged and needs to be built from source.
Note we only provide support for the current Debian Stable distribution.

```
sudo apt install cmake make build-essential gcc libigraph0-dev libglib2.0-dev python3-dev
```

### Install Python modules

  + **OnionPerf** python modules: stem (>= v1.7.0), lxml, networkx, numpy, matplotlib.

#### Option 1: Package Manager

The easiest way to satisfy all system dependencies is to use a package manager.

```
apt install tor libxml2-dev python3-lxml python3-networkx python3-scipy python3-matplotlib python3-numpy python3-stem

```

#### Option 2: pip

Python modules can also be installed using `pip`. The python modules that are
required for each OnionPerf subcommand are as follows:

  + `onionperf monitor`: stem
  + `onionperf measure`: stem, lxml, networkx
  + `onionperf analyze`: stem
  + `onionperf visualize`: scipy, numpy, pylab, matplotlib

You must first satisfy the system/library requirements of each of the python modules.
Note: pip installation is not recommended as software installed by pip is not verified.

```
sudo apt-get install python3-dev libxml2 libxml2-dev libxslt1.1 libxslt1-dev libpng12-0 libpng12-dev libfreetype6 libfreetype6-dev
```

It is recommended to use virtual environments to keep all of the dependencies self-contained and
to avoid conflicts with your other python projects.

```
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip # make sure that you have a recent version of pip
pip install -r requirements.txt # installs all required python modules for all OnionPerf subcommands
deactivate
```

If you don't want to use virtualenv, you can install with:

```
pip install stem lxml networkx scipy numpy matplotlib
```

**Note**: You may want to skip installing numpy and matplotlib if you don't
plan to use the `visualize` subcommand, since those tend to require several
large dependencies.

### Build Tor

**Note**: You can install Tor with apt, although the
preferred method is to build from source. To install using from backports:

```
echo 'deb http://deb.debian.org/debian stretch-backports main' >> /etc/apt/sources.list
apt update
apt-get -t stretch-backports install tor
```
Or, if building from source:

```
apt install libevent libevent-dev libssl-dev
```

```
git clone https://git.torproject.org/tor.git
cd tor
./autogen.sh
./configure --disable-asciidoc
make
```

### Build TGen Traffic Generator

The traffic generator currently exists in the Shadow simulator repository,
but we will build TGen as an external tool and skip building both the full
simulator and the TGen simulator plugin.

```
git clone https://github.com/shadow/tgen.git
cd tgen
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/home/$USER/.local
make
ln -s build/tgen /usr/bin/tgen
```

### Build and Install OnionPerf

If using pip and virtualenv (run from onionperf base directory):

```
source venv/bin/activate
pip install -I .
deactivate
```

If using just pip:

```
pip install -I .
```

Otherwise:

```
python3 setup.py build
python3 setup.py install
```

### Run OnionPerf

OnionPerf has several modes of operation and a help menu for each. For a
description of each mode, use:

```
onionperf -h
```

  + **monitor**: Connect to Tor and log controller events to file
  + **measure**: Measure Tor and Onion Service Performance using TGen
  + **analyze**: Analyze Tor and TGen output
  + **visualize**: Visualize OnionPerf analysis results

### Measure Tor

To run in measure mode, you will need to give OnionPerf the path to your custom
'tor' and 'tgen' binary files if they do not exist in your PATH
environment variable.

```
./onionperf measure --tor=/home/rob/tor/src/or/tor \
--tgen=/home/rob/shadow/src/plugin/shadow-plugin-tgen/build/tgen
```

This will run OnionPerf in measure mode with default ports; a TGen server runs
on port 8080. Port 8080 **must** be open on your firewall if you want to do
performance measurements with downloads that exit the Tor network.

By default, OnionPerf will run a TGen client/server pair that transfer traffic
through Tor and through an ephemeral onion service started by OnionPerf. TGen and Tor
log data is collected and stored beneath the `onionperf-data` directory, and other
information about Tor's state during the measurement process is collected from Tor's
control port and logged to disk.

While running, OnionPerf log output is saved to component-specific log files.
Log files for each OnionPerf component (tgen client, tgen server, tor client, tor
server) are stored in their own directory, under `onionperf-data`:

 + `tgen-client/onionperf.tgen.log`
 + `tgen-server/onionperf.tgen.log`
 + `tor-client/onionperf.torctl.log`
 + `tor-client/onionperf.tor.log`
 + `tor-server/onionperf.torctl.log`
 + `tor-server/onionperf.tor.log`

Every night at 11:59 UTC, OnionPerf will analyze the latest results from these log
files using the same parsing functions that are used in the `onionperf analyze`
subcommand (which is described in more detail below). The analysis produces a
`onionperf.analysis.json` stats file that contains numerous measurements and other
contextual information collected during the measurement process. The `README_JSON.md`
file in this repo describes the format and elements contained in the `json` file.

The daily generated `json` files are placed in the web docroot and are
available through the local filesystem in the `onionperf-data/htdocs` directory.
These files can be shared via a web server such as Apache or nginx.

Once the analysis is complete, OnionPerf will rotate all log files; each log file
is moved to a `log_rotate` subdirectory and renamed to include a timestamp. Each
component has its own collection of log files with timestamps in their own `log_rotate`
subdirectories.

You can reproduce the same `json` file that is automatically produced every day while
running in `onionperf measure` mode. To do this, you would run `onionperf analyze` on
specific log files from the `log_rotate` directories. You can also plot the measurement
results from the `json` files by running in `onionperf visualize` mode. See below for
more details.

### Analyze/Visualize Results

OnionPerf runs the data it collects through `analyze` mode every night at midnight to
produce the `onionperf.analysis.json` stats file. This file can be reproduced by using
`onionperf analyze` mode and feeding in a TGen log file from the
`onionperf-data/tgen-client/log_archive` directory and the matching Torctl log file from
the `onionperf-data/tor-client/log_archive` directory.

For example:

```
onionperf analyze --tgen onionperf-data/tgen-client/log_archive/onionperf_2015-11-15_15\:59\:59.tgen.log \
--torctl onionperf-data/tor-client/log_archive/onionperf_2015-11-15_15\:59\:59.torctl.log
```

This produces the `onionperf.analysis.json` file, which can then be plotted like so:

```
onionperf visualize --data onionperf.analysis.json "onionperf-test"
```

This will save new PDFs containing several graphs in the current directory. These include:

#### TGen

- Number of transfer AUTH errors, each client
- Number of transfer PROXY errors, each client
- Number of transfer AUTH errors, all clients over time
- Number of transfer PROXY errors, all clients over time
- Bytes transferred before AUTH error, all downloads
- Bytes transferred before PROXY error, all downloads
- Median bytes transferred before AUTH error, each client
- Median bytes transferred before PROXY error, each client
- Mean bytes transferred before AUTH error, each client
- Mean bytes transferred before PROXY error, each client

#### Tor

- 60 second moving average throughput, read, all relays
- 1 second throughput, read, all relays
- 1 second throughput, read, each relay
- 60 second moving average throughput, write, all relays
- 1 second throughput, write, all relays
- 1 second throughput, write, each relay

### Troubleshooting

While OnionPerf is running, it will log heartbeat status messages to indicate
the health of the subprocesses. It also monitors these subprocesses with
"watchdog threads" and restarts each subprocess if they fail. If more than 10
failures happen within 60 minutes, then the watchdog will give up and exit,
and at that point the heartbeat message should indicate that one of the
subprocesses died.

While running, a number of `Warning` messages might be logged. For example:

```
2016-12-21 12:15:17 1482318917.473774 [onionperf] [WARNING] command
'/home/rob/shadow/src/plugin/shadow-plugin-tgen/build/tgen
/home/rob/onionperf/onionperf-data/tgen-server/tgen.graphml.xml'
finished before expected"
```

This specific warning indicates that the tgen server watchdog thread detected
that the tgen server prematurely exited. The watchdog should have spun
up another tgen server to replace the one that died.

To find out why this is happening you should check the specific component logs
which are all subdirectories of the onionperf-data directory.
In this particular case tgen-server log file revealed the problem:

```
2016-12-20 18:46:29 1482255989.329712 [critical] [shd-tgen-server.c:94] [tgenserver_new] bind(): socket 5 returned -1 error 98: Address already in use
```

The log indicated that port 8080 was already in use by another process.

### Contribute

GitHub pull requests are welcome and encouraged!
