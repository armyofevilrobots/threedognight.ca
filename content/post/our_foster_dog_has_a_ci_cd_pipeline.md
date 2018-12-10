---
title: Our Foster Dog Has a CI/CD Pipeline
date: 2018-12-08T12:00:00Z
author: Derek
---

Disclaimer: This is nerdy AF. If you're not into CI/CD pipelines, build automation, and serverless computing,
you should probably turn around and run screaming. Abandon all hope, etc. etc.

---

It all started so innocently...
<!--more-->
> "We should blog about fostering Fender... It'll help find a forever home for him, and it'd be cute..."

## The Concept

Wordpress is a PITA to theme, and expensive to host. It's made with PHP, which is kind of like 
heating your house with radioactive waste. I mean, it works, but it's likely to give you cancer
and organ failure (and perhaps have some russians you don't know very well moving in and taking over).

So, I decided to do things "right", by building a modern Continuous Integration and Continuous Deployment
pipeline that would turn plain old [markdown](https://github.github.com/gfm/) into well formed HTML5.

All the code you need to replicate this thing is available here: https://github.com/armyofevilrobots/threedognight.ca

## Toolchain:
* For the HTML generator, I decided on [hugo](https://gohugo.io/), mostly because I am terrified of [javascript](https://thehackernews.com/2018/11/nodejs-event-stream-module.html).
* For code hosting, obviously [github](https://github.com/).
* I decided to try out [Travis CI](https://travis-ci.com/) to run the build-bot, and that's worked out AWESOME so far.
* For hosting, I wanted to use Amazon S3, but the problem there is that it's kinda limited in regards to SSL and caching, so...
* I added [AWS Cloudfront](https://aws.amazon.com/cloudfront/) to the mix. This gives me SSL termination (for free via AWS certs), multi-geo shortest path, caching, and a bunch of other neat features that I *totally* don't need.
* SSL certs were managed via [AWS ACM](https://aws.amazon.com/certificate-manager/).
* And finally DNS is provided via [AWS Route53](https://aws.amazon.com/route53/). You can do it with an external provider, but that's a huge PITA because you CAN'T HAVE A ROOT @ CNAME (at least not legally).

## Process
My process for the Cloudfront setup roughly followed 
[this guide](https://simpleit.rocks/golang/hugo/deploying-a-hugo-website-to-aws-the-right-way/), 
with some divergences due to drift over time, and some customization I wanted to do.

### TL;DR To Setup Cloudfront:

1. Set up S3 buckets that are web hosting enabled, and hold your website, one each for your root domain 
   (ie: [threedognight.ca](https://threedognight.ca/)), and www.$YOURDOMAIN, 
   ie: [www.threedognight.ca](https://www.threedognight.ca/). 
   Only the first holds your website, the second is just a redirect domain. 
   Empty domains are basically free on S3, so no reason not to.
2. Set up Cloudfront to host BOTH of these domains. You'll use the same config, backing on S3, and 
   pointing at your SSL certificates, which you set up:
3. AFTER you set up DNS in Route53; you'll need to create $YOURDOMAIN and www.$YOURDOMAIN pointing at the 
   ALIAS records that are automatically created in step #2 (be sure to choose the cloudfront ones, NOT 
   the S3 ones).
4. And THEN you can create the certificates in ACM by requesting them for your domain. You'll have to go back
   into Route53 to create some CNAMES, but they'll give you detailed instructions.
5. Go back and set up the certs in Cloudfront in each of your domains. You have to do this now because 
   they didn't exist when you were doing step #2.
6. There is no step 6. You've got Cloudfront configured. YAY!

Yeah, I should really setup ansible roles/playbooks for this, but it was a one-day project.

Now we need to setup travis.

### Configuring Travis CI

This is mostly cribbed from [parsiya.net's travis -> S3 docs](https://parsiya.net/blog/2018-04-24-deploying-my-knowledge-base-at-parsiya.io-to-s3-with-travis-ci/)...

1. Go to [Travis CI](https://travis-ci.com/) and login using your github creds.
2. Find your repo from the menu on the left. And turn on CI/CD for it (that'll automagically generate 
   a webhook and get everything up and running).
3. Create the following environment variables:
  * AWS_ACCESS_KEY_ID
  * AWS_SECRET_ACCESS_KEY
  * S3_BUCKET
  * S3_REGION
  * AWS_CLOUDFRONT_DISTRIBUTION_ID
4. Add the .travis.yml below, customized for your repo of course ;) if you've 
   branched (because you don't wanna redeploy _our_ blog).

```yaml
# safelist - only build on pushes to these branches
language: go
branches:
  only:
  - master
  - travis
cache: pip
go:
- 1.11
# Install the AWS CLI (SLooooooW)
before_install:
  - pip install --user awscli
  - export PATH=$PATH:$HOME/.local/bin
install:
# change this version as it goes up
# get and install Hugo
- wget https://github.com/gohugoio/hugo/releases/download/v0.52/hugo_0.52_Linux-64bit.deb
- sudo dpkg -i hugo*.deb
- git clone https://github.com/armyofevilrobots/threedognight.ca
- cd threedognight.ca
script:
# build the website with Hugo, output will be in public directory
- hugo
# deploy public directory to the bucket
deploy:
  provider: s3
  access_key_id: $AWS_ACCESS_KEY_ID
  secret_access_key: $AWS_SECRET_ACCESS_KEY
  bucket: $S3_BUCKET
  region: $S3_REGION
  local-dir: public
  skip_cleanup: true
  acl: public_read
  on:
    # make it work on branch other than master
    # change this to master or any other branch if needed
    branch: master
after_deploy:
- aws cloudfront create-invalidation --paths '/*' --distribution-id $AWS_CLOUDFRONT_DISTRIBUTION_ID
```

## Wrapping up

You're done. 
```
git commit -a -m "My Changes" && git push
```

should update your repo, kick off the build in travis, and push the new HTML up to S3. It'll then invalidate the
cloudfront cache, and you should get fresh pages, just like mom used to make (in dreamweaver, gross).
