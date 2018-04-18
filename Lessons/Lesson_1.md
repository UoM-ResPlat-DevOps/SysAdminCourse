
-- *Slide* --
# Introduction
<img src="https://raw.githubusercontent.com/UoM-ResPlat-DevOps/SysAdminCourse/master/Images/spartanlogo.png" />
-- *Slide End* --

-- *Slide* --
## Goals for today
* Part 1: Spartan's architecture
* Part 2: Configuration and Communication
* Part 3: Karaage, Projects, and Accounts
* Part 4: Job Scheduling and Management
* Part 5: Operations
-- *Slide End* --

-- *Slide* --
## Slide Respository
* A copy of these slides and sample code is available at: `https://github.com/UoM-ResPlat-DevOps/SysAdminCourse`
-- *Slide End* --

-- *Slide* --
# 1. Spartan's architecture
-- *Slide End* --

-- *Slide* --
<img src="https://raw.githubusercontent.com/UoM-ResPlat-DevOps/SysAdminCourse/master/Images/spartanlayout.png" width="150%" height="150%" />
-- *Slide End* --


-- *Slide* --
## 1.1 Nodes
Spartan is as a hybrid HPC and cloud compute system that is orientated towards maximising throughput in a cost-efficient manner. There is a smaller traditional HPC partition and a larger partition of virtual machines from the Melbourne node of the NeCTAR research cloud. This matches the dominance of single-node jobs that were submitted on Spartan's predecessor, Edward. As typical with other HPC systems, it has a management node and a login node (extra login nodes added during courses). The management and login nodes are virtual machines.
-- *Slide End* --

-- *Slide* --
"Bare metal" HPC consists of is 44 nodes, 21 GB per core. 2 socket Intel E5-2643 v3 E5-2643, 3.4GHz CPU with 6-core per socket, 192GB memory, 2x 1.2TB SAS drives, 2x 40GbE network. There is currently 5 GPU nodes, 12xE5-2643 v3 at 3.40GHz with 251 GB per nodes and with 4*Tesla K80 each. There is also 280 cloud virtual machines with 8  2.3GHz Haswell cores with 8GB per cores.
-- *Slide End* --

-- *Slide* --
## 1.2 Network
The cloud nodes are connected via Cisco Nexus 10Gbe TCP/IP 60 usec latency (mpi- pingpong); and the bare metal Mellanox 2100 Cumulos Linux 40Gbe TCP/IP 6.85 usec latency and then RDMA Ethernet 1.15 usec latency. Later was superior to Infiniband FD14 (1.17 usec). Four different VLANs are used for communication within the cluster (IPMI baremetal), build and management (baremetal), main cluster traffic, and RoCE traffic.
-- *Slide End* --

-- *Slide* --
The new GPGPUs will have six racks requiring network for data, inband communication and out-of-band communication, with 14 nodes per rack, requiring 168 data ports, 84 * 1G inband ports and 84*1G out-of-band ports with Mellanox ConnectX 5 direct connect cards The network connection speed will be to 2 x 50Gbps (total of 100Gbps) connections per node, all connected to a Mellanox SN2100 switch for data, and more generic switches for inband and out-of-band.
-- *Slide End* --

-- *Slide* --
## 1.3 Storage
There are mountpoints to home, projects (/project /home for user data & scripts, NetApp SAS aggregate 70TB usable) and applications directories across all nodes. Additional mountpoins to VicNode Aspera Shares. Applications directory currently on management node, needs to be decoupled. Bare metal nodes have /scratch shared storage for MPI jobs (Dell R730 with 14 x 800GB mixed use SSDs providing 8TB of usable storage, NFS over RDMA)., /var/local/tmp for single node jobs, pcie SSD 1.6TB. Summary on spartan-m
-- *Slide End* --

-- *Slide* --
```
(vSpartan) [root@spartan-m vSpartan]# df -h
Filesystem                                   Size  Used Avail Use% Mounted on
/dev/vda1                                     30G   19G   12G  63% /
/dev/vdb1                                    1.0T  473G  552G  47% /usr/local
spartan-scratch.hpc.unimelb.edu.au:/scratch  8.0T  5.1T  3.0T  64% /scratch
spartan-stg2.hpc.unimelb.edu.au:/projects     63T   60T  3.3T  95% /data/projects
spartan-stg.hpc.unimelb.edu.au:/data01       4.5T  3.2T  1.4T  70% /home
```
-- *Slide End* --

