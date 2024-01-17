---
title: 'EC2 Instance Security Hardening'
date: 2024-01-08T15:40:03-05:00
tags: ["AWS", "Cloud Security", "Instance Security"]
author: ["Rudra"]
cover:
  image: /posts/images/blog3.png
  hiddenInList: false
draft: false
---
# Enhancing EC2 Security By Instance Hardening: Restricting IAM Role Access to Specific Users

### Introduction to EC2, IAM Roles, and the Need for Restriction

In the dynamic world of Amazon Web Services (AWS), Elastic Compute Cloud (EC2) instances are pivotal for hosting critical applications. These applications often need to interact with AWS resources like S3 buckets. IAM roles offer a secure method to grant necessary permissions without relying on long-term credentials. The challenge, however, lies in ensuring that only the application or a specific user running it on the EC2 instance has the necessary IAM role privileges, while other users do not.

### Scenario: Application-Specific IAM Role Usage

Consider a scenario at XYZ Corp. Here, an application running on an EC2 instance requires access to an S3 bucket for data storage. Access to the S3 bucket is granted through an IAM role assigned to the EC2 instance. XYZ Corp's policy is specific: only the application, or to be more precise, the user under which the application runs, should have access to this IAM role's privileges. It is crucial that other users logging into the same instance do not inherit these privileges.

#### Scenario Diagram
![Scenario Diagram](/posts/images/Blog3-1.png )

---

### Securing the Instance Metadata Service (IMDS)

To address this, XYZ Corp employs a strategy to limit access to the Instance Metadata Service (IMDS), which is where the application retrieves the temporary security credentials associated with the IAM role. They use a Linux iptables command to restrict access to the IMDS. This command allows only the specific user, say app-user, under which the application runs, to retrieve these credentials:

Before we delve into the iptables command used for restricting access, it's crucial to understand the role of the 169.254.169.254 endpoint in an EC2 instance. This IP address is known as the instance metadata service (IMDS), a pivotal element in how EC2 instances manage and access their permissions.

When an EC2 instance is assigned an IAM role, the instance itself doesn't inherently 'know' the permissions it has been granted. Instead, it queries the IMDS at this specific IP address to retrieve the necessary information. The IMDS acts as a gateway for the instance to access its own metadata, which includes IAM role credentials. These credentials are what the instance uses to make authenticated API calls to various AWS services, adhering to the permissions defined in the IAM role.

Every time an application on the EC2 instance needs to access an AWS resource, it makes a call to the 169.254.169.254 endpoint to fetch these temporary security credentials. AWS ensures these credentials are dynamically rotated and updated, providing an effective way to manage access without the risks associated with long-term static credentials. By securing this endpoint, as we discussed with the iptables command, we effectively control who on the instance can access these IAM role credentials and, consequently, the corresponding AWS resources.

#### iptables command used for restricting access of IAM role for specific user
```bash
sudo iptables --append OUTPUT --proto tcp --destination 169.254.169.254 --match owner ! --uid-owner app-user --jump REJECT
```
What exactly this command does?

1. `sudo`: This is a command-line utility in Unix and Linux systems that allows users to run programs with the security privileges of another user, typically the superuser (root). In this context, it's used to execute the iptables command with root privileges, as modifying firewall rules requires administrative permissions.

2. `iptables`: This is the user-space utility program that allows a system administrator to configure the IP packet filter rules of the Linux kernel firewall. These rules are organized into a collection of tables, each serving a different function.

3. `--append OUTPUT`: This option tells iptables to append the rule to the end of the OUTPUT chain. The OUTPUT chain is responsible for controlling the behavior of outgoing traffic from the EC2 instance.

4. `--proto tcp`: This specifies that the rule applies to TCP protocol traffic. Since the instance metadata service is accessed over TCP, this targets the relevant traffic type.

5. `--destination 169.254.169.254`: This indicates the destination IP address for the rule, which is the instance metadata service address. It means the rule will only apply to traffic intended for 169.254.169.254.

6. `--match owner`: This is a match specification that allows the rule to inspect the owner of the process sending the packet. In this context, it's used to match the UID (User ID) of the process's owner.

7. `! --uid-owner app-user`: The exclamation mark '!' is used to negate the condition that follows. Therefore, this part of the command matches any traffic that is NOT owned by the user app-user. In other words, it applies the rule to all users except app-user.

8. `--jump REJECT`: This tells iptables what to do with packets that match the rule. In this case, REJECT means that any packet matching the criteria (TCP traffic going to 169.254.169.254 from any user other than app-user) will be rejected, effectively blocking those users from accessing the instance metadata service.

In summary, this command configures the EC2 instance's firewall to block all outgoing TCP traffic to the instance metadata service (169.254.169.254) from any user on the instance except for app-user. This is a targeted way to ensure that only specific processes running under app-user can retrieve IAM role credentials from the instance metadata, enhancing the security of the instance.

By implementing this rule, any other user logged into the EC2 instance is denied access to the IMDS, and consequently, cannot use the IAM role privileges.


#### Rough Diagram of a solution
![Scenario Diagram](/posts/images/Blog3-2.png )

---
### Best Practices for IAM Role Assignment

When creating IAM roles, it is crucial to follow the principle of least privilege. In our XYZ Corp example, the IAM role specifically grants access to the required S3 bucket and nothing more. This minimizes security risks and ensures that even if other users could access the IAM role, their capabilities would be limited.

### Practical Implementation and Considerations

In practice, setting up such a security measure requires careful planning. It involves not only configuring IAM roles and iptables rules but also ensuring that the application is run under the specific user (app-user in our case). System administrators and security teams need to collaborate to implement these measures effectively.

### Conclusion

Restricting IAM role privileges to a specific user on an EC2 instance enhances security by ensuring that only authorized entities have access to critical AWS resources. In our example with XYZ Corp, this approach ensures that even though multiple users can access the EC2 instance, only the application running under app-user can access the S3 bucket, aligning with the best security practices in AWS environments.