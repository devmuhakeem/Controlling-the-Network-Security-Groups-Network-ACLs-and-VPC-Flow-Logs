# Controlling the Network: Security Groups, Network ACLs, and VPC Flow Logs

A hands-on AWS Builder Lab where I locked down a WordPress application running across three network zones in a VPC, using security groups, network ACLs, and VPC Flow Logs, then checked the results in CloudWatch.

## Scenario
The lab framed the task as replicating on-premises network zones in AWS. The environment had a Network Load Balancer in two public subnets, an EC2 web server running Apache and WordPress in a private subnet, and a separate EC2 database server in another private subnet. The goal was to allow only the traffic the application needs and verify that everything else is rejected.

## What I did

### 1. Restricted traffic with security groups
- Confirmed the starting state: the Apache test page timed out because no rule allowed traffic from the load balancer to the web server
- Added an inbound rule to **Web Server SG** allowing HTTPS (TCP 443) from 0.0.0.0/0. The lab explains this is required because the Network Load Balancer passes traffic straight through to the web server
- Loaded the Apache test page over HTTPS (the web server uses a self-signed certificate, so the browser shows a warning)
- Opened the WordPress URL and got a Gateway Timeout, because the web server had no path to the database
- Added an outbound MYSQL/Aurora (TCP 3306) rule on **Web Server SG** with **DatabaseServerSG** as the destination
- Added a matching inbound MYSQL/Aurora (TCP 3306) rule on **Database Server SG** with **WebServerSg** as the source
- Reloaded WordPress, completed the install (site title "AWS Security Best Practices", admin user `wpadmin`), and logged in to the dashboard

### 2. Restricted traffic at the subnet level with network ACLs
Network ACLs are stateless, so every subnet needed rules in both directions, including the ephemeral port range (TCP 1024-65535) for return traffic.

**load-balancer-nacl** (associated with both public load balancer subnets)
- Inbound: rule 100 HTTPS (443) from 0.0.0.0/0, rule 101 TCP 1024-65535 from 10.10.0.0/16
- Outbound: rule 100 HTTPS (443) to the web server subnet, rule 101 TCP 1024-65535 to 0.0.0.0/0

**web-server-nacl** (private web server subnet, built as a knowledge check)
- Inbound: HTTPS (443) from 0.0.0.0/0, TCP 1024-65535 from 10.10.0.0/16
- Outbound: TCP 1024-65535 to 0.0.0.0/0, MYSQL/Aurora (3306) to the database subnet

**database-server-nacl** (private database subnet, built as a knowledge check)
- Inbound: MYSQL/Aurora (3306) from the web server subnet, TCP 1024-65535 from 10.10.0.0/16
- Outbound: TCP 1024-65535 to 10.10.0.0/16

Each new ACL starts with a rule that denies everything, so associating it with a subnet first broke access (the page timed out), and adding the rules restored it. I re-tested the Apache page and the WordPress site after each ACL.

### 3. Inspected the traffic with VPC Flow Logs
- Created a CloudWatch log group named `blog-access-logs` with a 1-day retention setting
- Created a flow log named `vpc-flow-logs-for-blog` on Lab VPC with filter set to All, a 1-minute maximum aggregation interval, CloudWatch Logs as the destination, and `VpcFlowLogsRole` as the service role
- Used a custom log format: account-id, interface-id, srcaddr, srcport, dstaddr, dstport, subnet-id, flow-direction, action
- Generated traffic by refreshing the WordPress site three times, then tried the site over HTTP
- The HTTP request timed out, as expected, since only HTTPS is allowed to the web server
- Searched all log streams in `blog-access-logs` and compared accepted traffic (HTTPS on port 443) with rejected traffic (HTTP on port 80)

## Key takeaways
- Security groups are stateful and attach to resources; network ACLs are stateless and attach to subnets. A connection has to be allowed by both, and a network ACL needs rules for the return traffic
- Referencing a security group as a rule's source or destination expresses "anything in the web server group" without hardcoding IP addresses
- Testing for the failure first, then adding a rule, then testing for success shows exactly which change made the difference
- Flow Logs turn "I didn't open that port" into a log record showing the connection was rejected. They record traffic metadata, not packet contents
- Some settings here are lab shortcuts, such as 0.0.0.0/0 on port 443, the 1-minute aggregation interval, and the 1-day log retention. In production they should be reviewed against the real architecture and policy

## Tools
Amazon VPC, Security Groups, Network ACLs, VPC Flow Logs, Amazon CloudWatch Logs, Amazon EC2, Network Load Balancer

---
*Completed as an AWS Builder Lab (Controlling the Network).*