-- *Slide* --
There is around 100TB of SAS performance storage from the NetApp Platform, however this is due to be decommissioned at the end of the year. This is currently used for project storage. The default allocation for projects is 500GB. The Project storage is currently divided up into a 70TB mount (maximum NetApp mount size)  and a 30TB mount on standby. There is 8TB of high-performance scratch/ which is a single compute node with SSDs. This is suffering some strain due to be accessed potentially by 350 nodes simultaneously.The application directory is on `/usr/local` is also mounted across all nodes
-- *Slide End* --

-- *Slide* --
The current proposed replacement is to use Ceph FS using Near-Line SAS. The recommended expansion is for an additional 6 nodes (to our existing 6) providing a total of 256TB. Unlike our current storage service, it does not have stable snapshot capability, although there is an expectation that this will be available at the end of the end. An intermin backup service (e.g., Commvault) will be used. Likewise the scratch partition will also be replaced with an 128TB of SSD storage. Finally, the application directory will be separated from the management node to its own virtualmachine using the current scratch parttion.
-- *Slide End* --

-- *Slide* --
## 1.4 Operating Systems, Deployment, and Monitoring
Like nearly all other HPC centres in the world, Spartan uses Linux and specifically the Red Hat distribution for its operating system; RHEL 7.* is used for all nodes, are we're subscribed to Unimelb Red Hat Satellite Server `https://sputnik.its.unimelb.edu.au` 
-- *Slide End* --

-- *Slide* --
There is also significant use of OpenStack's Nova (compute) service for provisioning and decommissioning of virtual machines on demand, which includes the management and login nodes. Specifically Spartan makes use of the following from the NeCTAR research cloud.
-- *Slide End* --

-- *Slide* --
Cell: melbourne-qh2-uom   
Tenant: vSpartan f31a36cc4dec4727a70b87917936f73e   
Instance flavor: mel.hpc.8c64g (8 vcpus, 64gb of ram, 30gb root, no ephemeral)   
OS image: os image list --private - usually the latest spartan-rc-gold-YYYYMMDD   
-- *Slide End* --

-- *Slide* --
DNS is managed through `https://ns-q.melbourne.nectar.org.au:9000`. The reverse IP entry ${ip_address} IN PTR vm-${ip_address}.melbourne.nectar.org.au must be deleted otherwise ssh hostbased authentication will not work.
-- *Slide End* --

-- *Slide* --
Our monitoring system is conducted through the NeCATR monitoring services (`https://mon.rc.nectar.org.au/`), which for Spartan includes PuppetBoard, Nagios, amd Grafana. In the near future will we also be implementing Xdmod.
-- *Slide End* --

-- *Slide* --
# 2. Configuration and Communication
-- *Slide End* --

-- *Slide* --
<img src="https://raw.githubusercontent.com/UoM-ResPlat-DevOps/SysAdminCourse/master/Images/longines1897.jpg" width="150%" height="150%" />
-- *Slide End* --

-- *Slide* --
## 2.1 Spartan's Configuration Files
Spartan's configuration files are stored on a the NeCTAR Git repository as Puppet files.Then they are reviewed through Gerrit and Jenkins. The combination of these features ensures sanity checking and creates "paired systems administration".
-- *Slide End* --

-- *Slide* --
Setup a NeCTAR Gerrit Account with either Launchpad, Yahoo!, or OpenID.
`https://review.rc.nectar.org.au/login/`
To clone the puppet repository for Spartan's configuration.
`git clone ssh://$username@review.rc.nectar.org.au:29418/internal/puppet-spartan.git`
Make changes locally, and commit.. `$ git add`, `git commit`, `git review`. 
-- *Slide End* --

-- *Slide* --
There is also the operations document Wiki for Spartan.
`git clone git@github.com:UoM-ResPlat-DevOps/ops-doc.wiki.git`
-- *Slide End* --

