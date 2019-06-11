# Cloud-init Analyze Module Kernel Telemetry Changes
Sam Gilson, Anh Vo, Hariharan Jayaraman

# Overview:
Today the customer does not have an easy way of analyzing the end to end performance of provisioning, from the time the system boots up until the time cloud-init finishes provisioning actions. In between the time when the kernel finishes booting and cloud-init begins to provision, many other processes must first be run in order for cloud-init to begin to provision, and it is not consistent from kernel to kernel about when this occurs. Therefore, we have 2 goals with this project. <br />  1)Gather timestamps from when kernel boots, kernel finishes boot, and cloud-init starts and add in functionality to cloud-init analyze in-order to give the customer this data when requested. <br />  2)Make sure this functionality works for all cloud service providers, not just Microsoft’s Azure cloud, covering as many distros/Linux kernels as possible.

# Context:
Currently, it is not convenient for non-experts to gather the data necessary to see when the kernel boots, finishes booting, and when cloud-init begins provisioning. This data is important and can be applied in a large variety of contexts, such as tracking regression of kernel start-up times, and gaining clarity about whether something failing before cloud-init begins is affecting its ability to provision. 

# Gathering Timestamps:
When a Linux VM starts up, there are a few steps that occur before cloud-init begins to provision. Kernel Initialization occurs first, begins when StartContainer finishes, and ends when boot kernel phase completes. User-space Initialization then occurs, which is the remaining time between end of kernel boot and beginning of cloud-init provisioning. Based on the init system being used, different processes will be started in different orders based on dependencies, with the process we want to track being cloud-init. Once cloud-init begins provisioning, it will emit an event to the cloud-init logs that we can obtain. 
With the different init systems available on different Linux distros, there is not one consistent way to get timestamps. We will outline how it can be done in the most popular init system, systemd.
<br />In order to gather the necessary timestamps using systemd, one can run the commands
<br />systemctl show -p UserspaceTimestampMonotonic  
<br />systemctl show clout-init-local -p InactiveExitTimestampMonotonic
<br />to gather the UserSpaceTimestamp and InactiveExitTimestamp. The UserSpaceTimestamp is tracks when the UserSpace has been created, which marks the end of the kernel boot. The InactiveExitTimestamp tracks when the system goes into InactiveExit, marking the beginning of cloud-init’s provisioning. Both timestamps begin counting when the kernel boots, so all it takes to get the times we want is some simple arithmetic. To get the time at which the kernel boots, one can use
<br />kernel_boot = time.time() – cloudinit.util.uptime()
<br />in order to get the timestamp. Util.uptime() tracks when the first signal is received to start boot, so subtracting that from the current time would provide you the time stamp for kernel boot. Therefore, end of kernel boot would be
<br />kernel_finish_boot = kernel_boot + UserSpaceTimeStampMonotonic
<br />and beginning of cloud-init provisioning would be
<br />cloud-init_start = kernel_boot + InactiveExitTimestampMonotonic

# Changes to cloud-init analyze module:
In cloud-init analyze, there are currently 3 different options on how to analyze the clout-init.log that is created by cloud-init at runtime. The first is blame, which orders the timestamps by how long each action that was logged took. The second is show, which presents them in chronological order. The third is dump, which simply dumps the entire log file for you to view. In order to access these functionalites, the user can issue the command:
cloud-init analyze <blame| analyze | show  | boot>
while either choosing blame, show, boot,  or dump. Cloud-init analyze is completely separated from any data source and can always be run after provisioning. Therefore, in order to achieve our goal of keeping this project data source agnostic, this is the perfect place to aggregate this information and give it to the user when requested. To achieve this, there are a few options that can be taken.
1)	Have cloud-init write the timestamps of the kernel boot, kernel end boot, and cloud-init start to the logs using the current reporting system that cloud-init uses. From there, cloud-init analyze can interpret those logs just like it would any other logs and display the data in the 3 forms mentioned above. The only caveat with this is that a proper place in cloud-init must be found to emit these events for the emission to be data source agnostic. 
2)	Add a new parser to cloud-init analyze. As an example, we can add a parser called ‘boot’, so that now users can run the command ‘cloud-init analyze boot’ which will then gather the timestamps and present them to the user. A disadvantage with this is that the data is only on-demand and is not logged anywhere in the system.

# Challenges:
Possibly not all init systems or distros log these timestamps in a consistent way, making it difficult to roll out to all Linux distros. In the end, being both distro and data source agnostic is the goal, but whether this is possible or not is yet to be seen. Another challenge may be the fact that collecting these timestamps in the manner outlined above may cause provisioning performance to drop, as collecting the time stamps is not free.

# Current Work:
So far, for Ubuntu 18.04: latest, gathering this data and presenting it to the user is possible. Both changes to cloud-init analyze have been done, with DataSourceAzure.py emitting these events to the logs on set-up and a new parser called ‘boot’ has been added and can be used to get the timestamps on demand. DataSourceAzure.py emitting the events is clearly not in line with having the project be data source agnostic; this was more done as a test to see whether it was possible to do or not. 

# Supported Linux Distributions:
Cloud-init supports Ubuntu, SLES, RHEL, Open SuSE, Gentoo, FreeBSD, Fedora, Debian, CentOS, arch, and CoreOS. Of these, Ubuntu, SLES, RHEL, OpenSuSE, Fedora, Debian, CentOS, arch, and CoreOS are using systemd, which provides systemctl and the ability to obtain timestamps past a certain distro version. FreeBSD and Gentoo use different init systems that do not have the functionality to get timestamps that indicate the beginning of userspace and cloud-init. Therefore, all the cloud-init supported distributions except FreeBSD and Gentoo will be supported by these kernel telemetry changes. However if a customer is running a Gentoo or FreeBSD virtual machine, nothing will change for them after cloud-init is updated to have kernel telemetry.

# Timeline:
This feature is intended to be fully functional by August 2nd, 2019. 
