# Considerations when transitioning workloads to Ampere processor based instances or bare-metals

Ampere processors power the Arm64 instances on many major Cloud Service Providers, among them A1 and A2 instances on Oracle Cloud Infrastructure, Dps, Dpds, Eps, and Epds instances on Azure, and Tau T2a and C3a instances on Google Cloud Platform. These instances type provide the best price-performance for a wide variety of Linux-based workloads. Examples include application servers, micro-services, CPU-based machine learning inference, video encoding, electronic design automation, gaming, open-source databases, and in-memory caches. In most cases transitioning to Ampere instances is as simple as updating your infrastructure-as-code to select the new instance type and associated Operating System (OS) images. Ampere also partners with the best system providers the world has to allow building your own infrastructure. The bare-metal platforms we use to provide seamless integrations are detailed listed by manufacturer and model can be found [here](https://amperecomputing.com/systems/altra). However, because Ampere processors implement the Arm64 instruction set, no matter you decide to use instances on CSPs or build your own infrastructures, there can be additional software implications. This transition guide provides a step-by-step approach to assess your workload to identify and address any potential software changes that might be needed.

## Introduction - identifying target workloads

The quickest and easiest workloads to transition are Linux-based, and built using open-source components or in-house applications where you control the source code. Many open source projects already support Arm64 and by extension Ampere processors, and having access to the source code allows you to build from source if pre-built artifacts do not already exist. There is also a large and growing set of Independent Software Vendor (ISV) software available for Arm64 (a non-exhaustive list can be found [here](isv.md). However if you license software you’ll want to check with the respective ISV to ensure they already, or have plans to, support the Arm64 instruction set.

The following transition guide is organized into a logical sequence of steps as follows:

* [Learning and exploring](#learning-and-exploring)
    * Step 1 -  [Optional] Understand the Ampere Processor and review key documentation
    * Step 2 - Explore your workload, and inventory your current software stack
* [Plan your workload transition](#plan-your-workload-transition)
    * Step 3 - Install and configure your application environment
    * Step 4 - [Optional] Build your application(s) and/or container images
* [Test and optimize your workload](#test-and-optimize-your-workload)
    * Step 5 - Testing and optimizing your workload
    * Step 6 - Performance testing
* [Infrastructure and deployment](#infrastructure-and-deployment)
    * Step 7 - Update your infrastructure as code
    * Step 8 - Perform canary or Blue-Green deployment

### Learning and exploring

**Step 1 - [Optional] Understand Ampere Processors and review key documentation**


1. [Optional] Start by reviewing [product line of Ampere Processors](https://amperecomputing.com/products/processors), [Ampere Developer Center](https://amperecomputing.com/developers) and [Ampere Solutions for the Sustainable Cloud](https://amperecomputing.com/solutions), which will give you an overview of the Ampere processors and some insights on how to design, build and deploy varies of applications on Ampere processor based platform.
2. [Optional] Keep on learning by watching [Ampere Strategy & Product Roadmap Update, 2023]([https://youtu.be/k2l9SJpNdrk?si=4ewtIilQosIEkU_J](https://youtu.be/k2l9SJpNdrk?si=4ewtIilQosIEkU_J)to better understand Ampere long-term commitment to high performance, high efficiency compute.
3. Get familiar with the rest of this [Getting started with Ampere Processors repository](README.md) which will act as a useful reference throughout your workload transition.


**Step 2 -  Explore your workload, and inventory your current software stack**

Before starting the transition, you will need to inventory your current software stack so you can identify the path to equivalent software versions that support Arm64. At this stage it can be useful to think in terms of software you download (e.g. open source packages, container images, libraries), software you build and software you procure/license (e.g. monitoring or security agents). Areas to review:

* [Operating system](os.md), pay attention to specific versions that support Arm64 (usually more recent are better)
* If your workload is container based, check container images you consume for Arm64 support. Keep in mind many container images now support multiple architectures which simplifies consumption of those images in a mixed-architecture environment.
* All the libraries, frameworks and runtimes used by the application.
* Tools used to build, deploy and test your application (e.g. compilers, test suites, CI/CD pipelines, provisioning tools and scripts). Note there are language specific sections in the getting started guide with useful pointers to getting the best performance from Ampere processors.
* Tools and/or agents used to deploy and manage the application in production (e.g. monitoring tools or security agents)
* The [Ampere Porting Advisor](https://github.com/AmpereComputing/ampere-porting-advisor) is an open-source command line tool that analyzes source code and generates a report highlighting missing and outdated libraries and code constructs that may require modification along with recommendations for alternatives. It accelerates your transition to Ampere processors by reducing the iterative process of identifying and resolving source code and library dependencies
* This guide contains language specifics sections where you'll find additional per-language guidance:
  * [C/C++](c-c++.md)
  * [Go](golang.md)
  * [Java](java.md)
  * [.NET](dotnet.md) 
  * [Python](python.md)
  * [Rust](rust.md)

As a rule the more current your software environment the more likely you will obtain the full performance entitlement from Ampere processors.

For each component of your software stack, check for Arm64 support. A large portion of this can be done using existing configuration scripts, as your scripts run and install packages you will get messages for any missing components, some may build from source automatically while others will cause the script to fail. Pay attention to software versions as in general the more current your software is the easier the transition, and the more likely you’ll achieve the full performance entitlement from Ampere processors. If you do need to perform upgrades prior to adopting Ampere processors then it is best to do that using an existing x86 environment to minimize the number of changed variables. We have seen examples where upgrading OS version on x86 was far more involved and time consuming than transitioning to Ampere processors after the upgrade. For more details on checking for software support please see Appendix A.

Note: When locating software be aware that some tools, including  GCC, refer to the architecture as AArch64, others including the Linux Kernel, call it arm64. When checking packages across various repositories, you’ll find those different naming conventions.

### Plan your workload transition

**Step 3-  Install and configure your application environment**

To transition and test your application, you will need a suitable environment running on Ampere processors. Depending on your execution environment, you may need to:

* Obtain or create an Arm64 image to boot your Ampere processor based instance(s) from. Depending on how you manage your VM images, you can either start directly from an existing reference VM images for Arm64, or you can build a Golden VM image with your specific dependencies from one of the reference images (see [here](os.md) for a full list of supported OS’ with CSP links) ;
* If you operate a container based environment, you’ll need to build or extend an existing cluster with support for Ampere processor based instances. Mixed x86 and Arm64 dpeloyment and autoscaling are widely support by Azure, GCP and OCI. For AKS, you can [optimize costs for your cluster](https://learn.microsoft.com/en-us/azure/aks/best-practices-cost) by adding Ampere processor based VMs to your cluster. For GCP, [Migrate x86 application on GKE to multi-arch](https://cloud.google.com/kubernetes-engine/docs/tutorials/migrate-x86-to-multi-arch-arm) with Ampere processors can be a choice. for OCI, you can [create application deployments that are seamlessly portable between the Ampere A1 based kubernetes clusters and x86(Intel and AMD) based clusters](https://docs.oracle.com/en/learn/arm_oke_cluster_oci/#introduction).
* Complete the installation of your software stack based on the inventory created in step 2.
    * Note: In many cases your installation scripts can be used as-is or with minor modifications to reference architecture specific versions of components where necessary. The first time through this may be an iterative process as you resolve any remaining dependencies. 


**Step 4 - [Optional] Build your application(s) and/or container images**

Note: If you are not building your application or component parts of your overall application stack you may skip this step.

For applications built using interpreted or JIT’d languages, including Java, PHP or Node.js, they should run as-is or with only minor modifications. The repository contains language specific sections with recommendations, for example [Java](java.md), [Python](python.md), [C/C++](c-c++.md), [Golang](golang.md), [Rust](rust.md) or [.Net](dotnet.md). Note: if there is no language specific section, it is because there is no specific guidance beyond using a suitably current version of the language as documented [here](README.md#recent-software-updates-relevant-to-arm64) (e.g. PHP Version 7.4+). .NET-core is Arm64 ready and a great way to benefit from Ampere processor based system, this [page](https://dotnet.microsoft.com/en-us/download/dotnet) covers .NET versions supports Arm64, latest .NET8 is recommended.

Applications using compiled languages including C, C++ or Go, need to be compiled for the Arm64 architecture. Most modern builds (e.g. using Make) will just work when run natively on Ampere processor based system, however, you’ll find language specific compiler recommendations in this repository: [C/C++](c-c++.md), [Go](golang.md), and [Rust](rust.md).

Just like an operating system, container images are architecture specific. You will need to build arm64 container image(s), to make the transition easier we recommend building multi-arch container image(s) that can run automatically on either x86-64 or arm64. Check out the [container section](containers.md) of this repository for more details and this [guide](https://docs.docker.com/build/building/multi-platform/) provides a detailed overview of building multi-architecture container image for "Build once, deploy anywhere" purpose.

You will also need to review any functional and unit test suite(s) to ensure you can test the new build artifacts with the same test coverage you have already for x86 artifacts.

### Test and optimize your workload

**Step 5 - Testing and optimizing your workload**

Now that you have your application stack on Ampere processors, you should run your test suite to ensure all regular unit and functional tests pass. Resolve any test failures in the application(s) or test suites until you are satisfied everything is working as expected. Most errors should be related to the modifications and updated software versions you have installed during the transition (tip: when upgrading software versions first test them using an existing x86 environment to minimize the number of variables changed at a time. If issues occur then resolve them using the current x86 environment before continuing with the new Ampere processor environment). If you suspect architecture specific issue(s) please have a look to our [C/C++ section ](c-c++.md) which documents them and give advice on how to solve them. If there are still details that seem unclear, please reach out to your Ampere account team, or to the Ampere support for assistance.

**Step 6 - Performance testing**

With your fully functional application, it is time to establish a performance baseline on Ampere processors. In many cases, Ampere processors will provide performance and/or capacity improvements over x86-based system.

One of the major differences between Ampere processor based system and other x86-based system is their logical-to-physical-core mappings. Every single core reported by OS on a Ampere processor is a physical core, and there is no Simultaneous Multi-Threading (SMT). Consequently, Ampere processor provides better linear performance scalability in most cases. When comparing to existing x86-based system, we recommend fully loading both instance types to determine the maximum sustainable load before the latency or error rate exceeds acceptable bounds. For horizontally-scalable, multi-threaded workloads that are CPU bound, you may find that the Ampere processor based system are able to sustain a significantly higher transaction rate before unacceptable latencies or error rates occur.

During the transition to Ampere processor based instances, if you are using autoscaling provided by CSPs, you may be able to increase the threshold values for monitoring tool's alarms that invoke the scaling process. This may reduce the number of Ampere processor based instances now needed to serve a given level of demand.

Important: This repository has sections dedicated to [Optimization](optimizing.md) and a [Performance Runbook](perfrunbook/graviton_perfrunbook.md) for you to follow during this stage.

If after reading the documentation in this repository and following the recommendations you do not observe expected performance then please reach out to your Ampere account team, or send email to [support@amperecomputing.com](mailto:support@amperecomputing.com) with details so we can assist you with your performance observations.


### Infrastructure and deployment

**Step 7 - Update your infrastructure as code**

Now you have a tested and performant application, its time to update your infrastructure as code to add support for Ampere processor based instances or clusters. This typically includes updating instance types, image IDs, autoscaling-groups to support multi-architecture on CSPs or provisioning clusters by infrastructure-as-code tools, and finally deploying or redeploying your infrastructure.

**Step 8 - Perform canary or Blue-Green deployment**

Once your infrastructure is ready to support Ampere processor based system, you can start a Canary or Blue-Green deployment to re-direct a portion of application traffic to the Ampere processor based instances. Ideally initial tests will run in a development environment to load test with production traffic patterns. Monitor the application closely to ensure expected behavior. Once your application is running as expected on Ampere processor based system you can define and execute your transition strategy and begin to enjoy the benefits of increased price-performance.


### _Appendix A - locating packages for Arm64/Ampere-Processors_

Remember: When locating software be aware that some tools, including  GCC, refer to the architecture as AArch64, others including the Linux Kernel, call it arm64. When checking packages across various repositories, you’ll find those different naming conventions, and in some cases just “ARM”.

The main ways to check and places to look for will be:

* Package repositories of your chosen Linux distribution(s). Arm64 support within Linux distributions is largely complete: for example, Debian, which has the largest package repository, has over 98% of its packages built for the arm64 architecture.
* Container image registry. DockerHub allows you to search for a specific architecture ([e.g. arm64](https://hub.docker.com/search?type=image&architecture=arm64)).
    * Note: Specific to containers you may find an amd64 (x86-64) container image you currently use transitioned to a multi-architecture container image when adding Arm64 support. This means you may not find an explicit Arm64 container, so be sure to check for both as projects may chose to vend discrete images for x86-64 and Arm64 while other projects chose to vend a multi-arch image supporting both architectures.
* On GitHub, you can check for arm64 versions in the release section. However, some projects don’t use the release section, or only release source archives, so you may need to visit the main project webpage and check the download section. You can also search the GitHub project for “arm64” or “AArch64” to see whether the project has any arm64 code contributions or issues. Even if a project does not currently produce builds for arm64, in many cases an Arm64 version of those packages will be available through Linux distributions or additional package repositories (e.g. [EPEL](https://docs.fedoraproject.org/en-US/epel/)). You can search for packages using a package search tool such as [pkgs.org](https://pkgs.org/).
* The download section or platform support matrix of your software vendors, look for references to Arm64 or AArch64.

Categories of software with potential issues:

* Packages or applications sourced from an ISV may not yet be available for Arm64. We recommand to check latest release notes of the package/application to see Arm64 supportance was included or not. A non-exhaustive list of some ISV software can be found in [here](isv.md).

* The Python community vend lots of modules built using low level languages (e.g. C/C++) that need to be compiled for the Arm64 architecture. You may use modules that are not currently available as pre-built binaries from the Python Package Index. Open-source communities are accelerating the availability for most popular modules on Arm64. In the meantime we provide specific instructions to resolve the build-time dependencies for missing packages in the [Python section](python.md#1-installing-python-packages) of the Ampere Processors Getting Started Guide.


If you find other software lacking support for Arm64, please let your Ampere team know, or send email to [support@amperecomputing.com](mailto:support@amperecomputing.com).
