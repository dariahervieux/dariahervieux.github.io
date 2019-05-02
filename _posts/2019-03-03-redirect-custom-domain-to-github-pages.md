---
title: "Custom domain configuration for GitHub pages with OVH DNC provider"
categories:
  - Configuration
tags:
  - github-pages
  - dns
  - ovh
---


> **TL;DR** Quick tutorial for configuring redirection of your custom domain to your GitHub pages static web-site.


Finally I've chosen my own domain name: **da-sha1.me**. Lovely, isn't it? With a little homage to the theme I cherish the most - security. [SHA1](https://en.wikipedia.org/wiki/SHA-1) is a hashing function, not recommended to be used nowadays however.. Vintage! I like it :smile:.

And now straight to the point!

## GitHub configuration recommendations

GitHub [help](https://help.github.com/en/articles/quick-start-setting-up-a-custom-domain) provides a really nice and details explanation about configuring a custom domain, which boils down to:

1. Buy your domain name.
2. Add your custom domain to your GitHub Pages site (event if it won't work).
3. Add a DNS record for your custom domain with your DNS provider. 
4. Remove and re-add your custom domain to GitHub pages.

Ok. Adding a custom domain to GitHub Pages is very easy. Go to tour GitHub project Settings, GitHub Pages section, and fill in Custom domain input:

{% include figure image_path="assets/images/GitHub-Pages-custom-domain-name.png" alt="GitHub Pages Custom domain name configuration" caption="GitHub Pages Custom domain name configuration" %}

Now a DNS record. I have an Apex domain, i.e a domain without a sub-domain part.
In this case GitHub Pages help [suggests](https://help.github.com/en/articles/about-supported-custom-domains#apex-domains) to configure A , ALIAS, or ANAME record through your DNS provider.

> Just a little explanation for those who, like me :smile:, sees all these magical formulas for the first time:
>
> * A - standard record type, point the root (apex) domain or subdomain to an IP address
> * CNAME - standard Canonical Name record, point a subdomain to a Fully Qualified Domain Name. They cannot be used at the root level.
> * ALIAS or ANAME - non-standard record, the combination of A and CNAME, i.e point a root domain to a Fully Qualified Domain Name

The type of a record to use depends on your provider.

## DNS Zone configuration using OVH DNS provider

I've chosen [OVH](www.ovh.com) as my DNS provider.
To add a new DNS zone in OVH configuration record you should:

1. Go into your [Control Panel](https://www.ovh.com/auth/?action=gotomanager).
2. Choose the domain you'd like to configure in **Domains** menu.
3. Choose **DNS zone** tab.
4. Click **'Add an entry'** button

My DNS provider doesn't support ALIAS, ANAME records:

{% include figure image_path="assets/images/OVH-DNS-Zone.png" alt="OVH - adding new DNS Zone pop-up" caption="OVH - adding new DNS Zone pop-up" %}

I've added 4 A records for each IP address specified in GitHub Pages [help](https://help.github.com/en/articles/setting-up-an-apex-domain#configuring-a-records-with-your-dns-provider)

{% include figure image_path="assets/images/OVH-DNS-Zone-added.png" alt="OVH - added A records" caption="OVH - added A records" %}

To test if DNS configuration is correct, you can either use DNS lookup	utility [dig](https://www.freebsd.org/cgi/man.cgi?query=dig&apropos=0&sektion=0&manpath=FreeBSD+12.0-RELEASE+and+Ports&arch=default&format=html) for Linux distributions as recommended by GitHub Pages help like so:
```
> dig +noall +answer example.com
```
Or if you are on Windows, you can use [nslookup](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/nslookup) to find an A record like so:
```
> nslookup example.com
```
This should give you something like this:
{% include figure image_path="assets/images/nslookup-result.png" alt="nslookup result example" caption="nslookup result example" %}

It can take some time for DNS record to be taken into account. In my case it took several minutes.

If you are looking for a tutorial on using nslookup, I found a nice one on [clouds.net blog](https://www.cloudns.net/blog/10-most-used-nslookup-commands/).

## Configuring custom domain name - final step

Now  I have a DNS record configured correctly and I can proceed and remove and re-add my custom domain into GitHub Pages section in my project settings.
In addition I'd like to enforce HTTPS for my site. And fortunately for me, as stated in [help documentation](https://help.github.com/en/articles/securing-your-github-pages-site-with-https)
> All GitHub Pages sites, including sites that are correctly configured with a custom domain, support HTTPS and HTTPS enforcement.

So all I have to do is to check the box with "Enforce HTTPS" just below my custom domain configuration.
GitHup Pages help warns:
> It can take up to an hour for your GitHub Pages site to become available over HTTPS after you add and correctly configure your custom domain.

After being patient for 1 hour I finally have my personal site on my personal domain. Ta-daaaaa! :tada: :tada: :fireworks: