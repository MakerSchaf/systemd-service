
# Starting a service when the Raspberry boots

One of the frequently asked question is how a service or program (the
software part of your project) can be started when the Raspberry
boots.  There's a [guide from Thagrol on Github][1] and there's not
much to add.

Although I don't like _systemd_ very much (short version: for most of us
learning _systemd_ is a waste of time) I see it as the go-to for
service-like programs.  Other recommended ways of starting service-like
things are sometimes _rc.local_ or _cron_.

_systemd_ might be the correct correct way but it is also more difficult
than the other solutions.  Thagrol's guide lists these disadvantages:

 1. It’s more complex than cron or rc.local.
 2. Creating or modifying service files must be done as root or with
    sudo.
 3. The default user and group are root.
 4. The default CWD is /
 5. By default all output is discarded.
 6. By default no input is received.

They are true but it is possible to simplify things.  Of course, not
all: _sudo_ will be required (but we do that anyway all day long) and
you can't send input from the console to the service.  For the other
items I have created [_systemd-service_][2]: a shell script to make
using _systemd_ a little bit easier.

> For impatient readers: running `./systemd-service` without
> parameters gives a brief overview about options and operations.

### Installation

_systemd-service_ needs no installation.  You copy to the directory
where your project lives.  That's all.  Let's assume that name off the
directory is `test-project`.

    $ pwd
    /home/pi/work/test-project

Please notice that _systemd-service_ is not made to be installed
system-wide in e.g. _/usr/bin_.


### The unit file

Every _systemd_ service needs a unit file.  This configures what needs
to be done and how this needs to be done to start the service.  There
is documentation available explaining how to write unit files but
_systemd-service_ creates that file for you.  You can inspect the file
that would be installed with `systemd-service show`:

    $ ./systemd-service show
    [Unit]
    # unit file: test-project.service
    # unit path: /etc/systemd/system/test-project.service
    Description=test-project service
    After=network-online.target
    
    [Service]
    Type=simple
    ExecStart=/home/pi/work/test-project/start.sh
    # ExecStopPost=
    User=pi
    Restart=yes
    
    [Install]
    WantedBy=multi-user.target

