# AWS EC2 Phase 1 — Complete Hands-On Notes

> Goal: These notes are written so that even years later you can rebuild the same mental model and repeat the entire EC2 workflow from launch to cleanup without relying on memory.

---

# 1. What We Built

In this phase, we launched an Ubuntu EC2 virtual machine in AWS, connected to it over SSH, installed Git and Docker, cloned a Dockerized React application, ran it behind Nginx inside Docker, exposed it through an EC2 Security Group, inspected CPU/RAM/disk usage, and finally terminated the EC2 instance safely.

The final architecture looked like this:

```text
Your Browser
     |
     | HTTP
     v
EC2 Public IPv4
     |
     v
Security Group
     |
     v
EC2 Ubuntu
     |
     | Host Port 8080
     v
Docker
     |
     | 8080 -> 80
     v
Nginx Container
     |
     v
React Application
```

For SSH:

```text
Your Linux PC
     |
     | SSH TCP 22
     | Private key: EC2 Tutorial.pem
     v
Internet
     |
     v
EC2 Security Group
     |
     v
EC2 Public IPv4
     |
     v
Ubuntu SSH Server
     |
     v
ubuntu user
```

---

# 2. EC2 Mental Model

EC2 stands for:

```text
Elastic Compute Cloud
```

The simplest mental model is:

> EC2 is a virtual computer running inside AWS.

A normal computer has:

```text
CPU
RAM
Disk
Operating System
Network
Applications
```

An EC2 instance has the same broad components:

```text
Instance Type
    -> vCPU
    -> RAM

AMI
    -> Operating System

EBS
    -> Disk

Network Interface
    -> Private IP

Optional Public IPv4
    -> Internet connectivity

Security Group
    -> Virtual firewall
```

Our instance was approximately:

```text
t3.micro
├── 2 vCPU
├── ~1 GiB RAM
└── 8 GiB gp3 EBS storage
```

Important:

```text
RAM != Disk
```

The 8 GiB we configured was **EBS disk storage**, not RAM.

---

# 3. Region

AWS resources are usually created inside a region.

Example:

```text
Asia Pacific (Hyderabad)
```

A region contains multiple Availability Zones.

Conceptually:

```text
AWS
└── Region
    ├── Availability Zone A
    ├── Availability Zone B
    └── Availability Zone C
```

Our instance was created inside one Availability Zone in the selected region.

Why regions matter:

- latency
- service availability
- data residency
- pricing
- disaster recovery design
- where your dependent AWS resources live

For learning, the most important rule is:

> Always look at the region selector before creating or searching for a resource.

A resource created in Hyderabad will not appear if you are currently viewing another region.

---

# 4. AMI

AMI means:

```text
Amazon Machine Image
```

Think of an AMI as a machine template.

It determines the initial operating system and software image used when the EC2 instance is launched.

We used:

```text
Ubuntu Server 26.04 LTS
64-bit x86
```

After logging in, Linux confirmed this:

```bash
cat /etc/os-release
```

Example output:

```text
PRETTY_NAME="Ubuntu 26.04 LTS"
VERSION="26.04 LTS"
```

The AMI also determines the default SSH username.

For Ubuntu AMIs:

```text
ubuntu
```

That is why the SSH command used:

```bash
ssh -i key.pem ubuntu@PUBLIC_IP
```

---

# 5. Instance Type

The instance type controls the compute size.

Example:

```text
t3.micro
```

It determines things such as:

```text
vCPU
RAM
network characteristics
CPU behavior
```

Our Linux machine confirmed:

```bash
lscpu
```

and showed:

```text
CPU(s): 2
Architecture: x86_64
```

The EC2 console said roughly:

```text
2 vCPU
1 GiB RAM
```

Linux reported about:

```text
908 MiB usable RAM
```

This is normal. The guest OS does not necessarily expose the entire advertised memory value to normal userspace because some memory is reserved/used by the kernel and virtualized hardware environment.

---

# 6. EBS Storage

EBS means:

```text
Elastic Block Store
```

EBS is disk storage attached to EC2.

Think of it like the SSD attached to a laptop, except it is an AWS-managed block storage resource.

We selected:

```text
8 GiB
gp3
```

Mental model:

```text
EC2
├── CPU
├── RAM
└── attached EBS volume
       └── Ubuntu filesystem
```

