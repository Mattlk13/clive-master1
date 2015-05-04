

### Update presenter notes with all this stuff
- Intro
  - My name / job / location
  - What my work history is
    - IT Helpdesk
    - Managed Hosting
    - Webmaster
    - System Admin
    - DevOps Guy
    - Infra Automation
  - What we are going to be talking about
    - Unikernels
    - Erlang/OTP
    - LING
    - EC2
    
    
- Part 1: Unikernels
  - Let's look at the Ubuntu stack for an application
    - Overview
      - Physical Machine
      - Hypervisor
      - Linux OS kernel
      - Generalized libraries
      - Generalized supporting services (ntp, ssh, snmp)
      - Application runtime env (JVM)
      - Parallel threads
      - Configuration files
      - Application binary
      
    - Metrics
      - Number of packages installed
      - Number of services running
      - Number of ports open
      - Number of drivers/libraries
      
    - What is the skillset needed to maintain a stack?
      - Customized, general knowledge
      - Lots of unknowns (we didn't write the software)
      - Who can know the whole thing?
    - What does this look like in terms of
      - Maintenance
        - Upgrades: So large that they 'could be anything'
      - Attack surface
        - Heartbleed
        - Shellshock
      - Complexity
      - Supporting systems
        - Patch management: Tools like spacewalk
        
  - Let's think about what we actually need
    - What are we actually using on the system?
      - Disk I/O drivers
      - Network I/O drivers
      - Compute drivers
      - The Erlang VM
      - Any explicitly included libraries in the App
      
    - What are we not using on the OS?
      - FDD Drivers
      - Python
      - libpango
      - SSH
      - kmod
    - What is the cost of keeping these (unused) things around?
      - More patching
      - More upgrades
      - Larger security footprint
      - Need a specialized team to run everything (OPS)
      - TL;DR: Uptime and Team throughput
      
      
  - What is a unikernel?
    - Your app code bundled with it's configuration information and required
      libraries (from a Library OS) and sealed in an immutable artifact that can
      be deployed to a public cloud.
    - Written against virtual device drivers instead of a POSIX kernel

  - What is actually in a unikernel?
    - Your compiled application
      - Compiler removes anything not needed
    - Your configuration information
    - Any required libraries you need to operate (statically linked)
  - Where / How does a unikernel run?
    - In the cloud!
    - Against hypervisor
       - This provides the hardware abstraction
       - Allows us to only consider one type of device manufacturer
    - Uses the network for inter-service communication
    
  - Specifically why is this better?
    - Smaller size of images
      - From mb to kb
    - More efficient resource utilization (RAM)
      - While each instance of a unikernel pre-allocates RAM, the hypervisor can
        still overcommit
        
    - The security footprint is much smaller
      - Type safety/checking makes it even safer (OCaml)
    - If something is not used (as the compiler sees it) it is not included
    - The team that built the image knows exactly what is in it and how to run it
      - Total domain knowledge
      - Not relying on Ops' black box
      - Since all teams are coding against the same interface (Xen) knowledge and
        internal libraries can be taken care of in small amounts of time.
    - Less complexity in the runtime env (where things break with consequenses)
    - The next logical step after containerization
      - Containers still implicitly have access to the host libs etc..
      - They share a kernel
      - They share resources at a kernel level instead of a hypervisor level
      
  - What are the challenges that can come with unikernels?
    - Your teams have more responsibility
    - Complexity has been shunted to the build/deployment environment instead of
      the runtime env
    - We can no longer stick to a single binary artifact that is then deployed
      to many environments with only changes of configuration.
        - Configuration is compiled in
    - Don't expect backwards compatibility
    - Immutiblity can be hard to think of at first
  - What are some of the implications of unikernels?
    - Teams know the entire app and how to run it (they built the whole thing)
    - Ops teams which were once busy maintaing the runtime env can focus on tools etc..
  
    - What kind of unikernels are there?
      - Generalised and somewhat backwards compatible
        - Rump kernel (BSD)
        - OSv (Linux)
        - Drawbridge (Windows)
      - Specialized purpose built projects
        - MirageOS (OCaml)
        - HalVM (Haskell)
        - LING (Erlang yey)
        
        
- Part 2: Building and Deploying a LING unikernel
  - Overview: What are we going to do?
    - Compile a very basic webserver in Erlang
    - Package that webserver as a Xen image
    - Deploy the image to Xen to test
    - Package the tested image as an AMI
    - Upload the AMI to EC2 and register it
    - Deploy a server running that AMI
  - What are the tools we are going to use?
    - Erlang / LING
    - IntelliJ with the Erlang/OTP Plugin
    - Vagrant
    - LING's railing utility
    - Basho's rebar utility
    - Xen hypervisor
    - Amazon's CLI tools
    - Amazon's Console

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

TODO:
Get IntelliJ configured for the webserver project
Slides for demo
Slides for wrapup


PV-GRUB: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/UserProvidedKernels.html
http://events.linuxfoundation.org/sites/events/files/slides/OSv_ACEU14.pdf
https://www.usenix.org/conference/nsdi15/technical-sessions/presentation/lorch
