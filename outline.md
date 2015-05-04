

## Look ma, no OS!

## By: Matt Bajor
- Current at rally
- Originally from NH
- Went from 
  - IT helpdesk (Buongiorno)
  - System Admin (Mojohost)
  - Webmaster (Mobile Media Cloud)
  - DevOps (Beatport)
  - Infra automation (Rally)
  
## TL;DR:
- Two parts
- Part 1: Intro to unikernels
  - Recap of our existing systems
  - What unikernels are.
  - What types of unikernels there are.
  - What has enabled this change.
  - Why they are better than our existing solutions
- Part 2: Demonstration of deploying a LING unikernel to EC2
  - Build an app
  - Package it for xen
  - Package it for EC2 as an AMI
  - Deploy an instance

## Part 1: Intro to unikernels (welcome to the future)
## What do our existing envs look like?

## Our current stack is
- Consists of
  - Hardware
  - Hypervisor
  - Linux Kernel
  - General Libraries
  - Userland Services
  - Application Runtime (JVM)
  - Parallel Threads
  - Application configuration files
  - Application code itself
- Designed to support lots of hardware
- Designed to support lots of software
- Designed to be compatible
- Forklifted from the 90s
- Too thick
- Ripe for a refactor!

## Look at all of the things!
- 95 Processes by ps ax
- 649 Packages by dpkg
- 31 Ports by netstat -l
- 50 Kernel modules
- Not fair because it's a xenserver but still...

## But we get all of that for free right?
- Stallman's GNU said software should be free..

## No.
- Staffing and running ops teams is tricky
- Ops is required because
  - Linux is a black box to a lot of people
  - That's because its so generized with such a wide range of support
  - There are tons of things running that a feature team doesn't use
  - Maintaining all of these things takes a lot of risky work
- Security teams are also expensive
  - Auditing 3rd party code is a big task
  - Lets only explicitly use it if we need it. Dont' rely on everything
    being installed everywhere.
  - Limit exposure to vulnerabilities:
    - no shell == no shellshock
    - no libssl == no heartbleed
    
# So what do we really need out of all this?
 
# The things we are actually using:
- Virtualized Disk I/O
- Virtualized Network I/O
- Virtualized CPU / RAM
- The application runtime
  - The Erlang VM
  - JVM
  - HHVM
- Libraries that provide system functionality the app relies on
- External, network-based services
- The application and it's configuration