EBS is separate from RAM.

### RAM

Temporary working memory.

```text
Used while programs are running.
Lost when processes stop.
```

### EBS

Persistent disk storage.

```text
OS files
Docker images
Git repositories
logs
application files
```

Check filesystems:

```bash
df -h
```

Our root filesystem eventually showed approximately:

```text
/dev/root
Size: 6.7G
Used: 3.7G
Available: 3.0G
```

Why was `/` not the full 8 GiB?

Because the disk also contained boot-related partitions such as:

```text
/boot
/boot/efi
```

Conceptually:

```text
8 GiB EBS
├── root filesystem /
├── /boot
├── /boot/efi
└── filesystem/partition overhead
```

To inspect block devices more clearly:

```bash
lsblk
```

---

# 7. gp3

`gp3` is a general-purpose SSD-backed EBS volume type.

It is a good default for many ordinary workloads.

Think:

```text
gp3
= general-purpose SSD block storage
```

For small learning servers, it is more than enough.

Do not increase disk size simply because a larger value is available.

Start small, monitor usage, and resize when required.

---

# 8. Key Pair and SSH Authentication

When launching EC2, we selected a key pair.

Example:

```text
EC2 Tutorial.pem
```

This `.pem` file is your **private key**.

AWS places the corresponding public key on the EC2 instance.

Conceptually:

```text
Your computer
└── Private key

EC2
└── Public key
```

During SSH login, the EC2 machine verifies that you possess the matching private key.

Important:

> Never commit a `.pem` file to Git.
> Never upload it to a public repository.
> Never share it casually.

---

# 9. Private Key File Permissions

OpenSSH intentionally rejects overly-permissive private key files.

We used:

```bash
chmod 400 'EC2 Tutorial.pem'
```

Meaning:

```text
Owner:
  read = yes
  write = no
  execute = no

Group:
  no permissions

Others:
  no permissions
```

Verify with:

```bash
ls -l 'EC2 Tutorial.pem'
```

Example:

```text
-r-------- ... EC2 Tutorial.pem
```

---

# 10. Common SSH Path Error

We initially had the key in:

```text
~/Downloads/EC2 Tutorial.pem
```

But attempted SSH after moving to:

```text
~
```

with:

```bash
ssh -i 'EC2 Tutorial.pem' ubuntu@PUBLIC_IP
```

That made SSH search for:

```text
~/EC2 Tutorial.pem
```

which did not exist.

Result:

```text
Warning: Identity file EC2 Tutorial.pem not accessible:
No such file or directory.
```

Fix either by going into the correct directory:

```bash
cd ~/Downloads
ssh -i 'EC2 Tutorial.pem' ubuntu@PUBLIC_IP
```

or by using the full path:

```bash
ssh -i ~/Downloads/'EC2 Tutorial.pem' ubuntu@PUBLIC_IP
```

General lesson:

> Relative paths are resolved from the current working directory.

---

# 11. SSH Host Verification

On the first SSH connection, we saw:

```text
The authenticity of host '...' can't be established.
Are you sure you want to continue connecting?
```

This is different from your `.pem` key.

There are two directions of trust:

```text
EC2 verifies you
    <- your private key

Your computer verifies EC2
    <- EC2 server host key
```

When you type:

```text
yes
```

the server host key is stored in:

```text
~/.ssh/known_hosts
```

Future connections compare the server's key against that known value.

---

# 12. SSH Command Anatomy

Example:

```bash
ssh -i 'EC2 Tutorial.pem' ubuntu@18.x.x.x
```

Breakdown:

```text
ssh
    Secure Shell client

-i
    identity file option

EC2 Tutorial.pem
    private key used for authentication

ubuntu
    remote Linux username

18.x.x.x
    EC2 public IPv4 address
```

SSH normally uses:

```text
TCP port 22
```

Therefore the EC2 Security Group must allow inbound TCP 22 from your IP.

---

# 13. Security Groups

A Security Group is a virtual firewall associated with AWS resources such as EC2.

For our learning instance:

```text
SSH
TCP 22
Source: My IP

HTTP
TCP 80
Source: 0.0.0.0/0
```

Later, when we temporarily exposed the Docker-host port 8080:

```text
Custom TCP
Port 8080
Source: 0.0.0.0/0
```

