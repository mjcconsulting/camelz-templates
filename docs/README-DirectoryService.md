# Notes on how to integrate DirectoryService without using it direct in a DHCPOPtions set


## Notes from Ticket on this topic

-  Thu Dec 12 2019 21:04:55 GMT-0800 (PST)
    ```
    Hi Michael,

    Thank you for taking my call yesterday and explaining your configuration in-detail.
    I was talking to engineers in different teams to identify a solution for your current and possible feature Cloudformation based configurations.

    A)Your current configuration :
    -All AWS Resources are deployed using Cloudformation template.
    -DHCP options set is configured with Managed directory services DNS.
    -Stand-alone resources like EFS file systems will be deployed in the same VPC and will be accessed from domain joined resources.

    Issue :
    Domain joined EC2 instance can't resolve standalone resource like EFS file system in the same VPC.
    This is because the default forwarder (169.254.169.253) configured in Managed directory DNS service is not pointing to your VPC.

    Workaround using your existing Cloudformation method :
    By adding these conditional forwarders to Managed Directory services DNS, you can resolve both private and public resources.
    Add conditional forwarders for these domains :
    Conditional Forwarders                                   IP address
    Amazonaws.com                                              VPC IPv4 network range plus two   <--this will resolve resource like EFS and RDS.
    Region.compute.internal                                VPC IPv4 network range plus two   <--this will resolve EC2 private names.
    b.us-west-2.camelz.io                           VPC IPv4 network range plus two   <--Private hosted zone
    camelz.com                                                       VPC IPv4 network range plus two  <--Public hosted zone
    You can add required route53 private and public zones to conditional forwarders.
    Why ?
    You have to create these conditional forwarders because the default forwarder configuration in Managed directory services is not linked to your VPC.
    Other options :
    By running the following powershell commands from a domain joined EC2 instance you can update the forwarder with VPC-IPv4-network-range-plus-two.

    # This will add DNS management tools
    Install-WindowsFeature -Name RSAT-DNS-Server
     #This will update default forwarders configured on Managed directory services DNS.
    Set-DnsServerForwarder -computername DNS-server-ip-1 -IPAddress "172.31.0.2" -PassThru
    Set-DnsServerForwarder -computername DNS-server-ip-1 -IPAddress "172.31.0.2" -PassThru
    Note - I have tested these commands with my directory services DNS and able to update forwarders without any issues.
    I saw your EC2 domain join information in Cloudformation, is it possible for you to add these PowerShell commands with exiting configuration ?

    B)Future plan : to use Amazon provided DNS in DHCP options set and use Route53 resolver for private domains.
    This will work without any issues for private domains and managed directory services domains however as I mentioned before, there will be an on-going cost associated with resolver service.

    Please see my comments for your questions.
    1. Use CloudFormation to create all resources.
    You are already using CloudFormation and this is the best way to deploy resources in AWS.

    2. For instances created in a public subnet, create DNS names that map to the EIP associated with the instance, so they are reachable by anyone on the internet
    I can't see any issues with this configuration.

    3. For instances created in public or private subnets, create DNS names that map to the private IP, so that other instances can reach the new instance by some predictable DNS name, ideally
    on our own defined private domain.
    This is a common use case to access resources with a friendly name.

    4. Support designs that do not use Windows without a need for DirectoryService.
    I have provided our public document[1] for VPC and how DNS resolution will work in a new configuration, we don’t have a specific design document  because use case will change for each customer.
    Also, I have verified your DNS configuration with both Networking/Directory services teams and the configuration is valid for your use case.

    5. Support designs that do use Windows, and need a Domain, which use DirectoryService.
    Please check the link[2] provided, the document has the best practices for Managed directory services, based on the information, you are already using the correct configuration (dhcp options set pointing to directory services DNS).
    Only change required there is the default forwarder configured on DNS servers must be updated with  VPC IPv4 network range plus two.

    As we discussed before, I am happy to call back to go through the items in this email so let me know when we can do this.
    Have a great weekend and I am looking forward to hearing from you soon.

    References:
    [1] Using DNS with Your VPC : https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html
    [2] Best Practices for AWS Managed Microsoft AD : https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_best_practices.html

    Best regards,

    Mano M.
    Amazon Web Services
    ```