# Not..
- Tape drive drivers
- Python (unless it's your runtime)
- libpango
- SSH
- kmod
- multi user support

# Keeping those things around does cost $$
- So lets only deploy and maintain what we have to

# What is a unikernel?
# **Unikernels**

# A unikernel is...
- An application compiled with everything it needs to operate
- This includes
  - Compiled appliction code
  - Required application libraries
  - Appliction configuration
- in combination with
  - The application runtime
  - Required system libraries (by a libos)
  - A RAMDisk and PV Kernel (the interface to the HV)
- all packaged up and sealed as an artifact that can be run on the H/V

# More specifically an Erlang LING unikernel contains
- The applications beam bytecode
- The applications compiled dependencies
- A version of the Erlang VM (LING)
- Any explicity included parts of Erlang's stdlib
  - mnesia
  - inets
  - ssl
- A block device for non-persistent disk access (mnesia)
- Any configuration information the app requires
  - A good way to handle this is to use service discovery
    with something like consul. This allows you to hard code
    a DNS address, but one that is updated dynamically

# and then...
- Is packaged using LING's railing tool
  - Will output a Xen image
  - This can be run against a xen server
- Is packaged using Amazon's EC2 tools
  - Will output an AMI
- Is deployed to EC2
  - Manually to show the proces
  
# There are two main types of unikernels:
- Purpose built (ASIC)
- Generilzed (CPU)

# Specialized & purpose built
- These are true unikernels
- Meant to run just one app and one app alone
- More performant, less complex, leaner, meaner
- Can even be compiler optimized! (OCaml/mirage)
- Examples are
  - LING (Erlang)
  - HalVM (Haskell)
  - MirageOS (OCaml)
  - Clive (Go)

# Generalized 'fat' unikernels
- These are much fatter and more complex than the purpose built type, but
  can support running unmodified Linux applications in a single process model
- They are fatter than specialized unikernels and contain more than just the
  app code and what it needs to run.
- Re-introduces common components that can be utilized in a security breach
- Still get a lot of the efficiency gains, but without the security and
  complexity gains
  
# I like the specialized ones
- If there was a time for a refactor, it's now

# But how is that possible?

# Paravirtualization & Netowork Interfaces
- Paravirtualization (used by Xen / Amazon) provides a virtualized and
  consistent interface for accessing hardware
- No more supporting 1000s of types of hardware
- Removes the need for a lot of complexity
- It's the new interface for an app (coming from POSIX)

- Network intefaces to applications remove the need for colocation
- Function programming makes us think about where to put side effects
  making it easier to run on an immutable infrastructure
- Everything happens over http (or at least tcp)

# ..oh yeah, and dropping backwards compat
- The true unikernel requires a refactoring of most apps
- Apps need to explicitly include what they need
  - This takes some time to figure out
  - But is easily replicated between teams as a pattern
  - hopefully with understanding of the software
- Luckily this can normally be done at the runtime level, making the work
  feasible
- We have forklifted things enough as mentioned before. Let's invest in
  efficiency, security, and reducing complexity. 
  
# Specifically, why is this better?

# In three words
- Efficiency
- Security
- Complexity

# Efficiency
- Images go from GB to MB or KB, reducing storage footprints
  - We no longer need the OS so the images are tiny, even when
    compared to Docker images
- No uneccessary duplication of supporting services
  - Reduction in RAM
  - Reduction in CPU
  - Example would be NTP or portmapperd or irqbalance or sshd
    - sshd would get you to the bastion and you connect from there
      if production access to the runtime is needed (unsupported)
  
# Security
- The attack surface of a running app is tiny. Since there are no
  'common' system libraries, an attacker has to rely on compromises
  in your actual application code and included libraries. There is no
  glibc or libssl or any other installed but unused tool to take advantage
  of.
- It's no longer hacking wordpress. It's exploiting remote memory vulnerabilities
  and buffer exploits that even once exploited, drop you into a land that
  has ONLY the pre-compiled things you needed to run the app.
- There are no tools on these images
  - No shells (other than the runtime)
  - No compilers
  - No system tools
  - No monitoring services except what your app (uniquely) exposes
- With no common tools or libraries other than the unique combination
  that makes the app run, it is very hard to exploit the system to
  recover anything of value
- In combination with a statically type-checked language (like OCaml)
  these images are near impossible to compromise.
  
# Complexity
- Removing the generalized OS removes the bulk of complexity
- The majority of the remaining complexity is moved out of production
  where currently admins and config management wreak havok, and into the
  build and test system which does not affect customers or revenue when
  failures occur.
- The team that built the app knows not only the app but how to run it as
  well. They are not dependent on another team to take care of that aspect
  for them. (No more black box)
  
# It's also the next logical step from containerization
- We went from
  - Mainframes
  - Servers
  - VMs
  - The cloud
  - Unikernels
- Unikernels attempt to take the great concepts of containerization to the next
  level
- They pick up where Docker has left off

# What are some of the challenges when adopting unikernels?

# Feature teams have more responsibility, so give them a devops
- No os maintained by another team
- Team has to have deeper knowledge of their apps' dependencies
- The team has more ownership through all envs. Don't huck it over the wall
- The team is primary POC for all failures, including prod
- Give the teams a devops to help with this effort
- It's easier to staff a devops on a feature team
- More fun, less stress, common goal not us vs them
- Transfer opsy knowledge throughout the team
- Build tools to help support the application, as well as help with
  best practices for deployment and building etc..
- The devops still remains part of a larger operations team in order to share
  tools and transfer knowlege, as well as standardize tools and techniques
  across teams (See spotify's scaling agile paper)
  
# Configuration managment is way different but more powerful
- Twelve factor app doesn't apply here
- Configuration is baked in requiring mutliple binaries
  - or maybe a complex service discovery layer
- It's worth a try because
  - Complexity is taken out of production
    - The artifacts running in prod just do one thing and are simple as hell
      to deploy. 
    - That complexity is moved to the build/test env where failures are ok
    - Better to have a complex build enviornment and a prod env that is as
      simple as possible
  - When using a compiler that supports it, the compiled code can be optimized
    to the point of dropping code that is configured not to run. This means
    that the only thing going out to prod is exactly what is needed and no
    more or less. The compiler (if it supports it) can also check for config
    errors in build and blow up there instead of just a service not restarting
    in prod.
    - When is the last time your configuration management removed unused
      packages?

# Cya backwards compatibility! Immutable infra is different.
- This next evolution will not be a forklift
- Did it from mainframes to datacenters and from datacenters to clouds
- This time we need to re-architect to reclaim efficency, security, and complexity
- Also immutable infrastructure is different
  - Requires different thinking
  - Segregating side-effects
  - (Almost) any part of the app can die at any time
  - Persistence requires more than a 777 directory

# But if we can do it
- Development teams will have a much better understanding of the full application
  they work on because things are in very small, head-sized chunks
    - Also there is no black box; teams see everything
- Operations teams can focus on tool building and escalations
- Project managers will have a better understanding of the real cost of
  software development, but it will be a steady pace
- Companies can horizontally scale feature teams as each team is self-sustainable
- Infrastructure Engineering / Ops will not be the blocking resource!

# Whats out there right now?

# ClickOS
- MiniOS based 
- Primarily for networking equipment
- Runs the Click Modular router

# Clive
- Research project status
- Runs Go
- Uses a modified compiler and set of libraries
- Appears to target bare metal (which implies KVM / Xen as well)
- Very cool lookign project

# Microsoft Drawbridge
- Just a research project
- Picoservices in Microsoft
- No real code to download

# HaLVM
- Haskell Lightweight Virtual Machine
- A simple platform for simple platforms (xensummit presentation)
- A port of the Glasgow Haskell Compiler
- Specialized unikernel, purpose built
- Originally made to prototype operating system components
- Grown to support a much larger set of use cases
- Developed by Galois

# LING (aka Erlang on Xen)
- Implementation of BEAM
  - Almost compatible
  - Almost as performant
- Runs erlang against the Xen hypervisor
- Supports most of the stdlib (you include what you need when packaging)
- Used in the zero footprint cloud
  - Instances can be spun up in hundreds of milliseconds
  - Webserver spun up when request is received, spun down after response
  - No requests == no servers
  - Zero footprint cloud
  - Zerg is an example

# MiniOS
- A library OS put out by Xen
- Is included in your project where needed
- Supports C
- I couldn't find a license

# MirageOS
- OCaml platform
- Seems to be the most promising
- What is OCaml?
- A few whitepapers about it

# OSv
- Generalized POSIX compatible unikernel
- Runs un-modified apps
- Version availble for JVM
- Sends data home
- Licensing is not 100% BSD I don't think
- Very cool for running normal POSIX apps

# Rump Kernels
- BSD Rump kernels are generalized bsd kernels
- Allows the running of slightly modified or unmodified apps
- Basically a library os that includes I/O and POSIX syscall handlers
- RAMP stack
  - Rump, Nginx, Mysql, PHP
  - Looks sweet

# And many more to come

# Part 2: Building and deploying a LING unikernel to EC2

# What are we going to be doing exactly?
- Write (paste) an Erlang app in IntelliJ.
- Compile and run the app in IntelliJ to test it.
- Package the app as a Xen image with LING's railing.
- Run the Xen image on Xen in Virtualbox to test.
- Package the Xen image as an AMI.
- Upload and register the AMI.
- Deploy an EC2 instance from AMI.
- ...
- Profit.
  
# And what will we be using to do it?
- Erlang/OTP 17
- IntelliJ IDEA & Erlang/OTP Plugin
- railing: LING's packaging utility
- rebar: Basho's Erlang build tool
- Vagrant (to run Ubuntu
- Ubuntu (to run Xen)
- Xen (to run our LING artifact)
- AWS CLI tools
- AWS GUI

What I need to do
- Give overview of the webserver directory structure
- Open up the source code in IntelliJ
- Run the code in IntelliJ, show that it works
  - Fire up observer:start()
- Log into the vagrant vm
- Talk about the vagrant vm
  - Problems with kernel support (can't get vmware/vbox tools installed)
  - Some manual steps
  - Will work on solifiying it
  - Has Xen installed
- Walk through build.sh
  - rebar g-d clean compile
    - outputs to ebin
  - railing image --name vmling
    - generates the domain_config
- Walk through the run.sh script
  - sudo xl create -c webserver_config
  - sudo xl list # in another terminal
  - ping 10.100.199.230
  - curl 10.100.199.230:8080
- Walk through the build_ec2.sh script
  - make an image
  - create an ext2 fs on the image
  - mount the image and add
    - menu.lst
    - vmling unikernel
    - sync and umount
  - ec2-bundle-image -c info/cert-26PJBUNPW4LYLYUOXHY66Q6ELQZGLKER.pem \
      -k info/pk-26PJBUNPW4LYLYUOXHY66Q6ELQZGLKER.pem \
      -u 676336494080 \
      -i webserver.img -d data -r x86_64
- Walk through upload.sh
  - ec2-upload-bundle -b unikernel-demo/webserver-master \
      -m data/webserver.img.manifest.xml \
      -a <ACCESSKEY> \
      -s <Secret Key> \
      --location us-west-2
- Walk through register.sh
  - ec2-register \
      --aws-access-key <ACCESSKEY> \
      --aws-secret-key <Secret Key> \
      --region us-west-2 \
      unikernel-demo/webserver-master/webserver.img.manifest.xml \
      -a x86_64 \
      --name webserver-unikernel-master \
      --kernel aki-fc37bacc \ # pv-grub-hd0_1.03-x86_64.gz
      --virtualization-type paravirtual


