# PlaneFence
Collection of scripts using Socket30003 logs to create a list of aircraft that fly low over your location
Copyright 2020 by Ramon F. Kolb - Licensed under GPL3.0 - see separate license file.

For an example, see http://ramonk.net/planefence


## Attributions, inclusions, and prerequisites

1. You must have a Raspberry Pi with a working version of dump1090, dump1090-fa, dump1090-mutability, or the equivalent dump978 versions installed. If you don't have this, stop right here. It makes no sense to continue unless you understand the basic functions of the ADSB receiver for Raspberry Pi
2. The scripts in this repository rely on [dump1090.socket30003](https://github.com/tedsluis/dump1090.socket30003), used and distributed under the GPLv3.0 license. 

What does this mean for you? Follow the installation instructions and you should be good :)

## Installation

Follow the following steps in order.

### Prerequisites
1. These instructions assume that you already have a relatively standard installation of dump1090, dump1090-fa, dump1090-mutability, or the equivalent dump978 on your Raspberry Pi. If you don't have this, Google "FlightAware feeder", "Radarbox24 feeder", or something similar to get started. Get a RPi 3B+ or 4, an RTL-SDR dongle, an antenna, and come back here when you can access a map with aircraft flying over your home.
2. You should feel somewhat comfortable installing and configuring software on a Raspberry Pi or a similar Linux system using Github. You will be making modifications to your system, and there is a chance you screw things up. You do so at your own risk.
3. You should have GIT installed on your Raspberry Pi. If you don't, you can add it by typing the following on the command line: `sudo apt install git` 

### Install Dump1090.Socket30003
`Dump1090.Socket30003` collects and stores data about all aircraft within reach in CSV files. We will use these CSV files to extract data about aircraft that fly over our location. Here's how.