-   Fri Dec 06 2019 22:20:08 GMT-0800 (PST)
    ```
    Just in case you want to think about this a bit based on the domain names we're currently using, let me share those too before I stop and go eat. Pretty late here.

    I'll use our Build Account for this client, it's different than the Account where this case exists, not sure if that matters, but we'd need to switch to that account to work through a demo based on the names below.

    These are the domains involved
    1. Public: b.us-west-2.camelz.com - This exists as a Route53 Public Hosted Zone. It's created in it's own CloudFormation Stack
    2. Private: b.us-west-2.camelz.com - This exists as a Route53 Private Hosted Zone. It's associated with only our Build VPC. The Build VPC, the hosted zone with this domain name, and a DHCP options set which specified this domain, and AmazonProvidedDNS are all created in our VPC Stack.
    3. Directory: b.ad.camelz.com (we've also used b.ad.us-west-2.camelz.com). My assumption was that this domain had to be different than the private domain, so that we could create the conditional forwarder, so we're (at a minimum) adding "ad." as a subdomain to the private domain. Originally our private domains did not have the region code, but that was added more recently in case we wanted to connect them via delegation records. I thought we might not want to add the region to the Directory domain as eventually we might want to try and share one directory across multiple VPCs. Right now, it's one Directory per VPC, which is easiest when building via CloudFormation templates, but I'd love to have a way to re-use the same Directory across multiple VPCs in different regions and even share across accounts. Maybe we can talk about that too.  So, we have a 3rd stack, which creates the DirectoryService, using this domain. Once the DirectoryService is up, I next create a ConditionalForwarder within the DirectoryService (via a CustomResource I wrote), which forwards b.us-west-2.camelz.com to VPC CIDR + 2 IP, then I replace the DHCP Options created in the VCP template with a new one that has the same private domain, but uses the 2 IP addresses of the DirectoryService. This design was based on an earlier pattern where we created a pair of AD Domain Controllers on instances. It seems to work well enough, but there's not an easy way to reuse one Directory across multiple VPCs to save on costs. I'd love to fix that.

    So, hopefully you have a better idea of what we're currently doing and thinking of improving. What I want, if it can work, is something close to what I understand your option B to be. We do not replace the DHCP options set which points to AmazonProvidedDNS. So, all instances which start up go direct to VPC CIDR + 2 IP to resolve, and they directly get any private Route53 names without touching DirectoryService. This leaves how Windows servers get to AD for Windows stuff, where I think the Resolver is involved. But still not clear how this works, and what domains are set in each location where I need to specify a domain. So, that's what I want to work through, ideally using these domains so it's clear.
    ```

-   Fri Dec 06 2019 22:00:49 GMT-0800 (PST)
    ```
    I think what you're testing looks excellent, but I'm having trouble following it from purely a text description - I think I'll need to either try to follow in your footsteps, or next week perhaps we can find a time and if you've left this running, you can maybe demo what you have created.

    If you may have created this via a CloudFormation stack or something in code that I could review and try to run, I think I'd understand that a lot more than your description in text form.

    I think I can probably follow WHAT you're doing - where I'm having trouble is really understanding your DESCRIPTION of what you're doing. This is at least partly because I haven't used the Resolver, and some of what you're describing about how directory service goes through a default resolver on 169.254.169.253 is something I completely do not follow, but perhaps that would become clear if we discussed it and I could ask questions, or you could show me where you're doing this and how.

    I have a pretty good amount of experience configuring DNS and setting up VPCs, DHCP Option Sets, Route53 Hosted Domains and RecordSets, all through CLI, SDK in JavaScript and mostly in CloudFormation. The main area where I have a hole in my knowledge that's preventing me from following you easily, is really understanding how Directory Service works (like your point about it's default resolver), and how it integrates with other AWS services, in particular how "join to an existing domain" might work in side the AWS EC2 console, or how it integrates with Route53.

    I can tell from what you've done that you understand this enough and I could greatly increase my knowledge of how these two things work together with a working session on this. So, maybe we can try and schedule something next week, Tuesday or later better. I'm in California, can usually meet between 8:30am PT to as late as midnight, depending on where you are.

    If you have any code you used to create these tests scenarios, that would really help, as a lot of time seeing what you're doing as code makes a lot more sense than having you try to describe it, absent watching you do what you describe via screen share.

    Thanks again for your efforts to help us here. You're definitely the right guy to help us figure this out.

    Another option is that I could build up one of our Accounts up to the point where we have created the public hosted zone, then our VPC template which creates the VPC, then a private hosted zone, then a DHCP Options set to connect the two, using Amazon provided DNS. This would be all we needed if DirectoryService or Windows was not involved. I'd really love it if all systems other than Windows using AD did not even know DirectoryService and AD even existed - that they would see AmazonProvidedDNS servers via DHCP, and could then do all DNS lookups without DS getting involved, unless there was some need to reach a Windows resource. So, perhaps with that starting point built out, you might be able to show me how to quickly add what's needed manually for option B, using our DNS domain names, where it would really sink in best. LMK if that's possible too.
    ```