-- *Slide* --
## 2.2 Review Process on Gerrit
Gerrit is a web-based code collaboration tool. Whilst usually used by software developers, we also use it for configuration review. Modifications to the configuration code can be viewed on web browser and approve or reject those changes. It has an interesting interface. Jenkins provides the integration.
e.g., `https://review.rc.nectar.org.au/#/c/10294/`
-- *Slide End* --

-- *Slide* --
## 2.3 Slack
The NeCTAR research cloud runs a Slack service for synchronous communication between groups and individuals. The main relevant channel is `\#uom-hpc` and `\#uom-ops`. Slack is useful for providing alerts from humans that require a quick intervention from others, or to report on active task activities.
`https://researchcloud.slack.com/`
-- *Slide End* --

-- *Slide* --
# 3. Karaage, Projects, and Accounts
-- *Slide End* --

-- *Slide* --
<img src="https://raw.githubusercontent.com/UoM-ResPlat-DevOps/SysAdminCourse/master/Images/karaage.png" width="150%" height="150%" />
-- *Slide End* --

-- *Slide* --
## 3.1 Karaage
Karaage is a cluster account management tool. It can manage users and projects in a cluster and can store the data in various backends and uses a Django framework. User information can be stored in LDAP, Active Directory, or passwd files. On Spartan we use a local LDAP. Karaage has an email notifcayion service, allows for the creation of accounts by project leaders, track software usage, and cluster utilisation (per institute, system, or project) by CPU hours.
-- *Slide End* --

-- *Slide* --
Spartan's Karaage is located at `https://dashboard.hpc.unimelb.edu.au/karaage/`. This is also the site of the Spartan website. We are also considering shifting some of the Slurm database functions (at least as a backup) to this site as well.
-- *Slide End* --

-- *Slide* --
## 3.2 Accounts and Projects
Users belong to accounts, and accounts belong to projects. When a user applies for an account and project this will trigger a pending user and project on Karaage, after it has been approved by the Head of Research Compute. It is worth checking to see if the project applicant has actually requested a cluster acocunt. Sometimes the user just forgets to tick the box. However, if you see a professor/manager applying, they actually might not need access to the clusters. Check with the users before tick it for them.
-- *Slide End* --

-- *Slide* --
Once project lead user has been approved, go to the newly created projects and click on Grant Access next to the user to make them the Project leader. This will allow us to hand off the management of the projects to the users. Then create a project directory. If a directory with project name in `/data/projects` (usually punimXXXX) has not been created:
`cd /root/`
`./mkproject.sh $project_name`
-- *Slide End* --

-- *Slide* --
# 4. Job Scheduling and Management
-- *Slide End* --

-- *Slide* --
<img src="https://raw.githubusercontent.com/UoM-ResPlat-DevOps/SysAdminCourse/master/Images/slurm.jpg" width="150%" height="150%" />
-- *Slide End* --

-- *Slide* --
## 4.1 Slurm
The Slurm Workload Manager (formerly SLURM, Simple Linux Utility for Resource Management), is a free and open-source job scheduler for Linux and Unix-like systems and combines the functions of a job scheduler and resource manager. It is notable for its efficiency, scalability, reporting tools, and modular design with around 100 optional plugins.
-- *Slide End* --

-- *Slide* --
The main daemon for slurm is `slurmctld` which coordinates the queueing of jobs, monitoring node states, and allocating resources to jobs. In addition there is a slurmd daemon running on each compute node. The user commands include: `sacct`, `salloc`, `sattach`, `sbatch`, `sbcast`, `scancel`, `scontrol`, `sinfo`, `smap`, `squeue`, `srun`, `strigger` and `sview`. 
-- *Slide End* --

-- *Slide* --
Man pages exist for all Slurm daemons, commands, and API functions (c.f., `https://slurm.schedmd.com/man_index.html`). The Slurm daemons manage nodes, partitions, jobs. The partitions are effectively job queues with resource and user constraints. Jobs are allocated to nodes within a partition according to proirity. 
-- *Slide End* --