Important rule:

> Opening a port in Docker is not enough.
> Opening a port on Linux is not enough.
> The AWS Security Group must also allow the traffic.

Traffic path:

```text
Internet
   |
   v
Security Group
   |
   v
EC2 host port
   |
   v
Docker port mapping
   |
   v
Container
```

---

# 14. Why SSH Should Use "My IP"

Avoid this for SSH unless absolutely necessary:

```text
0.0.0.0/0
```

That means:

```text
any IPv4 address on the internet
```

For SSH we prefer:

```text
My IP /32
```

Conceptually:

```text
Your IP
   |
   | allowed
   v
EC2 :22

Everyone else
   X
```

HTTP is different because public websites are supposed to accept requests from users on the internet.

---

# 15. Public IP vs Private IP

We connected using a public IPv4 such as:

```text
18.x.x.x
```

But inside the EC2 instance:

```bash
hostname
```

returned something such as:

```text
ip-172-31-14-106
```

and:

```bash
ip addr
```

showed:

```text
172.31.14.106
```

This is the private IP.

Mental model:

```text
Internet
   |
   | Public IPv4
   v
AWS networking
   |
   | mapping
   v
EC2 private IPv4
```

Inside the guest operating system, you commonly see the private IP, not the public IPv4.

The private IP is used inside the VPC.

The public IPv4 is used to reach the instance from the internet.

This becomes much more important in the VPC/networking phase.

---

# 16. Linux Commands We Used to Inspect EC2

## Current user

```bash
whoami
```

Output:

```text
ubuntu
```

## Hostname

```bash
hostname
```

Example:

```text
ip-172-31-14-106
```

## Current working directory

```bash
pwd
```

Example:

```text
/home/ubuntu
```

## Kernel/system architecture

```bash
uname -a
```

## Operating system version

```bash
cat /etc/os-release
```

## CPU details

```bash
lscpu
```

## Memory

```bash
free -h
```

## Filesystem usage

```bash
df -h
```

## Network interfaces

```bash
ip addr
```

## Block devices

Useful later:

```bash
lsblk
```

---

# 17. RAM: Used, Free, Cache, Available

Example:

```bash
free -h
```

Output:

```text
               total    used    free   buff/cache   available
Mem:           908Mi    457Mi   111Mi     456Mi       451Mi
Swap:             0B       0B      0B
```

Do not look only at:

```text
free = 111 MiB
```

Linux intentionally uses otherwise-idle memory for caches.

The more useful number is often:

```text
available = 451 MiB
```

Interpretation:

```text
total
    total usable RAM visible to Linux

used
    memory currently in active use

free
    completely unused RAM

buff/cache
    memory used for cache/buffers

available
    approximate amount available for new workloads
```

Linux can reclaim much of cache memory when applications need it.

---

# 18. Swap

Our EC2 showed:

```text
Swap: 0B
```

That means there was no configured swap space.

Swap is disk space that Linux can use as overflow when RAM pressure is high.

It is much slower than RAM.

For a tiny EC2 instance, memory pressure matters because a 1 GiB server has little headroom.

Do not treat swap as a replacement for sufficient RAM.

---

# 19. Installing Git

Ubuntu package installation:

```bash
sudo apt update
sudo apt install git -y
```

Verify:

```bash
git --version
```

Meaning of:

```text
sudo apt update
```

Refresh package metadata.

Meaning of:

```text
sudo apt install git -y
```

Install Git and automatically confirm prompts.

---

# 20. `sudo`

We logged in as:

```text
ubuntu
```

not:

```text
root
```

Administrative operations require elevated privileges.

Example:

```bash
sudo apt update
```

Mental model:

```text
ubuntu user
     |
     | sudo
     v
temporary elevated/root privilege
```

Do not operate permanently as root unless there is a specific reason.

---

# 21. Installing Docker

We installed Docker on Ubuntu.

Example:

```bash
sudo apt update
sudo apt install docker.io -y
```

Verify:

```bash
docker --version
```

Check service:

```bash
sudo systemctl status docker
```

Press:

```text
q
```

to exit the `systemctl status` viewer.

Test Docker:

```bash
sudo docker run hello-world
```

This verifies:

```text
Docker CLI
   |
   v
Docker daemon
   |
   v
Pull image
   |
   v
Create container
   |
   v
Run container
```

---

# 22. Docker Compose v2

Our first attempt:

```bash
docker compose up -d
```

failed because Compose support was not available.

The modern syntax is:

```bash
docker compose ...
```

Older tutorials may show:

```bash
docker-compose ...
```

Prefer Compose v2:

```text
docker compose
```

Install the Compose v2 package/plugin appropriate for the system, then verify:

```bash
docker compose version
```

Start services:

```bash
sudo docker compose up -d
```

Stop/remove the Compose services:

```bash
sudo docker compose down
```

---

# 23. What `-d` Means

Example:

```bash
docker compose up -d
```

`-d` means:

```text
detached mode
```

Without `-d`, logs remain attached to the terminal.

With `-d`, containers continue running in the background and the shell prompt returns.

---

# 24. Docker Port Mapping

`docker ps` showed:

```text
0.0.0.0:4173->4173/tcp
0.0.0.0:8080->80/tcp
```

Syntax:

```text
HOST_PORT : CONTAINER_PORT
```

Therefore:

```text
8080:80
```

means:

```text
EC2 host port 8080
       |
       v
Docker container port 80
```

Our Nginx container listened on port:

```text
80
```

but users connected to EC2 port:

```text
8080
```

Therefore the browser URL was:

```text
http://PUBLIC_IP:8080
```

---

# 25. Container Port vs Host Port vs Security Group Port

This is one of the most important concepts from the entire exercise.

Suppose Docker says:

```text
8080:80
```

Then:

```text
Browser
    |
    | PUBLIC_IP:8080
    v
AWS Security Group
    |
    | must allow 8080
    v
EC2 host port 8080
    |
    | Docker mapping
    v
Container port 80
    |
    v
Nginx
```

All layers must align.

If Docker exposes 8080 but the Security Group allows only 80:

```text
Browser -> EC2:8080
            X
```

The container may be perfectly healthy, but the request never reaches it.

---

# 26. Verify Locally Before Blaming AWS

Before editing a Security Group, verify the service from inside EC2.

Example:

```bash
curl http://localhost:8080
```

or:

```bash
curl -I http://localhost:8080
```

If this returns:

```text
HTTP/1.1 200 OK
```

then:

```text
Docker works
Nginx works
React is being served
```

If the browser still cannot connect, investigate:

```text
Security Group
host firewall
public IP
port mapping
```

This is a strong troubleshooting habit:

> Test from the inside outward.

---

# 27. Better Production Port Mapping

We temporarily used:

```text
8080:80
```

For a normal public HTTP website, a cleaner mapping is:

```text
80:80
```

Then:

```text
Browser
   |
   | HTTP 80
   v
EC2 :80
   |
   v
Docker
   |
   v
Nginx :80
```

The URL becomes:

```text
http://PUBLIC_IP
```

instead of:

```text
http://PUBLIC_IP:8080
```

---

# 28. Why Vite Port 4173 Should Not Be Public

Our Docker environment also exposed a Vite-related port:

```text
4173
```

But when Nginx is the public frontend server, internet users should normally reach Nginx, not the Vite preview/dev server.

Preferred idea:

```text
Internet
   |
   v
Nginx
   |
   v
React static build
```

Avoid opening every application/container port in the Security Group.

Expose only what users actually need.

---

# 29. Inspect Running Docker Containers

```bash
sudo docker ps
```

Example:

```text
CONTAINER ID
IMAGE
STATUS
PORTS
NAMES
```

Useful for checking:

```text
Is the container running?
Which image is it using?
Which ports are published?
What is its container name?
```

---

# 30. Inspect Container CPU and RAM

One-time snapshot:

```bash
sudo docker stats --no-stream
```

Live view:

```bash
sudo docker stats
```

Example:

```text
NAME                 CPU %    MEM USAGE / LIMIT
vite                  0.00%    67.48MiB / 908.7MiB
nginx                 0.00%     6.76MiB / 908.7MiB
```

Combined container memory was roughly:

```text
67.48 MiB + 6.76 MiB
≈ 74 MiB
```

Meaning:

```text
React/Vite container
~67 MiB

Nginx container
~7 MiB
```

At that moment, both were essentially idle in CPU terms.

---

# 31. Inspect Linux Processes by Memory