-   Fri Dec 06 2019 20:57:48 GMT-0800 (PST)
    ```
    Hi Michael,

    Thanks for allowing me to test your scenario in my lab.
    I had tested two different methods and provided the steps below.

    A.VPC DHCP-options-set-1[1] configured with AWS Managed directory DNS server addresses.
    B.VPC DHCP-options-set-2 configured with AmazonProvidedDNS and Route53 resolver.

    Method A :
    1.Lab domain - it.syd.com - AWS managed directory service.
    2.Created and applied DHCP-options-set-1 to use AWS Managed directory DNS servers.
    3.In AWS managed directory DNS servers, removed the default forwarder (169.254.169.253) and added VPC resolver( for my subnet 172.31.0.2).
    4.Deployed an EC2 instance and created a Windows server 2016 based domain controller - sydney.com - private domain.
    5.Added conditional forwarder in managed directory DNS for the EC2 based domain created above.
    6.Created an EFS file systems for testing - fs-7e6b5847.efs.ap-southeast-2.amazonaws.com.
    7.Deployed an EC2 instance and domain joint it to it.syd.com domain.
    8.From domain joined EC2 instance, I was able to resolve :
    -it.syd.com
    -fs-7e6b5847.efs.ap-southeast-2.amazonaws.com
    -Sydney.com
    Also using ping, I was able to confirm the ip addresses of above services.

    DNS lookup :
    Domain joined EC2 instance was able to resolve fs-7e6b5847.efs.ap-southeast-2.amazonaws.com using the forwarder(step-3) configured in Managed directory DNS.
    Domain joined EC2 instance was able to resolve Sydney.com using the conditional forwarder(step-5) created in Managed directory DNS.
    Optional : As an additional step, I had removed the conditional forwarders for sydney.com in Managed directory services, created route53 resolver for sydney.com for my VPC and able to resolve/ping the domain.

    Method B :
    1.Existing lab domain - it.syd.com - AWS managed directory service ( same as Method A).
    2.Created and applied DHCP-options-set-2 to use AmazonProvidedDNS on VPC.
     AmazonProvidedDNS, DNS server- 172.31.0.2(VPC).
    3.In AWS managed directory DNS servers, removed all other forwarders and added VPC resolver- 172.31.0.2.
    Also removed the conditional forwarders created for sydney.com private domain.
    4.Used the same private domain controller - sydney.com for testing.
    5.Used the existing EFS file systems for testing - fs-7e6b5847.efs.ap-southeast-2.amazonaws.com.
    6.Used the existing domain (it.syd.com) joined instance.
    7.Using route53 resolver[2] [3] outbound endpoints, created rules for both it.syd.com and sydey.com.
    8.From the same domain joined EC2 instance, I was able to resolve all the domains ( same as method A) :
    -it.syd.com
    -fs-7e6b5847.efs.ap-southeast-2.amazonaws.com
    -Sydney.com

    DNS lookup :
    Domain joined EC2 instance was able to resolve fs-7e6b5847.efs.ap-southeast-2.amazonaws.com using the VPC resolver- 172.31.0.2.
    Domain joined EC2 instance was able to resolve Sydney.com using the rule and outbound endpoint configured in Route53 resolver.
    Please note - Route53 resolver is a paid service and I have provided the link[4] , which has the pricing details.

    By using method A ( without route53 resolver option), you can resolve both private and public domains so please let me know if you get any errors.
    I have validated method-A with Directory services team however changing the forwarder from default (169.254.169.253) to VPC will resolve resources with private DNS and public domains.

    From the previous case notes,  I can see the test with dig command :
    Example: I created a conditional forwarder in DirectoryService, so domain us-west-2.amazonaws.com uses  DNS 10.160.0.2, then on an instance in AZ A:
    $ dig +short @10.160.0.2 fs-df001474.efs.us-west-2.amazonaws.com
    10.160.2.142 <== This is the correct value, the IP of the mount target in this same subnet
    $ dig +short fs-df001474.efs.us-west-2.amazonaws.com
    10.160.6.136 <== This is the INCORRECT value, when I go through the DirectoryService DNS servers, for the mount target in the AZ B subnet

    You had mentioned 10.160.6.136 is not correct, is that part of the EFS or some other resource ?
    Also, like to know if the VPC subnets are the same for 10.160.0.2 and fs-df001474.efs.us-west-2.amazonaws.com or different ?
    I would to discuss this issue over a phone call so if possible, please share your contact details and the local time zone.

    Let me know if you require any further assistance whatsoever.
    References :
    [1] DHCP options set : https://docs.aws.amazon.com/vpc/latest/userguide/VPC_DHCP_Options.html
    [2] Getting Started with Route 53 Resolver : https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-getting-started.html
    [3] Amazon Route 53 Resolver for Hybrid Clouds : https://aws.amazon.com/blogs/aws/new-amazon-route-53-resolver-for-hybrid-clouds/
    [4] Route 53 pricing : https://aws.amazon.com/route53/pricing/

    Best regards,

    Mano M.
    Amazon Web Services
    ```