-- *Slide* --
## 4.2 Spartan's Partitions
Partitions (equivalent of queues) can be viewed with the `sinfo` command. Nodes can belong to multiple partitions simultaneously. For example, all nodes are in the `debug` partition. Detailed information about a partition and constraints can be viewed with `scontrol show partition`.
-- *Slide End* --

-- *Slide* --
Spartan's main partitions are the `cloud` partition and the `physical` partition. As the names indicate the cloud partition consists of nodes that are virtual machines, where as the physical partition consists of nodes that are on bare metal, running with a higher-speed interconnect. Of note are project specific partitions (e.g., punim0095), departmental partitions (e.g., `water`, `ashley`), and partitions with different architecture (e.g., `gpu`).
-- *Slide End* --

-- *Slide* --
## 4.3 General Job Submission
Like other scheduling systems Slurm requires that a user submit a batch request of resources, for a particular period of time. The resources include nodes, tasks, tasks-per-node, cpus-per-task, generic resources (e.g., GPUs). Submission is with the `sbatch jobname` command, or `sinteractive` for interactive jobs. Numerous example scripts are in the `/usr/local/common/` directory. Note in particular those in the `IntroLinux`, `HPCshells`, `array`, and `depend` directories.
-- *Slide End* --

-- *Slide* --
## 4.4 Common Administrator Operations
* Users and administrators can retrieve detailed information about a job with: `scontrol show job [jobid]`
* Administrators can specify the new walltime for a job. e.g., `scontrol update jobid=1042541 TimeLimit=30-0:0:0`
-- *Slide End* --

-- *Slide* --
* Managers often want usage metrics. The following sreport command provides examples of utilisation: e.g.,
`sreport -t Hours cluster Utilization start=2016-01-01 end=2016-11-10`
`sreport cluster AccountUtilizationByUser cluster=spartan user=khalid start=2016-01-01 end=2017-01-17`
-- *Slide End* --