The first two comment lines in the `[Unit]` section list the unit's
file- and pathname.  By default _systemd-service_ uses the directory's
name for it and you should make sure that it doesn't conflict with
other services (yours or system).

 - The service is started after _systemd_ has brought up the network,
   which I think is a reasonable default.

 - The service type (`[Service]` section) is `simple` which means that
   _systemd_ shall simply run the command line from `ExecStart`.

 - `ExecStart` defines the command that _systemd_ executes to start
   the service.  We will come back to that later.

 - The service will be run as user `pi`.  Change that to `root` if
   required or _sudo_ from your service.

 - The final `Restart=yes` means that the service is automatically
   restarted when it terminates.  (That's the usual standard.)

I have tried to choose reasonable defaults but of course you might
want or need to change them.  There are command line options for that:

 : **-c** _cmd_ :
  changes the `ExecStart` command to _cmd_.

 : **-n** _name_ :
  sets the service's name.

 : **-p** _cmd_ :
  sets the `ExecStopPost` command.

 : **-r** yes | no:
  makes the service restart when it terminates.

 : **-u** :
  changes the user running the service.

So, if you run the command

    $ ./systemd-service -c /path/to/my-program -n my-service \
        -u www-data -p "/bin/echo no example available" show

you will get the unit file

```
    [Unit]
    # unit file: my-service.service
    # unit path: /etc/systemd/system/my-service.service
    Description=my-service service
    After=network-online.target
    
    [Service]
    Type=simple
    ExecStart=/path/to/my-program
    ExecStopPost=/bin/echo no example available
    User=www-data
    Restart=yes
    
    [Install]
    WantedBy=multi-user.target
```

Command line options are cool but here I strongly recommend to modify
parameters in the _systemd-service_ script.  First of all, you will
not break anything with that.  The shell script is supposed to live
in your local project directory.  It's not a system-wide script.  Then
there's the advantage that modifying the script makes the command line
options permanent:  Imagine you set some options and come back in one
or two months.  Will you remember the options you used?  Or will you
have written them down?  Putting them in the script is the recommended
way to document.

The _systemd-service_ script starts with the possible options:

```
    #
    # Define permanent options/defaults here.
    #
    
    # The service name.  Default: directory name.
    NAME=
    
    # The service description.
    DESCRIPTION=
    
    # User running the service. Default: the current user.
    USER=
    
    # Command to run after the service was stopped.
    POST_STOP=
    
    # Command to execute.  Default: 'start.sh'
    CMD=
    
    # Set to 'yes' if the service should be restarted by systemd
    # when it exits.
    RESTART=
```

The parameters work as described (and are be overwritten by command line
options).


### Starting the program

So far we can construct a unit file and we will see later how it is
installed.  Remember the `ExecStart` option? It points to the file
_start.sh_ but you may not have it.  That's not a problem because
_systemd-service_ creates also this script for you with the `init`
operation.

    $ ./systemd-service init
    -rwxr-xr-x 1 pi pi 1029 Dec 15 19:45 /home/pi/work/test-project/start.sh
    Start script is /home/pi/work/test-project/start.sh.

_start.sh_ is a short shell script that deals with some of _systemd_'s
aspects and provides a better environment.  Let's take a quick tour.

The first thing _start.sh_ does is to change into the project
directory (= the directory where _start.sh_ is).

    dir=`dirname $0`
    cd $dir  ||  exit 1

The directory name is stored in the $BASE variable and added to the
$PATH variable to allow running scripts or executables without giving
the directory name.

    export BASE="$PWD"
    PATH="$BASE:$PATH"

Then, _start.sh_ makes sure that the $HOME variable is properly set.
_systemd_ sets the configured user id but that's all.  Everything else
must be done somewhere else and the $HOME variable is one of the most
important things.

    export HOME=/home/$USER

After that, a unique session key is created for whatever purpose.  It
is based on the system's date and time but notice that if _start.sh_
is run during boot the time is often not properly set and points to
the past.

    # Set a unique key for this run.
    export SESSION_KEY=`date +'%Y%m%d-%H%M%S'`

To capture all following output, a logfile is created using the session
key from above and all output (normal and error) is redirected there.
If you need to debug your project this file is the place to go.

    # Redirect all script/program output into
    # a logfile for later inspection (and removal).
    progname=`basename $0 .sh`
    if ! tty -s; then
        LOG=$progname-"$SESSION_KEY".log
        exec >$LOG 2>&1
    fi

Notice that the logfile is not used when you run _start.sh_ manually
from the command line to support interacting with the program to e.g.
debug it.

Logfiles are automatically created and the script must take care that
they are also removed automatically.  This is done by the following
lines and the comment says it all.

    # Keep all logfiles from today plus 6 older logs and
    # remove the others.  "Today" is difficult if the raspi
    # runs without network but you should have access to the
    # relevant log in case off an issue.  Even with network
    # date and time may be not correct because setting time
    # happens usually after the system startup.
    LIST=`ls -1r $progname-*.log |
    	awk '
    		/./ {
    			split($0, x, "-");
    			if (NR == 1)
    				today = x[2];
    			else if (x[2] == today)
    				;
    			else if (n < 6)
    				n++;
    			else
    				print $0 }'`
    if [ "$LIST" != "" ]; then
        rm $LIST
    fi

Then the script writes some debug output to the logfile.  This is to
confirm the current date, directory, username and perhaps most
important the environment variables.  They differ very much from what
you find when working in a normal shell so it might be worth to be
reminded of their values.  This is however not significant and if you
don't like or need it you can simple remove that part.

    { date; pwd; whoami;
      echo;
      env | sort;
      echo; }

After everything is set up you start your real project program.
Replace the `echo` statement with whatever you need to your project.

    # Start the real project script.
    echo no script configured.

Not necessary but an additional hint: Prefix the final command with
`exec` to terminate the shell interpreter.  It doesn't make much sense
to have the shell wait for your program when there is nothing to do
after it finished.

_systemd-service_ can put the final command into the script if it is
added to the `init` operation:

    ./systemd-service init busybox httpd -p 2080 -c ./httpd.conf ./data

would insert

    exec busybox httpd -p 2080 -c ./httpd.conf ./data

as last line into _start.sh_ just as expected.


### Installing the service

Ok, we have seen the unit file and a shell script that creates a basic
environment to run your project.  How is that installed as a service?
That's the easy part:

    $ sudo ./systemd-service install

creates the unit file in _/etc/systemd/system_ (i.e., the unit path
you saw in the output from `./systemd-service show`) and enables it.
The service will then start on next reboot or when you run

    $ sudo systemctl start test-project

right now.  Of course, replace `test-project` with the name of your
service.  (But you knew that already, right?)

One advantage of _systemd_ against _rc.local_ or _cron_ is that you
can use _systemctl_ to start, stop or restart your service with the
exact environment it would get when run at boot time.  Alternatively,
you can also stop the service and run it directly on the console

    $ sudo systemctl stop test-project
    $ ./start.sh

to e.g. interact with the program for debugging if you need to.


### Uninstalling the service

To uninstall your service you run

    $ sudo ./systemd-service uninstall

Do you remember the above suggestion to put options into the script
and not on the command line?  Here's the use case: When you install
the service and assign a service name with the **-n** option you must 
give the same name when you uninstall it.  Do you still remember the
option from one or two months ago?  Here, putting the name (if you
can't or don't want to use the default) directly into the script is
really handy.


### Final note

That was a long description of _systemd-service_. Run that script
without parameters to get a short overview about it's operations:

 - **init** creates the shell script to start your project,
 - **show** displays the "would be" unit file,
 - **install** creates and installs the unit file, and
 - **uninstall** removes it from the system configuration.
 .

Run `./systemd-service` to get a brief overview.


 [1]: https://github.com/thagrol/Guides/blob/main/boot.pdf
 [2]: systemd-service (systemd-service shell script)