-   Thu Dec 05 2019 21:31:34 GMT-0800 (PST)
    ```
    Hi Michael,

    My name is Mano and I will be responding to your query today.
    I can see the case was re-opened and you like to get some information based on the original request and reply.

    AWS Managed directory services will be deployed with 169.254.169.253 as the default forwarder so domain joined instances using the these DNS servers can resolve public domains.
    To resolve private hosted domains for a domain joined EC2 instance, you can use the following methods:
    1.Configure a conditional forwarder and point it to the DNS private servers.
    2.Remove the 169.254.169.253 forwarder in Managed directory services and use VPC (example - 172.x.x.2) as the forwarder and use Route53 resolvers.
    3.Use DHCP option set and point it to VPC (example - 172.x.x.2) and utilize route 53 resolvers.
    Please note- manually configured DNS servers in windows OS will overwrite dhcp-options-set DNS servers.

    For non-domain joined EC2 instances, dhcp-options-set and route 53 will be the ideal solution.
    I am testing these options in my lab for you and able to provide more detailed information shortly so if possible please allow a day or so.
    Also, If possible, please send me the details/screenshots of Route 53 resolver configured in this account.

    Best regards,

    Mano M.
    Amazon Web Services
    ```

-   Tue Dec 03 2019 19:01:25 GMT-0800 (PST)
    ```
    I want to re-open this ticket to further discuss one of the suggestions which were made to resolve this issue. While I could not explore reversing how we were doing DNS at the time (explained below, I think), I was intrigued by the suggestion as I think it's a better design - IF - it can be made to work. I was able to use another method to solve the specific issue raised here, a variant of using the IP address of the EFS endpoints, and combined with a lack of time to explore the second option, I had to park that idea for a while. Now, I want to explore it more. Still not sure I can make a change like this on the client using the current code, at least not quickly, but I do want to really understand what this alternative means, so I can try this out for potential future use, and confirm it works, and there are no unexpected side effects.

    What I'm looking for: THE "best practice" architecture for how to combine use of Route53 private hosted zones for CloudFormation-based registration of host and service recordsets, used for connection between tiers based on such names - AND - use of DirectoryService for Windows instances within that same VPC, which are then joined to this domain, and which have dynamic DNS records created for them on the DirectoryService domain.

    My assumption, which I still am not convinced is wrong, is that a pair of AD Domain Controllers running DNS, LDAP, Kerberos, etc. MUST be the DNS servers returned from DHCP when a Windows Instance starts if that Instance is to join a domain. This is why I was switching the VPC to a DHCP Options Set which pointed to the 2 IP addresses of the DirectoryService Instances, and then adding a ConditionalForwarder which pointed back to the private hosted zone associated with the VPC to resolve all the private DNS names we're creating within CloudFormation to glue everything together.

    While I can understand the concept of using the Resolver to implement "This setup is the inverse of the current setup. We're sending only what is necessary to the directory service DNS", it would seem this could handle DNS lookup requests, but might not work for all the other communications a Domain Member needs to do with a Domain Controller, including DHCP, Kerberos, service discovery, and so on. This is really stretching the limits of my AD networking knowledge.

    So, what I really want to know is: What does the DIrectoryService team recommend as their "best practice" architecture to handle both Route53 private DNS within a VPC for non-Windows, and also Windows instances in the same VPC which are joined to it? I can't be the first one to ask this question. I just can't recall seeing a fully documented official "right answer" for this. Hoping you can point me to one, which has the "blessing" of the DirectoryService team.

    I'd like an email response, ideally with links and an explanation as to how it addresses my concerns as needed. I'll want to study that, and then attempt to implement it in a spare environment if it's clear enough, or follow up if it's not clear enough.
    ```

