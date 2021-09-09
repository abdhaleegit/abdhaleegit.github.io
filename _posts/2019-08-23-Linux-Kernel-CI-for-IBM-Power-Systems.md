# Open source development model
In a typical open source development model a piece of code written by developers go through several hops before it is merged into a particular Linux distribution release which can be consumed by a IBM customer. This entire process takes anywhere from 6 to 9 months.

![](https://github.com/abdhaleegit/abdhaleegit.github.io/raw/master/resource/opensourcemodel.jpg)

# FVT test challenges
FVT teams predominantly have been focusing on Linux distribution code validation. With changes in Linux on Power business direction and recent Linux on Power strategy, support matrix for test teams have grown exponentially. It was increasingly becoming difficult to validate several combination of Linux distribution releases, operating environment, firmware and hardware platforms. 

# Shift left test strategy
To tackle this problem a Shift Left strategy with focus on upstream testing is implemented, that helped in: 
   * Finding software defects early, reducing the defect cost down stream 
   * Building a solid set of new test cases along with development teams. These test case can then be used during the Linux distribution testing phase reducing the regression test effort. 
   * Providing periodic feedback to a developer on functional regressions.

# Automation is the key
To support this new strategy, an end to end automated continuous integration framework aka linux kernel CI aka K-Bot [linuxci](https://github.com/linuxci/linuxci) was developed to provide quick feedback on build breaks, functional and performance regressions. 

![](https://github.com/abdhaleegit/abdhaleegit.github.io/raw/master/resource/devops.jpg)

# How does K-Bot works ?
The CI framework involves multiple stages such as Subscriber, Scheduler, Queuer and Runner, the backend code is written in python and jenkins as front end, it allows periodic automated testing of a given kernel git tree. The framework facilitates kernel build, boot and execute a set of test against the booted kernel on the Power machines available.

![](https://github.com/abdhaleegit/abdhaleegit.github.io/raw/master/resource/ciproject.jpg)

# The Subscriber
The Project [Subscriber](https://github.com/linuxci/linuxci/blob/master/jenkins-ci/subscription.py) allowes users to subscribe their test machine with a git tree and tests to be run against it. The kernel building process starts based on subscription details such as kernel configuration, patches and the frequency of building it, all the subscribers data is stored in json database file.

![](https://github.com/abdhaleegit/abdhaleegit.github.io/raw/master/resource/subscriber.jpg)

# The Scheduler
Everday the project [Scheduler](https://github.com/linuxci/linuxci/blob/master/jenkins-ci/scheduler.py) reads the subcribers json file, identify the jobs scheduled for the day and add it to Queue file for the executions

![](https://github.com/abdhaleegit/abdhaleegit.github.io/raw/master/resource/scheduler.jpg)

# The Queuer
The project [Queuer](https://github.com/linuxci/linuxci/blob/master/jenkins-ci/jobqueuer.py) periodically reads the job queue file, Reserves the system under test and spawns the job over Runner for actual build, it also allows Machine Parallelism facility, wherein multiple jobs are being scheduled in parallel on all available machines which resulted subsequent reduction of overall build time.

![](https://github.com/abdhaleegit/abdhaleegit.github.io/raw/master/resource/Queuer.jpg)

# The Runner
Project [Runner](https://github.com/linuxci/linuxci/blob/master/run_test.py) is the main wrapper that builds, boots and runs the actual tests and the results for each run will be available as Jenkins Report and Slack Notification/mail

# Test cases 
CI leverages Avocado test framework for the test runs, avocado is an open source test framework with easy to use testcases covering Kernel, Memory, CPU, FileSystem, Performance, Storage, RAS. Etc. A new testcase can be added to Avocado and will be used by CI as plug and play facility.

    ```
    Avocado : https://github.com/avocado-framework/avocado
    Avocado-misc-tests : https://github.com/avocado-framework-tests/avocado
    ```

# CI prerequisites
Required files on jenkins server under `/home/jenkins/userContent/`

    ```
    korgnode : contains list of machines under test, user needs to create this
    machineQ : contains machine names of current jobs running
    subscribers.json: CI creates this to store all machines subscribed to CI
    KORG#? : Data file gets created for each subscription contains json files of subscription
    KORG#?/KORG#?.json: detailed parameter related to job under test
    ```

CI Code is upstreamed [linuxci](https://github.com/linuxci/linuxci) under GNU General Public License v2.0

# Summary
As of today we have 6000+ test scenarios covering both kernel and IO subsystem and growing.

CI has completed 1000+ builds on various linux git trees on different hardware/firmware/linux combination with out manual intervention.

Overall 400+ valid defects is reported to mailing list and the fixes from IBM and community has been verified and merged upstream, saving huge $(dollars) down stream.



Thanks for reading my blog!
