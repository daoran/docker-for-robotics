# Motivation

Author: [Tobit Flatscher](https://github.com/2b-t) (August 2021 - February 2023)



## 1. Overview

When I (as a robotics developer) tell people that none of my computers have ROS installed on them, at least not on the host system, I mostly get puzzled looks. But **using Docker as an isolated development environment is a strategy that is shared by most leading companies and research institutions in robotics**. First and foremost we have **PickNik robotics** (they have customers including Google, Amazon, Samsung, Intel and NASA), a consulting agency for robotics that employs arguably some of the most brilliant and respected robotics engineers, that recommend using Docker for development as well as deployment as [outlined in this presentation of theirs](https://picknik.ai/ros/robotics/docker/2021/07/20/Vatan-Aksoy-Tezer-Docker.html). **Open Robotics** (OSRF), the main driver behind ROS has been using Docker as the backbone of their [build infrastructure](https://github.com/ros-infrastructure/ros_buildfarm) almost ever since Docker came out in 2013. They also provide the [official ROS images on Dockerhub](https://github.com/osrf/docker_images) as well as tools such as [Rocker](https://github.com/osrf/rocker) for working with graphic user interfaces from inside a Docker in a portable way. The ROS Industrial industrial training uses Docker in certain parts of its tutorials such as for [cloud computing](https://industrial-training-master.readthedocs.io/en/melodic/_source/session6/Docker-AWS.html). [Pal Robotics](https://github.com/pal-robotics/pal_docker_utils) and [Stogl robotics](https://rtw.stoglrobotics.de/master/docker/index.html) offer utilities for developing robotic applications with Docker. Furthermore you will find workflows and tools for leading research Institutions like [**Fraunhofer IPA**](https://github.com/ct2034/dockeros), [**DFKI**](https://github.com/dfki-ric/docker_image_development), **IIT** [[1]](https://github.com/diegoferigo/development-iit) [[2]](https://github.com/diegoferigo/ros-kubernetes), [**INRIA**](https://gitlab.inria.fr/locolearn/public/docker_inria_wbc) and [**MIT CSAIL**](https://roboticseabass.com/2021/04/21/docker-and-ros/) online. And finally leading companies like **Boston Dynamics** [[3]](https://dev.bostondynamics.com/docs/payload/docker_containers#) [[4]](https://dev.bostondynamics.com/docs/payload/spot_core_portainer) and [**Sony**](https://github.com/fujitatomoya/ros_k8s) use it for deployment on commercial products.

Over the last couple of years I have collaborated professionally with different companies and robotics research institutions and met quite a lot of people that would not understand the benefits or even worse would oppose the technology per se without actually having used it. But I managed convince most people so far to give it a go and they immediately see the advantages of Docker for robotics and implement it in their workflow as well. I have not yet quite understood why Docker as a technology has such a negative connotation. Maybe it is simply a lack of understanding (people have tried it and have failed in one way or the other), maybe this is historically grown. Anyways, my goal of this guide is to show best practices that I have found over the last couple of years (working with Docker for robotic software development and by examing workflows of people I worked with or what I could find online). In my opinion Docker as a technology is essential for developing robotics in an efficient and reliable manner.

As said I do not have ROS installed on my development computers and do not plan to do so. Instead of running a **vanilla system with Docker and my Visual Studio Code IDE installed**. There are a few tools for power management and networking installed on my host machine but all project related-code is put inside a Docker. **Every single workspace has its own dedicated Docker**, I have Dockers for testing particular sensors and robot stacks, some in ROS1, others in ROS2, that all can be run on my host system (currently Ubuntu 22.04). This way I can continue to keep up with the most recent host system and might revive an old retired project at any point (e.g. I have used it to test [traclabs' Affordance Templates stack](https://traclabs.com/projects/affordance-templates/) for ROS Indigo) or try a different distribution. If I need a new container that uses this sensor or robot I either copy the relevant parts from the corresponding Dockerfile or I leverage multi-layer builds to generate the new image starting from the sensor and robot images. Furthermore I might [overlay different workspaces on top of each other](http://wiki.ros.org/catkin/Tutorials/workspace_overlaying) or orchestrate multiple workspaces with Docker-Compose.



## 2. Why should I use Docker when developing robotics software?

- **Compatability**:
  - Working with robots in ROS generally requires working with **different Ubuntu distributions**. With Docker one can avoid having multiple partitions on the same system and instead **start different distributions from the same host system**. This way for example a ROS Indigo stack can still be run on a Ubuntu 22.04. This is very important as sooner or later robotic stacks have to be retired as they are not state-of-the-art anymore (sensors change, ROS gets replaced by ROS2) but nonetheless one should still be able to run this legacy code without having a legacy computer for every single distribution laying around.
  - Furthermore you can have a **non-Debian-based Linux host system** and nonetheless work with ROS if you prefer another Linux distribution over Ubuntu as your daily driver, yet have the convenience of working with Ubuntu when dealing with ROS.
- **Replicability**:
  - Robotic stacks often involve a large amount of packages. A single person often won't accomplish much if the application is complex and/or should be deployed commercially. **Multiple people** working on the same workspace should have an **identical set-up with the same software versions**. At the same time each contributor should have a central point were IPs and other parameters can be modified (which do not have to be commited to the common repository everybody is working on together).
  - Working with **multiple robotic stacks** even on the same Ubuntu distribution often requires working with **different versions of the same library** for different projects. With Docker this is no issue. One can further perform version pinning, meaning enforce a certain version of a package to increase replicability on another computer.
  - As software engineers and researcher develop on a machine they will install an immense amount of software that often is not needed at all but is never uninstalled when not needed anymore. Even an experienced programmer under stress will simply search for a solution on Stackoverflow, take the post with the most upvotes - even if they do not fully understand it - and give it a go. This means no developers system is actually clean, most of them actually contain a lot of useless garbage that gets worse and worse over time. This useless software might impact other code, that might break it or is required to install other packages without the developer knowing, or is incompatible with other packages or certain versions. Working without a fresh system means always carrying this installation history across projects. This leads to situation where code compiles on one computer but does not compile for another person with a slightly different set-up ("But it compiles on my computer..."-situations). From my experience most of the time the reason for these situations are **implicit dependencies and restrictions** that the developer is not aware of and not the fault of the person trying to replicate the set-up. Therefore you as a programmer (or researcher) should be interested in using technologies such as Docker: After all it is your responsability to make your software project and research as replicable as possible, and isolate any previous modifications from your current project.
- **Isolation**:
  - Working with different robotic stacks generally also means working on **different networks**. This means robotic developers will generally add new robots they need to communicate with inside their **`/etc/hosts`** configuration. This might lead to collision of IPs and results in different entries having to be commented or uncommented in order to be able to work with a particular robot. Similarly most people add certain definitions and aliases inside their **`.bashrc`** that they will comment and uncomment depending on the project they are currently working on. This is very inconvenient, cumbersome and error-prone. When working with a Docker these **configuration files are specific to a project** and do not leak into workspaces of other robots.
  - This isolation can also be very useful for parallelizing simulations. With Docker multiple ROS cores can be run in parallel on the same host system by letting each container have its own network and not exposing it to the other containers.
- **Continuous integration**: Docker is an essential building stone of most continuous integration pipelines. Maintaining a description of the environment that code should be run in is essential for testing your code on a remote server.
- **Fast deployment**: Instead of installing your code from source on a system, installing multiple Debian packages and configuring the network manually you can encapsulate all this in a Docker container and deploy the Docker container as a whole or orchestrate multiple Docker containers (e.g. one container for each node or group of nodes) on the same computer. This means that deploying might only reduce to pulling a single Docker image with the size of a couple of dozen megabytes.

As guarantueeing these characteristics without the use of Docker is close to impossible, there are many companies and institutions where a robotic software stack only runs only on a particular computer. Nobody is allowed to touch (or at least make significant changes to) that system as if it breaks that robotic system would be rendered useless (or at least inoperable for some time). In my opinion such thing should never happen and is a strategic failure per se. Any robot should be replicable just from a set-up that can be found online in your version control system in a semi-automated fashion without having to go through a long manual set-up routine. A pipeline based on Docker can help with that. **Even if you do not deploy a robot with Docker you should maintain one as a back-up solution** because you never know, it is software.

## 3. Why is Docker important in particular for academic and research institutions?

In particular research institutions are in desperate need for container-based solutions such as Docker - even more than companies in my opinion:

A company usually is made of a **stable** workforce of **experienced** engineers that work together on a **few** homogeneous products. Any decent company will work towards **common tools** and will have **mechanisms** in place that should **guarantee code quality**. These products will be maintained until they reach their lifecycle after several year and are either retired or replaced, resulting in an overall small variability.

Academic and research institutions are the complete opposite: Workers and students generally are **fluctuating**, might be motivated and brilliant but are at a start of their career, generally **lacking the experience** and might have very interdisciplinary backgrounds (which is in particular true for robotics where people might come from software, mechanical, electrical or biomedical engineering or might have an even more interdisciplinary background). Furthermore there are close to **no mechanisms in place that guarantee code quality**. The tools the students and research engineers will use are very **heterogeneous**. A large part of the developed code will **not be actively maintained** for an extended period as there are no resources for doing so after a project (and funding) ends. At a certain point projects have to be **retired** nonetheless they should be left **in a replicable state**. Many universities and research institutions fail to have mechanisms in place that help with this. I have already seen many projects die because the main developer left not only taking with themselves the know-how but also any chance of replicating his/her work as the code only worked on their machine. This is a fatal loss for any institution in my opinion as they lost the know-how and significantly reduced their chance of replicating the work (which means after all filling the gap and continuing in that research direction). Many research institutions I had to deal with suffer from a significant slow down due to this which significantly slows down their daily business and growth.

If you start on a new project it is incredibly frustrating to take over a complex robotic project from somebody that lacks documentation. Even if you have access to their working computer you will likely have to modify their `.bashrc` and network set-up. If you further have to start over with another computer you will have to search for dependencies, fiddling with different versions of libraries and implicit undocumented dependencies. This **slows down and demotivates**. This is particularly true for Bachelor and Master students that are only with these institutions for a limited amount of time. Some might have certain constraints and want to finish their work in a certain limited time frame. This means if you are not able to get them started quickly with a project they will not only lose motivation but also lose valuable time that they could do on research for fixing set-up problems that are structural and actually none of their business. PhD students in robotics might have different backgrounds and might not be familiar with best-practices. Generally they leave after some time without anybody having a complete overview of what they did: After all their supervisor is mainly interested in the scientific output and might not have the time to have a look in detail at the state of the documentation. It is essential to familiarize them with a standard workflow that allows to replicate their work. After all one of the core idea of any research is making findings **replicable** and not only making findings. This is in particular important for medical research where [paradoxically most research findings are actually false](https://journals.plos.org/plosmedicine/article?id=10.1371/journal.pmed.0020124)!

After all the time effort for setting up and maintaining a Docker is not bigger than installing the libraries by hand but it only works if you immediately introduce your students and junior researchers to these technologies and not only in the middle or towards the end of the project (nobody remembers what they installed half a year, let alone three or four years ago). Furthermore Docker allows you to reset quickly, go back to a previous state. This quickly pays off in particular if more than a single computer has to be used. And the gained time can be spent on research.

Building this infrastructure and workflows also as an academic institution is a long-term investment, of similar importance to building frameworks rather than single purpose code for a project.



## 4. What are the drawbacks of Docker?

For robotics **software development I do not see any drawbacks with using Docker**. After all Docker is a mature technology that is used successfully in many other fields. From my experience there are many prejudices surrounding Docker as a technology but graphic user interfaces, real-time capable code, working with hardware can all be worked around in a reliable and portable manner. This guide will explain how this can be done.

The only messy thing that one has to watch out for is actually user rights. Giving root privileges to a Docker user is probably not a good idea if you deploy the container, as an intruder might gain access to the host system in particular when running a container as `privileged`. On the other hand using another user might result in problems when sharing volumes as pointed out later on.