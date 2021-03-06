---
title: "Static Websites in the Age of Choice"
date: "2020-04-05T00:33:33Z"
tags: ["cloud", "colophon"]
---

Hosting a website today comes with a lot of choices and it can be a daunting task to analyse all the options available from many large and small corporations.
I'd like to take you on a whirlwind tour of the analysis I did when choosing where to host this site, starting from the requirements I need and looking at how each provider meets them.

## Requirements
I chose to create a static website because it reduces the amount of software necessary to run the website --- no database or WordPress installations means less to maintain.
My free time seems to easily dwindle, so being able to use that time to write instead of maintain a server is a huge benefit.
A static website should also theoretically cost very little to run, since almost all the content can be given larger cache life-times.

I want all of my content to be served over HTTPS, so HTTP access to the website will need to be redirected to HTTPS immediately.
The content here should also avoid pulling in unnecessary Javascript libraries because this website is focused on my written work --- web workers, pre-fetching potential future pages, and like features are irrelevant.

Finally, I would like to make sure that the website is protected against simple flood attacks, like repeated HTTP requests for the largest hosted resource, to avoid a sky-rocketing bandwidth utilization bill.
Put this all together and my list of requirements for hosting looks something like the following:

- Minimal server maintenance
- Maximize security of hosting
- HTTPS-only content delivery
- HTTP redirect to HTTPS
- Flood attack protection
- Minimal cost

Thankfully, excluding Javascript is an easy requirement to hit --- just avoid using things like Google Analytics.

## Flood Protection
Most hosting providers do little to protect you against application layer attacks like HTTP floods; providers tend to focus their energy on mitigating network level attacks like SYN floods.
This makes sense because the hosting providers want to keep their network available for the many customers using it for various products and services --- some customers may not even use HTTP for their service.

CloudFlare does provide HTTP flood absorption for their free tier and has fairly good geographic distribution as a CDN.
Putting my website closer to every potential reader across the world and having protection against drive-by HTTP floods for free sounds fantastic.
Of course there's a catch --- I will need to share an SSL certificate with a few other free tier accounts, or pay CloudFlare money for a dedicated certificate.
At the moment a dedicated certificate costs US$5/month, while that's fairly cheap I would like to keep my hosting costs as low as possible.
I am fine with sharing a certificate amongst a handful of free tier accounts as a trade-off for usage of the CloudFlare CDN.

This half solves the HTTPS requirement as the connection between the reader's browser and CloudFlare will be secure, but the connection between CloudFlare and my website origin can be accessed via HTTP or HTTPS.
CloudFlare calls this "Flexible" mode and this HTTP access between CloudFlare and my website origin leaves it open to man-in-the-middle attacks.
CloudFlare will save me bandwidth costs, but I still need to have a hosting solution that supports HTTPS.

As an added bonus I can solve the HTTP redirect requirement via CloudFlare by toggling their HTTP to HTTPS redirect functionality on --- a one-click solution to a requirement.
Solving two of my requirements for one smaller trade-off is a pretty good deal, especially when it's a free service from a reputable corporation.

## AWS
There's lots to like about AWS from the perspective of disaster recovery and service choices.
CloudFormation is a wonderful way to represent provisioning services within AWS and S3 is very easy to work with for static websites.
One of the larger problems with AWS is that it can be very expensive and most of that expense can be hidden behind bandwidth costs.
A more immediate problem is that an S3 bucket configured to host a static website does not support HTTPS access.

The solution to this problem is another AWS product called CloudFront, which is a CDN offering like CloudFlare.
You create a CloudFront distribution, attach an SSL certificate for HTTPS, assign the static website S3 bucket as its origin, and configure CloudFlare to use that CloudFront distribution as your website origin.
Your CloudFront costs per month are calculated by volume of requests and by volume of data transfer --- that can make cost estimation rather tricky.

With a CloudFront distribution you have a unique, randomly generated sub-domain, but there's still the possibility of receiving drive-by HTTP floods at the distribution level, which circumvents CloudFlare.
To solve this, we can use another AWS service called WAF --- Web Application Firewall --- to block all incoming requests to the CloudFront distribution except for those originating from CloudFlare.
AWS WAF will cost us another US$6/month plus additional costs from request volume.