Useful command:

```bash
ps aux --sort=-%mem | head
```

This shows the largest memory-consuming processes first.

For interactive monitoring:

```bash
top
```

Important columns include:

```text
%CPU
%MEM
RES
COMMAND
```

Press:

```text
q
```

to exit.

---

# 32. Docker Disk Usage

Docker can consume significant EBS storage through:

```text
images
containers
volumes
build cache
```

Inspect with:

```bash
sudo docker system df
```

Conceptually:

```text
EBS
├── Ubuntu
├── packages
├── Git repo
└── Docker
    ├── Images
    ├── Containers
    ├── Volumes
    └── Build cache
```

This explained why our disk usage increased significantly after installing and building Docker workloads.

---

# 33. Disk Usage Before and After Docker

Before Docker/application work, root storage was roughly:

```text
Used: 2.1G
```

Later:

```text
Used: 3.7G
```

Approximately:

```text
+1.6 GiB
```

was consumed by things such as:

```text
Docker packages
Docker images
build cache
Git repository
application files
other installed packages
```

This is why disk monitoring matters even for a frontend app.

---

# 34. User Data

User Data is a startup/bootstrap mechanism for EC2.

Example:

```bash
#!/bin/bash
apt update -y
apt install nginx -y
systemctl enable nginx
systemctl start nginx
```

Instead of manually doing:

```text
Launch EC2
SSH
Install packages
Configure services
```

User Data can automate initial setup:

```text
Launch EC2
   |
   v
cloud-init runs User Data
   |
   v
packages installed
   |
   v
services started
```

We intentionally left User Data empty for the first server so that we could understand every manual step.

Good learning order:

```text
1. Do it manually
2. Understand it
3. Automate it
```

Important:

> User Data is normally treated as initial bootstrapping.
> Editing it later does not automatically mean it will run on every reboot.

---

# 35. Stop vs Terminate

## Stop

Stopping an instance means:

```text
Running
   |
   v
Stopped
```

The virtual machine is not actively running compute workloads, but associated resources such as EBS may remain.

You can normally start it again.

Important:

```text
EC2 compute stops
EBS can remain
public IPv4 can change on stop/start
```

## Terminate

Terminating means:

```text
Running
   |
   v
Terminated
```

The EC2 instance is destroyed permanently.

You cannot restart a terminated instance.

If the root EBS volume has:

```text
Delete on termination = enabled
```

the root volume will usually be removed as part of termination.

---

# 36. Why a Terminated Instance Still Appears in the Console

After termination, AWS may continue displaying the instance temporarily.

Example state:

```text
Terminated
```

This is normal.

The row remains for history/reference and later disappears from the normal view.

Important distinction:

```text
Visible in console
!=
still running
```

Check:

```text
Instance state = Terminated
```

That confirms compute has been destroyed.

---

# 37. What to Check After Termination

Deleting the EC2 instance does not mean you should blindly assume every related resource disappeared.

Perform a cleanup audit.

## EC2

```text
EC2 -> Instances
```

Confirm:

```text
Terminated
```

## EBS Volumes

```text
Elastic Block Store -> Volumes
```

Look for leftover volumes.

If a volume is:

```text
available
```

it means it is unattached.

Do not delete it unless you confirm it belonged to the learning instance.

## Snapshots

```text
Elastic Block Store -> Snapshots
```

Snapshots store EBS data and can consume storage/billing.

## Elastic IPs

```text
Network & Security -> Elastic IPs
```

We did not intentionally create one.

An automatically assigned EC2 public IPv4 is not the same thing as an Elastic IP.

## Security Groups

A learning Security Group may remain after the instance is gone.

Security Groups themselves are not EC2 compute instances.

You may delete unused custom Security Groups for cleanliness.

Do not blindly delete the default Security Group.

## Key Pairs

Key pairs can remain.

They do not represent a running EC2 compute resource.

Keep your `.pem` file secure.

---

# 38. Cost Checking

AWS billing can lag behind actual resource creation/termination.

Do not expect every minute of usage to appear instantly.

Useful console areas:

```text
Billing and Cost Management
Bills
Cost Explorer
Budgets
```

Check costs by service such as:

```text
EC2
EBS
Public IPv4
RDS
S3
Load Balancers
NAT Gateway
CloudFront
```