-   Fri Oct 18 2019 19:38:32 GMT-0700 (PDT)
    ```
    Hi Michael,

    Thank you for your time on call. I have answered your questions below:

    1. What is the MOST correct way of setting up a VPC, DHCP Options Set, Route53 public and private hosted zones, and Directory Service so we can:

    a) Using CloudFormation, create Instances and Services (like EFS), allowing AWS to pick the private IP Address, and then in the same Template, create RecordSets in the VPC's private HostedZone, so we can have a stable endpoint to reference in other Templates which must integrate.

    - I'm not very familiar with Cloudformation but with the suggestion I provide below, I don't think you will need to make any significant changes to the template. The suggestion I provide below will require the use of just 1 DHCP options set. You can refer to document [1] for information about creating a Route 53 resolver using Cloudformation. If you face any issues with creating the new Cloudformation template, I suggest opening a case with the Cloudformation team as they will be able to assist better.

    b) Create Windows Servers which can join to the Domain implemented via DirectoryService.

    -I have discussed your situation with an engineer from the Windows team. The EFS DNS resolution behavior is expected. In this case it is recommended to use the IP address of the EFS endpoint when mounting the filesystem. However, there is another solution that you can try since you do not want to use a static mapping.  

    Solution for EFS DNS resolution:
    Instead of setting 10.160.7.253,10.160.3.244 as the DNS servers in the DHCP options set, set the DNS as the AmazonProvidedDNS. This will take care of the EFS issue and will also enable the instances to resolve records in the private hosted zone.   

    Solution for Windows servers to be able to reach the directory service:

    Create an outbound Route 53 resolver for the VPC[2]. Create a rule that forwards all traffic that needs to be directed to the directory service to be sent to 10.160.7.253,10.160.3.244. When specifying the domain name, ensure that you use the complete domain name as the R53 resolver considers a sub domain a match[3].  

    This setup is the inverse of the current setup. We're sending only what is necessary to the directory service DNS and the rest goes to .2 IP of the VPC. With this, you do not need the forward in the Windows server DNS to 10.160.0.2.

    2. What is the correct way to use EFS with DirectoryService?
    - The above solution addresses this.

    Resources:
    [1] AWS::Route53Resolver::ResolverEndpoint: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-route53resolver-resolverendpoint.html
    [2] Forwarding Outbound DNS Queries to Your Network: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-forwarding-outbound-queries.html
    [3] https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html#resolver-overview-forward-vpc-to-network-domain-name-matches
    [4] Resolving DNS Queries Between VPCs and Your Network: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html

    Best regards,

    Nigel F.
    Amazon Web Services
    ```
