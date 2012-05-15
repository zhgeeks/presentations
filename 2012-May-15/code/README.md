# Auto-scaling exercise

The artifacts in this repository serve the purpose to configure and exercise an auto-scaling setup in amazon's cloud:

 - `cloud-config.txt` uses [Ubuntu's CloudInit](https://help.ubuntu.com/community/CloudInit) to configure individual instances to listen
   and react to the following URLs: `localhost:30000/ping` and
   `localhost:30000/work`. The latter is designed to create load on
   the instance so that the auto-scaling behaviour can be exercised.
  - `nginx.conf` is the nginx configuration needed on the instance
  - `server.py` is the backend script that reacts to 'ping' and 'work'
    requests. The latter creates the load.
 - `configure-as` is the bash script that should be used to set
    up or tear down the auto-scaling configuration
 - `client` is a bash script that simulates a client
 - `runtest` is a bash script that runs [4 clients like this](http://instagr.am/p/J5enj2LXJE/) in
   order to exercise the auto-scaling setup. Once it terminates it will
   write the following files:
   - `last_run.alarms`: the auto-scaling alarm history
   - `last_run.instances`: a snapshot of the instances and their states
     while the test was in progress
   - `last_run.stats`: a snapshot of the CPU load statistic while the test
     was running
 - `track_instances`: the script that is actually producing the
   `last_run.instances` file.

In order to run these tools you need to set the following environment variables:

 - `EC2_ACCESS_KEY`
 - `EC2_SECRET_ACCESS_KEY`
 - `EC2_PRIVATE_KEY`
 - `EC2_CERT`

Please see e.g. the [EC2 starter guide](https://help.ubuntu.com/community/EC2StartersGuide) for an explanation of the above.

# Prerequisites
In order to make use of the artifacts in this repository you will need
to have the following tools installed and configured properly:
 1. [Amazon EC2 API Tools](http://aws.amazon.com/developertools/351)
 1. [Auto Scaling Command Line Tool](http://aws.amazon.com/developertools/2535)
 1. [Amazon CloudWatch Command Line Tool](http://aws.amazon.com/developertools/2534)

# Testing the solution

The solution at hand comes with its own test script (with [this default client timing](http://instagr.am/p/J5enj2LXJE/)) and can be tested as follows:

 1. Check whether the auto-scaling group is already running, if yes `as-describe-auto-scaling-groups asg-zhgeeks` will show:
    <pre><code>AUTO-SCALING-GROUP  asg-zhgeeks  lc-zhgeeks  eu-west-1a  zhgeeks  1  2  1
    INSTANCE  i-e14bbda9  eu-west-1a  InService  Healthy  lc-zhgeeks
    TAG  asg-zhgeeks  auto-scaling-group  name  zhgeeks  true
    </code></pre>

 1. Otherwise create it via `./configure-as --base-name=zhgeeks` and
    wait a little until `as-describe-auto-scaling-groups asg-zhgeeks` shows:
        
    <pre><code>AUTO-SCALING-GROUP  asg-zhgeeks  lc-zhgeeks  eu-west-1a  zhgeeks  1  2  1
	INSTANCE  i-47897f0f  eu-west-1a  InService  Healthy  lc-zhgeeks
	TAG  asg-zhgeeks  auto-scaling-group  name  zhgeeks  true
    </code></pre>

    the `InService` bit is important -- it means that the base instance
    in the auto-scaling group is up and ready for business.
 1. now you can run the actual test as follows: `./runtest`. It will take around 14 minutes and 4 terminal windows titled `A`, `B`, `C` and `D` will be opened in the progress. If you are running a system without the `gnome-terminal` program you will need to tweak `runtest` slightly to change the terminal command.
    If all goes well you should see something like:

    <pre><code>Tue May 15 16:14:48 CEST 2012
    started client A, it will run for 840 seconds; sleeping 120 seconds before starting client B

    Tue May 15 16:16:48 CEST 2012
    started client B, it will run for 480 seconds; sleeping 120 seconds before starting client C

    Tue May 15 16:18:48 CEST 2012
    started client C, it will run for 480 seconds; sleeping 120 seconds before starting client D

    Tue May 15 16:20:48 CEST 2012
    started client D, it will run for 120 seconds

    Tue May 15 16:22:48 CEST 2012
    client D terminated

    Tue May 15 16:22:48 CEST 2012
     5237 pts/9    Ss+    0:00 /bin/bash ./client 840 AAA
     5689 pts/10   Ss+    0:00 /bin/bash ./client 480 BBB
     6397 pts/16   Ss+    0:00 /bin/bash ./client 480 CCC
     7389 pts/17   Ss+    0:00 /bin/bash ./client 120 DDD

     ...
    </code></pre>
    However, the main "evidence" that the auto-scaling worked is in the `last_run.instances` file which should resemble the following:

    <pre><code>Tue May 15 16:21:18 CEST 2012
    INSTANCE  i-47897f0f  asg-zhgeeks  eu-west-1a  InService  HEALTHY  lc-zhgeeks
    Tue May 15 16:21:36 CEST 2012
    INSTANCE  i-47897f0f  asg-zhgeeks  eu-west-1a  InService  HEALTHY  lc-zhgeeks
    INSTANCE  i-75d82e3d  asg-zhgeeks  eu-west-1a  Pending    HEALTHY  lc-zhgeeks
    ...
    Tue May 15 16:22:29 CEST 2012
    INSTANCE  i-47897f0f  asg-zhgeeks  eu-west-1a  InService  HEALTHY  lc-zhgeeks
    INSTANCE  i-75d82e3d  asg-zhgeeks  eu-west-1a  InService  HEALTHY  lc-zhgeeks
    ...
    Tue May 15 16:25:49 CEST 2012
    INSTANCE  i-47897f0f  asg-zhgeeks  eu-west-1a  Terminating  HEALTHY  lc-zhgeeks
    INSTANCE  i-75d82e3d  asg-zhgeeks  eu-west-1a  InService    HEALTHY  lc-zhgeeks
    Tue May 15 16:26:07 CEST 2012
    INSTANCE  i-75d82e3d  asg-zhgeeks  eu-west-1a  InService  HEALTHY  lc-zhgeeks
    </code></pre>
    The above shows how the `i-75d82e3d` instance is added to the load balancer and goes from `Pending` to `InService` to meet the demand. Once the demand goes back to normal the auto-scaling terminates the (original) `i-47897f0f` instance and we see it going from `InService` to `Terminating` and disappearing altogether eventually.

    Last but not least you may want to take a peek at the `last_run.stats` file  which shows the `CPUUtilization` statistics for the instances involved in the test:

    <pre><code>test run started at 2012-05-15T14:14:00Z

    i-47897f0f
    2012-05-15 14:14:00  2.95   Percent
    2012-05-15 14:15:00  12.67  Percent
    2012-05-15 14:16:00  13.56  Percent
    2012-05-15 14:17:00  24.59  Percent
    2012-05-15 14:18:00  26.67  Percent
    2012-05-15 14:19:00  35.93  Percent
    2012-05-15 14:20:00  37.67  Percent
    2012-05-15 14:21:00  47.21  Percent
    2012-05-15 14:22:00  45.08  Percent
    2012-05-15 14:23:00  22.0   Percent
    2012-05-15 14:24:00  17.0   Percent

    i-75d82e3d
    2012-05-15 14:23:00  24.59  Percent
    2012-05-15 14:24:00  17.35  Percent
    2012-05-15 14:25:00  16.0   Percent
    2012-05-15 14:26:00  22.6   Percent
    2012-05-15 14:27:00  13.49  Percent
    </code></pre>

   If you are really *curious* take a look at the `last_run.alarms` file
   as well ;-)

 1. Finally, the following command should be used to tear down the auto-scaling configuration:
 
    <pre><code>./configure-as --base-name=zhgeeks --action=teardown</code></pre>

# Handy commands while testing
 - view the alarm states and actions:
    <pre><code>while ((1)); do clear; mon-describe-alarm-history | head -n 8; sleep 30; done</code></pre>
 - view the CPU load on the auto-scaling group:
    <pre><code>while ((1)); do clear; mon-get-stats --metric-name CPUUtilization --namespace AWS/EC2 --dimensions AutoScalingGroupName=asg-zhgeeks --statistics Average --start-time 2012-05-15T18:59:00Z; sleep 29; done</code></pre>