Important:

> There is no universal "nothing is running anywhere" button for all AWS services.

You must combine:

```text
resource cleanup
+
billing monitoring
+
budgets/alerts
```

---

# 39. Recommended Cleanup Checklist

After every AWS learning experiment:

```text
[ ] EC2 instance terminated/stopped intentionally
[ ] EBS volumes checked
[ ] snapshots checked
[ ] Elastic IPs checked
[ ] load balancers checked
[ ] NAT Gateways checked
[ ] RDS checked
[ ] S3 checked if used
[ ] custom resources cleaned up if no longer required
[ ] Billing reviewed
[ ] Budget alert configured
```

---

# 40. Full EC2 Workflow — Start to Finish

This is the complete repeatable procedure.

## Step 1 — Open EC2

AWS Console:

```text
EC2
-> Instances
-> Launch instances
```

## Step 2 — Name the instance

Example:

```text
aws-learning-ec2-01
```

## Step 3 — Select AMI

Example:

```text
Ubuntu Server
64-bit x86
```

## Step 4 — Select instance type

Example:

```text
t3.micro
```

Always check the current AWS pricing/free-tier status in your account.

## Step 5 — Select/create key pair

Example:

```text
EC2 Tutorial
```

Download:

```text
EC2 Tutorial.pem
```

Keep it safe.

## Step 6 — Networking

For an initial learning instance:

```text
Default VPC
Default/no-preference subnet
Auto-assign Public IP = enabled
```

## Step 7 — Security Group

Recommended initial rules:

```text
SSH
TCP 22
Source: My IP

HTTP
TCP 80
Source: Anywhere IPv4
```

Do not open ports you do not currently need.

## Step 8 — Storage

Example:

```text
8 GiB gp3
```

## Step 9 — User Data

For the first manual learning exercise:

```text
leave empty
```

## Step 10 — Launch

Wait for:

```text
Instance state: Running
Status checks: passed
```

## Step 11 — Prepare private key

```bash
chmod 400 'EC2 Tutorial.pem'
```

## Step 12 — SSH

```bash
ssh -i 'EC2 Tutorial.pem' ubuntu@PUBLIC_IP
```

On first connection:

```text
Are you sure you want to continue connecting?
```

type:

```text
yes
```

## Step 13 — Inspect machine

```bash
whoami
hostname
pwd
uname -a
cat /etc/os-release
lscpu
free -h
df -h
ip addr
lsblk
```

## Step 14 — Install Git

```bash
sudo apt update
sudo apt install git -y
git --version
```

## Step 15 — Clone project

For a public repository:

```bash
git clone <repository-url>
cd <repository-folder>
```

## Step 16 — Install Docker

Example:

```bash
sudo apt update
sudo apt install docker.io -y
```

Verify:

```bash
docker --version
sudo systemctl status docker
sudo docker run hello-world
```

## Step 17 — Install/verify Docker Compose

Verify:

```bash
docker compose version
```

If unavailable, install the supported Compose v2 package/plugin for the operating system.

## Step 18 — Start application

```bash
sudo docker compose up -d
```

## Step 19 — Check containers

```bash
sudo docker ps
```

Look carefully at port mappings.

Example:

```text
0.0.0.0:8080->80/tcp
```

## Step 20 — Test from inside EC2

```bash
curl -I http://localhost:8080
```

Expected:

```text
HTTP/1.1 200 OK
```

## Step 21 — Open required Security Group port

If host port is:

```text
8080
```

temporarily allow:

```text
Custom TCP
8080
Anywhere IPv4
```

## Step 22 — Visit from browser

```text
http://PUBLIC_IP:8080
```

## Step 23 — Monitor resources

RAM:

```bash
free -h
```

Processes:

```bash
ps aux --sort=-%mem | head
```

Docker resources:

```bash
sudo docker stats --no-stream
```

Disk:

```bash
df -h
```

Docker disk:

```bash
sudo docker system df
```

## Step 24 — Optional reboot test

```bash
sudo reboot
```

Reconnect after reboot and verify:

```bash
sudo docker ps
```

This teaches whether your containers automatically restart.

## Step 25 — Terminate

AWS Console:

```text
EC2
-> Instances
-> select instance
-> Instance state
-> Terminate instance
```

