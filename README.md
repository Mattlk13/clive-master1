## Denver Erlang / Elixir

### Look ma, no OS! Deploying an Erlang/OTP application as a Ling unikernel in EC2

Matt Bajor from Rally will be talking about the ins and outs of unikernels as
well as how to build an Erlang/OTP application into a Ling (aka Erlang on Xen)
AMI and deploy it to EC2. 

### Outline

- Intro
  - What is a unikernel?
    - What are it's origins?
    - Why is it so great?
    - What kind of unikernels are there?
    - Well if there's no complexity in the operating environment, where did it go?
      - compile time
      - orchestration time
      - why is that better than runtime?
  - What is immutable infrastructure?
    - Why is it great?
    - How do you work with it?
  - What is Ling (aka Erlang on Xen)?
    - What does it support?
      - ERlang/otp
      - Xen
      - Goofs / block devices
      - ec2
    - How does it compare to BEAM?
      - less performant
      - less mature
  - What are we actually deploying?
    - an erlang app
    - on a xen dom
    - with as little else as possible
    
- Building the app
  - Will be a webserver
  - Install rebar
    - How to integrate it into your project
    - How to configure it for your project
  - Generate the templates
  - Add deps
  - Generate / run a release to show it in that form
  - Build xen image with railing
  - Make into ami
- Building the platform
  - Walk through the packer stuff
  - Walk through the vagrant stuff
    - Why can't we use a shared folder?