You should see a pattern here --- AWS is built on cross-selling you additional products to solve problems caused by missing functionality in other products.
This is an example of a cross-sell from S3 static hosting, to CloudFront for HTTPS support, to AWS WAF to protect your CloudFront distribution.
There are many more of these cross-selling chains in AWS because it's their strategy for locking you into their services.
If you use enough of them, you won't want to research alternatives because it seems like too large of a headache.

## GCP
Google Cloud Platform I find to be better than AWS --- services feel more polished and they cross-sell other products far less aggressively.
They have good support for infrastructure-as-code tools like Terraform and their own tool is based on Python with a decent command-line interface.
The `gsutil` program is my favourite out of all the cloud providers for working with object storage because it allows parallel uploads and exposes useful flags across copy and sync operations.

For hosting a static website Cloud Storage is the equivalent of S3, but it operates in a fundamentally different way.
Google Cloud Storage already acts as a CDN and supports HTTPS out of the box for everything you store --- a large benefit over S3.
However, like S3, Cloud Storage suffers from requiring the bucket to be the domain of your website which makes it easy to access the bucket directly and avoid CloudFlare to issue HTTP floods.

Google provides a free solution for this problem which they call "VPC Perimeter" and it's a very useful piece of security functionality for every business.
When you create a VPC perimeter you explicitly choose what has access to services inside the perimeter, so this can protect our Cloud Storage bucket from unauthorized access.

Google Cloud Storage also provides an always-free tier consisting of 1GB egress bandwidth from North America to a few other regions and 5GB-months of storage --- bandwidth over 1GB will cost around US$0.12/GB.
The always-free tier essentially makes your website entirely or nearly free to host and is a terrific option for hosting a static website behind CloudFlare.

## Azure
Microsoft Azure also has decent infrastructure-as-code offerings, although they are not as polished as AWS or GCP, and are missing some important configuration options for certain resources.
They're still building out the command-line tooling, so some of it is a bit oddly structured and is missing functionality; as an example, there's no cache control functionality available on the storage sync command.

The features Azure does offer for hosting a static website are quite good --- their blob storage service supports HTTPS out-of-the-box and it does not suffer from the annoyance of requiring the bucket to be the domain.
Azure calls buckets "storage containers", they exist in "storage accounts", and to host a static website you create a storage container called `$web`.
That container is pointed to by an Azure sub-domain which you can then use to configure a custom domain.

Storage accounts also have built in support for firewall rules, so each storage account can have uniquely customized external access.
One minor downside is that the firewall rules only support IPv4 addresses and CIDR representations, so if you want to use IPv6 you are out of luck.

## Remaining Problems
None of these providers solve every problem I have without cross-selling me their CDN products.
For example, there is no way for me to set Content-Security-Policy headers on any of these static website offerings, but that is less critical since I am only serving basic web pages.

I've also actively avoided discussing corporations like Linode or Digital Ocean --- my browser-based security concerns would disappear by setting up my own VM on those services through running nginx and Let's Encrypt, but then I would need to manage a server and pay US$5/month.
It would be reasonable to pay that cost for the 1TB/month egress bandwidth that Linode or Digital Ocean provide on their cheapest VMs because the large cloud providers have expensive bandwidth pricing, but I am nowhere near needing that kind of scale.

## Hosting Solved
I've worked with AWS and GCP, so I chose Azure to learn a bit about that cloud platform; it meets enough of my security criteria that I can feel confident in it.
My current solution solves my requirements, costs me $0/month, and requires no active maintenance:

- static website hosted via Azure blob storage
- static website is accessible only from CloudFlare IPv4 addresses
- all connections between my website, CloudFlare, and the reader are secured via HTTPS
- all insecure HTTP requests are immediately redirected to HTTPS
- protection from flood attacks thanks to CloudFlare

That's a pretty good deal --- I think it would be hard to find a better one offered by any corporation.
One day I may get tired of the Azure command-line making me do [janky things](https://github.com/tinychameleon/tinychameleon.com/blob/2e37c8a3aa6ab19af5f29a3f091a19ef2916407b/Makefile#L71) and on that day I will probably migrate to GCP.