-- *Slide* --
* To build a bare-metal or cloud node see: `https://github.com/UoM-ResPlat-DevOps/ops-doc/wiki/Build-Spartan-Nodes`
* If a node needs to be upgraded etc, set it for drain. Check the workload of non-cloud partitions beforehand (`sinfo -p $partition`), and jobs on the nodes (`spartan-m: ~/jobsonnode.sh $node` or `squeue -w $nodename`
-- *Slide End* --

-- *Slide* --
* Then run the drain command `scontrol update nodename=$nodes state=DRAIN reason="$reason"`. The node, if idle, will go to drain. If it it has jobs running it will be marked as `draining`. Node status can be checked with `downbecause`. To return drained nodes to production use `scontrol update nodename=$nodes state=RESUME`.
-- *Slide End* --

-- *Slide* --
* You can stop Slurm from scheduling jobs on a per partition basis by setting that partition's state to DOWN. Set its state UP to resume scheduling. For example:
`$ scontrol update PartitionName=$partition State=DOWN`
`$ scontrol update PartitionName=$partition State=UP`
-- *Slide End* --

-- *Slide* --
* The slurm configuration file is located at `/usr/local/slurm/etc/slurm.conf` on the management node. This includes various resource limits, partitions etc. If Slurm is reconfigured, run `/usr/local/resplat/sbin/slurm-run-jobs.sh` to bring all partitions back up.
-- *Slide End* --

-- *Slide* --
* Spartan has regular updates of packages and controlled upgrades of the kernel. `https://github.com/UoM-ResPlat-DevOps/ops-doc/wiki/Weekly-Upgrades---Spartan`
* When upgrading Slurm, SlurmDBD must be upgraded first, always. `https://slurm.schedmd.com/quickstart_admin.html#upgrade`. Detailed instructions on the Wiki `https://github.com/UoM-ResPlat-DevOps/ops-doc/wiki/Spartan-Upgrades`.
-- *Slide End* --


-- *Slide* --
# 5.0 Operations
-- *Slide End* --

-- *Slide* --
<img src="https://raw.githubusercontent.com/UoM-ResPlat-DevOps/SysAdminCourse/master/Images/operation.jpg" />
-- *Slide End* --


-- *Slide* --
## 5.1 Project Workflow
Spartan uses Trello to manage project workflow through "cards" with associated users and content checklists. This is usually for multi-task objectives that are generated internally rather than from user requests. However sometimes user requests via the helpdesk can become Trello cards as well, if of sufficient complexity (the split between helpdesk tickets and Trello cards is not well defined).
-- *Slide End* --

-- *Slide* --
To access the Spartan HPC Trello, signup at `https://trello.com/signup` and login at `https://trello.com/login` (Google account option). The page URL is located at: `https://trello.com/b/kvLgvEZa/hpc` with the following Menus; "To Do", "On Hold", "Doing", "Admin & Doco", and "Done".
-- *Slide End* --

-- *Slide* --
## 5.2 Helpdesk
Spartan (currently) uses the NeCTAR Freshdesk system for managing user request tickets at `https://support.ehelp.edu.au/helpdesk/dashboard`. Our group is `UoM HPC`. General users are accredited by administrators and the existing current implementation has a national scope. In nearly all cases, ticket creation and responses are carried out by email as the single point of entry and communication.
-- *Slide End* --

-- *Slide* --
Freshdesk comes with SOAP etc and several REST APIs (ticket, user, agent, companies, forum (several subcategories), solution (several subcategories), time entries, survey, groups), full text-search capability, built-in customer satisfaction surveys, and an extensive range of customer support contact methods (email, web, phone, instant messaging).
-- *Slide End* --

-- *Slide* --
As a general workflow if you can respond to a ticket, assign it to yourself first, then respond. Avoid responding to tickets owned by another person, but if you have an idea or suggestion, send it through Slack to the owner or as a private note on the ticket. Make sure that close tickets (set to resolved). Provide solutions to the user's problem or, if this is not possible, provide alternatives.
-- *Slide End* --

-- *Slide* --
## 5.3 Software Builds
As with all HPC systems, whenever possible software is built from source. Usually this to ensure maximum optimisation of code (typical performance improvements are around 30%), and sometimes it is to ensure that the software can be installed in the first place. Most of all however, it is to ensure that multiple version of the same software can be installed. This provides consistency and better reproducibility in code, whilst also allowing for different features. To achieve this, environment modules are employed. 
-- *Slide End* --

-- *Slide* --
In particular Spartan uses the Easybuild system to ensure build consistency and which incorporates the LMod environment modules system. LMod is virtually identical in most cases to the TCL-based environment modules system. Easybuild requires a little explanation. Essentially it provides a repository of Python scripts to a particular software build ("Easybuild recipes"), which includes version numbers, dependencies, toolchains etc. These recipes in turn call an EasyBlock, which is a configuration style (e.g., Binary, ConfigureMake) and often specific to an application.
-- *Slide End* --

-- *Slide* --
Easybuild exists as a build user and has its own module file. It is important to purge existing modules before running an EasyBuild script.
```
ssh root@spartan-m.hpc.unimelb.edu.au   
su - easybuild   
module purge   
module load EasyBuild   
cd /usr/local/easybuild/ebfiles_repo/   
```
-- *Slide End* --

-- *Slide* --
## 5.4 Training
Many users (even post-doctoral researchers) require basic training in Linux command line, a requisite skill for HPC use. Extensive training programme for researchers available using andragogical methods, including day-long courses in "Introduction to Linux and HPC Using Spartan", "Linux Shell Scripting for High Performance Computing", and "Parallel Programming On Spartan". 
-- *Slide End* --

-- *Slide* --
Documentation online (Github, Website, and man pages) and plenty of Slurm examples on system.
Github:
`https://github.com/UoM-ResPlat-DevOps/SpartanIntro`   
`https://github.com/UoM-ResPlat-DevOps/HPCshells`    
`https://github.com/UoM-ResPlat-DevOps/SpartanIntro`   
On Spartan:
`man spartan`   
`/usr/local/common/`   
Website:
`https://dashboard.hpc.unimelb.edu.au`   
-- *Slide End* --

-- *Slide* --
<img src="https://raw.githubusercontent.com/UoM-ResPlat-DevOps/SysAdminCourse/master/Images/hypnotoad.png" width="150%" height="150%" />
-- *Slide End* --





