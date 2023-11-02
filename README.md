# An automated workflow for implementing CIS Docker Benchmark

This solution provides a workflow for implementing [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker) v1.5.0 for [Amazon Elastic Container Service (ECS)](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html).
Per CIS documentations:
> CIS Docker Benchmark, provides prescriptive guidance for establishing a secure configuration posture for Docker Engine v20.10. This guide was tested against Docker Engine 20.10.20 on RHEL 7 and Ubuntu 20.04.

## What is covered?

CIS Docker Benchmark has 7 sections:

1. Host Configuration
2. Docker daemon configuration
3. Docker daemon configuration files
4. Container Images and Build File Configuration
5. Container Runtime Configuration
6. Docker Security Operations
7. Docker Swarm Configuration

Sections 1 through 3 are covered by this solution. Section 6 controls are mainly driven by business requirements and needs, and section 7 is not applicable to ECS. For a complete list of controls excluded by this solution, see [Excluded controls](#excluded-controls).

## Architecture diagram

![Architecture Diagram](/assets/Architecture%20diagram.png)

### Tools and services

- [Ansible](https://docs.ansible.com/ansible/latest/index.html): implements Secctions 1 through 3 controls.
- [EC2 Image Builder Pipeline](https://docs.aws.amazon.com/imagebuilder/latest/userguide/what-is-image-builder.html)
  - Uses ECS-optimized AMI as the base image, applies Ansible playbook to implement CIS Docker controls and publishes the hardened AMI.
  - After publishing the new image, uses inspector to scan the image for vulnerabilities.
  - Runs [Docker Bench for Security](https://github.com/docker/docker-bench-security) to verify the applied controls and publishes the report to test results S3 Bucket.
- [Amazon Simple Notification Service (SNS)](https://docs.aws.amazon.com/sns/latest/dg/welcome.html): Once the new AMI is published, EC2 Image builder publishes a message to Image Pipeline SNS topic including new AMI image ID.
- [AWS Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html):
  - Step function trigger Lambda function, takes the AMI image ID published to Image Pipeline SNS topic and triggers an [AWS Step Function](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html).
  - The Step function first gathers ECS clusters' information and their associated autoscaling group, then runs an instance refresh on all the eligible autoscaling groups with the new image ID.
  - After autoscaling group instance refresh is complete. A message is published to Instance Refresh SNS topic.

Optionally you can deploy Image Update reminder solution which checks for new ECS-optimized base images and publishes a message to Image update reminder topic, so you can run the pipeline and update your autoscaling groups.

## Excluded controls

### Section 1

- 1.1.1 Ensure a separate partition for containers has been created
Done by EC2 Image Pipeline.
- 1.1.2 Ensure only trusted users are allowed to control Docker daemon
Should be covered by compensating controls. Users should not have access to ECS hosts.
- 1.2.1 Ensure the container host has been Hardened
Consider compensating control. This is a general requirement.1.2.2 Ensure that the version of Docker is up to date
Managed by AWS (this role is applied to ECS optimized images where docker is pre-installed)

### Section 2

- 2.1 Run the Docker daemon as a non-root user, if possible
This control needs to be implemented per business requierements basis.
- 2.4 Ensure Docker is allowed to make changes to iptables
Managed by ECS
- 2.5 Ensure insecure registries are not used
Use other compensating controls.
- 2.6 Ensure aufs storage driver is not used
ESC optimized images use overlay2 for storage drive.
- 2.7 Ensure TLS authentication for Docker daemon is configured
Docker daemon should not be exposed via a network socket, use compensating controls.
- 2.9 Enable user namespace support
Varies based on business requirements, consult docker documentations.
- 2.10 Ensure the default cgroup usage has been confirmed
ECS optimized image does not have --cgroup-parent set by default.
- 2.12 Ensure that authorization for Docker client commands is enabled
Varies based on business requirements, consult docker documentations.
- 2.17 Ensure that a daemon-wide custom seccomp profile is applied if appropriate (Level 2 - Docker - Linux)
Not supported by ECS. Covered in OPA policies.
- 2.18 Ensure that experimental features are not implemented in production
Not supported by ECS.

### Section 3

- 3.7 Ensure that registry certificate file ownership is set to root:root
- 3.8 Ensure that registry certificate file permissions are set to 444 or more restrictively
Fresh image does not have any registries.
- Docker daemon remote access
  - 3.9 Ensure that TLS CA certificate file ownership is set to root:root
  - 3.10 Ensure that TLS CA certificate file permissions are set to 444 or more restrictively
  - 3.11 Ensure that Docker server certificate file ownership is set to root:root
  - 3.12 Ensure that the Docker server certificate file permissions are set to 444 or more restrictively
  - 3.13 Ensure that the Docker server certificate key file ownership is set to root:root
  - 3.14 Ensure that the Docker server certificate key file permissions are set to 400

Docker daemon should not be accessed remotely. Use other controls like SGs.