## Step 26 — Verify cleanup

Check:

```text
EC2 -> Instances
EBS -> Volumes
EBS -> Snapshots
Network & Security -> Elastic IPs
Billing
```

---

# 41. Troubleshooting Flow

When a website does not open, do not randomly change AWS settings.

Use this sequence:

```text
1. Is the container running?
       docker ps

2. Is the service working inside EC2?
       curl localhost:PORT

3. Is Docker mapping the expected host port?
       docker ps

4. Is AWS Security Group allowing that host port?

5. Are you using the correct public IPv4?

6. Is the instance actually running?

7. Is the application listening on the expected interface/port?
```

For SSH:

```text
1. Correct public IP?
2. Instance running?
3. Security Group allows TCP 22 from your IP?
4. Correct username?
5. Correct .pem file?
6. Correct private-key permissions?
7. Correct path to key?
```

---

# 42. Common Errors We Hit

## Error: identity file not accessible

```text
Warning: Identity file ... not accessible
```

Cause:

```text
wrong path / wrong current directory
```

Fix:

```bash
ssh -i /full/path/to/key.pem ubuntu@PUBLIC_IP
```

---

## Error: Host key verification failed

Cause during first connection:

```text
host fingerprint was not accepted
```

Fix:

When prompted, type:

```text
yes
```

---

## Error: `docker compose up -d` does not understand Compose

Cause:

```text
Docker Compose v2 plugin is missing
```

Verify:

```bash
docker compose version
```

Install the appropriate Compose v2 package/plugin.

---

## Browser cannot access container

Possible cause:

```text
Docker host port is not allowed by Security Group
```

Example:

```text
Docker: 8080 -> 80
Security Group: only port 80
```

Then:

```text
PUBLIC_IP:8080
```

will fail until port 8080 is allowed.

---

# 43. Key Concepts to Remember

If you remember only the most important lessons, remember these:

```text
1. EC2 is a virtual machine.

2. Instance type controls CPU/RAM.

3. EBS is disk, not RAM.

4. AMI defines the machine image/OS.

5. Security Group is an AWS-level virtual firewall.

6. SSH usually uses TCP port 22.

7. A .pem file is your private SSH key.

8. Ubuntu AMIs commonly use the "ubuntu" login user.

9. EC2 can have both private and public IP addresses.

10. Docker container ports and EC2 host ports are different.

11. Docker exposure does not automatically bypass AWS Security Groups.

12. Test locally with curl before blaming AWS networking.

13. "free" RAM and "available" RAM are not the same thing.

14. Docker consumes both RAM and EBS disk.

15. Stop and Terminate are very different.

16. Terminating EC2 does not mean you should skip checking related AWS resources.

17. Billing data may take time to appear.

18. Build manually first; automate later with User Data, CI/CD and IaC.
```

---

# 44. Where This Fits in the Larger AWS Journey

This phase gave us the foundation for later topics.

We now understand enough to move toward:

```text
EC2
  -> VPC/networking
  -> IAM roles
  -> S3
  -> CloudFront
  -> RDS
  -> Load Balancers
  -> Auto Scaling
  -> ECR
  -> ECS/Fargate
  -> CI/CD
  -> CloudWatch
  -> Terraform
```

The most important achievement was not simply "hosting React".

It was understanding this chain:

```text
Application
   |
Docker
   |
Linux
   |
EC2
   |
Security Group
   |
AWS Network
   |
Internet
```

When something fails, you now have layers to investigate rather than randomly changing settings.

---

# 45. Final Phase 1 Summary

You successfully learned and practiced:

- launching EC2
- Ubuntu AMIs
- instance types
- vCPU and RAM
- EBS gp3 storage
- key pairs
- SSH
- private key permissions
- host fingerprints
- public vs private IP
- Security Groups
- Linux inspection commands
- Git installation
- Docker installation
- Docker Compose
- Docker port mapping
- Nginx inside Docker
- deploying a React application
- exposing the application to the internet
- checking RAM usage
- checking container memory
- checking CPU usage
- checking disk usage
- checking Docker disk usage
- EC2 stop vs terminate
- post-termination cleanup
- basic AWS cost hygiene
- User Data fundamentals
- systematic troubleshooting

This is the correct foundation before moving into deeper AWS networking and production architecture.