To install `Dump1090.Socket30003`, [go here](https://github.com/tedsluis/dump1090.socket30003) and follow the installation instructions from the start **UP TO INCLUDING** the section about adding a [Cron Job](https://github.com/tedsluis/dump1090.socket30003#add-socket30003pl-as-a-crontab-job).
Make sure to check that your lat/long has been correctly set to your approximate location in the `[common]` section of `socket30003.cfg`. If they aren't, your PlaneFence won't work.

If you want to use PlaneFence as-is, then the instructions below will assume that you DON'T change the location or format of the log files. This means, that they are written as `/tmp/dump1090_127_0_0_1-yymmdd.txt` and `....log`.

### Install the scripts from this repository
1. Clone the repository. Log into you Raspberry Pi and give the following commands:

```
cd
mkdir git
cd git
git clone https://github.com/kx1t/planefence.git
cd planefence
```

### Make a planefence directory in your existing HTML directory
Under normal circumstances, your FlightAware or dump1090 maps are rendered in this directory:
`/usr/share/dump1090-fa/html/` or `/usr/share/dump1090/html/`
Find them - figure out which one you actually use, and then do this:

```
sudo mkdir /usr/share/dump1090-fa/html/planefence
sudo chmod a+rwx /usr/share/dump1090-fa/html/planefence
```

Remember the location of your **planefence directory** . You will need to use it a few times below. For the rest of the installation instructions, we're assuming it is `/usr/share/dump1090-fa/html/planefence`. You will have to substitute your **planefence directory** name if it is different.

### Copy the utilities to the socket30003 directory
If you followed the `socket30003` install instructions to the letter and you didn't change any directories, then `socket30003` is installed in `/home/pi/socket30003`. We'll copy the scripts there.

```
cp scripts/* /home/pi/socket30003
```

### Install the Python dependencies
Planefence uses Python2.7, which comes standard with your Raspberry Pi on Stretch and Buster. However, there are a few modules that we need to install.
Type the following:

```
sudo apt update
sudo apt upgrade
sudo apt install python-pip python-numpy python-pandas python-dateutil
sudo pip install tzlocal
```

### Edit the script
If all the directories and file names exactly match up with what we wrote above, you can skip this step. If not, then let's make sure that the script can still find everything.

```
cd ~/socket30003
nano planefence.sh
```

- Go to the lines between the dashed separators
- `OUTFILEDIR` contains your *planefence directory* name. If you have a different name, change it there
- `PLANEFENCEDIR` contains the directory name where planefency.py is located. If you followed the instructions above, you won't need to change this.
- `MAXALT` contains the altitude ceiling in ft. `MAXALT=5000` means that only planes that are 5000 ft or lower are tracked
- `DIST` contains the radius around your station in (statute) miles. It relies on your location to be set accurately in `socket30003.conf` as described in the setup instructions for that software package.
- `LAT` and `LON` should be set to your approximate Latitude and Longitude. Note - these parameters ONLY affect the position shown on the web page. It does NOT recalculate the distance from your station for purposes of determining if an entry is close to your home. That should be set in `~/socket30003/socket30003.cfg` and will only change entries pro-actively.

### Create a cron job
CRON is a Linux utility to schedule running a program at regular intervals. Once you execute the following command, the system will run
`planefence` every 2 minutes. That way, your website is never more than 2 minutes out of date.

```
sudo cp ~/git/planefence/cron/planefence.cron /etc/cron.d/planefence
```

Note that this will take a reasonably high amount of processing power and disk I/O. If your don't mind a less frequent interval
to update your PlaneFence website, you can change it as follows:

```
sudo nano /etc/cron.d/planefence
```

By default, this document shows you the following line:
`*/2 * * * * root /home/pi/planefence/planefence.sh 2>&1`

The first few characters `*/2` indicate that the script should be run every 2nd minute. You can change this frequence, just pick the appropriate line below and change the text in your document to match it.
Note -- only pick ONE of the lines below. Don't copy all of them - that would make no sense!
```
*/5 * * * * root /home/pi/planefence/planefence.sh 2>&1  # every 5 minutes
*/10 * * * * root /home/pi/planefence/planefence.sh 2>&1 # every 10 minutes
*/30 * * * * root /home/pi/planefence/planefence.sh 2>&1 # every 30 minutes
0 * * * * root /home/pi/planefence/planefence.sh 2>&1    # every hour (on the hour exactly)
```
Note: [Here's a handy website](https://crontab.guru/) that allows you to determine what to set the Crontab to for the frequency you want

Once done, exit with CTRL-o (to save your changes) CTRL-x

## catchup.sh

***Note this is a work in progress - this script needs adapting and probably won't (yet) work in its current state***

This script will do a "catch-up" run. It will iterate through all `/tmp/dump1090*.txt` files and create heatmaps for them.
If you changed any directories, please make sure to update this script to reflect this.
The script must be run from the same directory where `planefence.py` is located.
It takes no command line arguments. Use it simply as:

```
./catchup.sh
```

# Seeing your PlaneFence page
Once you have rendered at least 1 PlaneFence, you can find it at `http://<address_of_rpi>/planefence`.
Replace `<address_of_rpi>` with whatever the address is you normally use to get to the SkyAware or Dump1090 map.
For reference, see (http://ramonk.net/planefence).

# Optional - Tweeting Your Updates
Once you have PlaneFence completely up and running, you can add an option to send a Tweet for every overflying plane.
The setup of this is a bit complicated as you will have to register your own Twitter Developer Account, and get a 
App Key for your application.
Luckily, I wrote detailed installation instructions that can be accessed here:
https://github.com/kx1t/planefence/blob/master/README-twitter.md

If you want to see an example of how this works, go here: https://twitter.com/PlaneBoston

# Known Issues
- The history of the map goes wild if there has been no plane within your filter as of today. I'll see if we can fix this soon. Note that once there's at least 1 plane in the area, the table will render correctly.
- The script isn't very friendly to changes to the directory naming conventions. Working on that too.
- The catch-up script has the paths hardcoded. This is another thing that needs changing.
- allow setting `LAT` and `LON` in the script to actually recalculate the distance from the station. Right now, it's only used for cosmetic purposes in rendering the webpage.

# Summary of License Terms
This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <https://www.gnu.org/licenses/>.

