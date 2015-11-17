# CentOS 7 Simple Kickstart

# Goals:
<p>
<ul>
<li>Create an extremely simple and trimmed down CentOS 7 kickstart environment<br />
...that produces small CentOS 7 images.</li>
<li>Use nearly the same method that a person might use to create this on bare metal servers in a data-center.<br />One could even build servers in a data-center off their laptop if they add a bridged or NAT interface to the Kickstart VM.</li>

<li>Use what Apple provides us on the Mac, plus VirtualBox if it is not already installed.</li>

<li>Avoid requiring sudo or root on the laptop.</li>

<li>Avoid vagrant as it is not repeatable in the data-center.</li>

<li>Learn how kickstart works and what is required to build a simple CentOS 7 machine.</li>
</ul>
</p>
___

# Requirements:
<p>
<ul>
<li>Mac Laptop with VirtualBox installed.</li>

<li>Access to the internet on TCP ports 80 (http), 443 (https) and 873 (rsync)</li>

<li>About 80 GB of free space on your hard drive. <i>Actual usage may be much less depending on number of VM you create.</i></li>

<li>Some type of command line interface.  Most folks use <b>Terminal</b> on Mac.</li>
</ul>
</p>
___

# Steps
<p>
<ul>
<li>Install VirtualBox if you have not done so already.  See virtualbox.org</li>

<li>Clone this repo.  See https://github.com/ for instructions on how to clone repos.<br />
<code>
mkdir -p ~/build/centos7_simple_kickstart/scripts
<br />
git clone https://github.com/ohdns/centos7_simple_kickstart.git
<br />
mv centos7_simple_kickstart/* ~/build/centos7_simple_kickstart/scripts/
<br />
</code>
<br /></li>

<li><u>Edit</u> the .cfg files in ~/build/centos7_simple_kickstart/scripts and replace the SHA512 hashes with your own.<br />
 ... or don't.  The default password for ohadmin and root will be <b>centos7</b><br />
<b>ohadmin</b> will be the automation account; and <i>temporarily</i>, the way you ssh to your VM.<br />
You <b>must</b> also replace the SSH Public Key in the c7 .cfg files with your public key.<br />
Please <b>do not</b> use this hash anywhere that contains sensitive data or build automation.</li>

<li>Create ~/build/centos7_simple_kickstart/www/html/mirror/docker_images and put your docker images in it.<br />This will get mirrored by the first VM you create in step1, which happens to be your yum/kickstart server.</li>

<li><u>Execute</u> <code>~/build/centos7_simple_kickstart/scripts/step1</code> to <br />
Sync the public CentOS and EPEL repo to your laptop<br />
Start up a local apache instance on 192.168.120.1 on a new private network of 192.168.120.0/24<br />
Start up a local rsync daemon on a high port on 192.168.120.1<br />
Kickstart your <i>new kickstart VM server</i><br /><br />
<b>When the first VM starts</b>, you should see a CentOS ISO install screen.<br />
At this point, hit the TAB key, backspace over <i>quiet</i> and type:<br /><br />
<code>cmdline ip=192.168.120.10 netmask=255.255.255.0 ks=http://192.168.120.1:8888/c7_ks.cfg</code><br /><br />
This will build the first VM (the kickstart server role) and will rsync the Yum repos to itself, pulling from your laptop. Get a cup of coffee or tea while this runs.</li>

<li>When the first VM completes building:<br /><br /><u>Execute</u> <code>~/build/centos7_simple_kickstart/scripts/step2 {n}</code> to spin up <i>n</i> number of VM's.<br /><br />Unless you change the step scripts, the VM's will use up to 1.2 GB of ram each. Most laptops do not have more than 16 GB of ram. By all means, fiddle with the memory allocation to see what you can create.</li>
</ul>
</p>
___

# Some Challenges For You!

<p>
<ul>
<li>Keep SELinux enabled, no matter what you plan to install.<br />
Instead of disabling SELinux, read up on setting booleans (<i>setsebool -P boolean or getsebool -a</i>), creating rules, setting file and directory contexts (semanage fcontext)<br />
Confine as many applications as you can, to the least amount of privilages required to run the application.<br />
Use audit2allow, audit2why or grep through /var/log/audit/audit.log to see why something was denied.<br />
<b>As a last resort</b>, set a user or process to <i>unconfined</i> or <i>permissive</i> instead of disabling SELinux.</li>

<li>Prefer a chroot restricted SFTP over unrestricted SSH trusts when feasible.</li>

<li>For custom applications, clearly define:<br />
where the application binaries should be installed. <i>i.e. /opt/application_name/{etc,bin,sbin,var}</i><br />
where the application logs should reside. <i>i.e. /data/application_name/logs</i><br />
where transitory data should reside. <i>i.e. /data/application_name/data</i><br />
or better, <b>use docker!</b> to keep the OS pristine and easy to patch.<br />
and what permissions are required for the application to run and for <b>the right folks</b> to view logs without sudo.</li>
<li>Aside from the Automation account, avoid SSH key trusts when you can. Proper automation is derrived from codified instructions that occur in the data-center, not from a laptop.</li>

<li>Avoid enabling root login via ssh.  In fact, avoid logging into any VM's if you can.<br />
The base configuration belongs in kickstart and customization needs to be done in a configuration management and orchestration system.<br />
All services need to be initialized by a proper startup script when the server OS starts up.</li>

</ul>
</p>
___


# Known Issues and Limitations:
<p>
<ul>
<li>To use NTP from the kickstart server to our laptop, we have to break our own rule one time and use either root or sudo to modify /private/etc/ntp-restrict.conf on our laptop to allow a query.<br />
<code>sudo echo "restrict 192.168.120.0/24" >> /private/etc/ntp-restrict.conf ; sudo pkill -HUP ntpd</code><br /></li>

<li>There is one manual step to create the DHCP/PXE/Yum/Kickstart server.  This is <i>probably</i> ok, since we should not be doing this often.</li>

<li>This method currently lacks end-to-end validation of the RPM GPG signatures.  (Work In Progress)</li>

<li>You will need to manually add your SSH public key in the ~/build/centos7_simple_kickstart/scripts/c7*cfg files so that you may SSH as the Automation user (ohadmin).</li>

<li>There is currently a bit of customization in the kickstart files.  Much of this will be moved into automation as this document and repo evolve to include managing these hosts with automation and orchestration services.</li>

<li>I renice this script to avoid blocking any work you are doing. This means if your laptop is under a heavy load, this kickstart build process may go very slow by design.</li>

<li>Since we are not using Vagrant, you would have to script startup/shutdown yourself.<br /><br />

<b>Examples:</b><br /><br />
<code>PATH=${PATH}:/Applications/VirtualBox.app/Contents/MacOS;export PATH</code><br /><br />
Clean / Graceful Power Off:<br />
<code>VBoxManage controlvm c7_small_1 acpipowerbutton</code><br /><br />
Power On:<br />
<code>VBoxManage startvm c7_small_1</code><br /><br />
Power On Headless (No GUI / console):<br />
<code>VBoxManage startvm --type headless c7_small_1</code><br /><br />
</li>
</ul>
</p>
___
<p><b><br />Disclaimer: This repo contains scripts that are for educational purposes only.  This repo contains default passwords and ssh public keys that must not be used anywhere beyond VirtualBox on your laptop for educational purposes only. DO NOT use this to deploy a production environment unless you have properly changed all defaults, changed settings to reflect that which is approved for your environment and have properly tested this in a lab and staging area that matches your live environments.  The author of these scripts assumes no responsibility for damages to persons or property.<br /><br /></b><br /></p>
___
<i>20151117</i>
