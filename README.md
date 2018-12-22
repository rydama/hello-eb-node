# Starter Repository for AWS Elastic Beanstalk with Node

A quick start for creating a web app on AWS with the following characteristics:

- Ideally suited for low-traffic side projects
- Quickly and painlessly deploy apps with https
- Dirt cheap to start. Easy to scale up as needed.
- Low cost ($5/month)
- HTTPS
- Auto-renew certificate from Let's Encrypt
- Provisions automatically from a brand new Elastic Beanstalk environment, so you can quickly be up and running again if the environment is rebuilt or terminated.

# Doesn't AWS already offer this?

Not in an easy way that meets all the above criteria.

- AWS does support an auto-renew certificate manager, but only if your HTTPS is on the load balancer (additional $18/month)
- API gateway and lambda could work, but there is a cold-start penalty (5 seconds or so in my experience) that is a non-starter for my applications

This repository and README are the result of many many hours of troubleshooting on AWS to get this set of instructions that works and meets the requirements above.

# What about DB/storage?

Databases can be expensive. For a side project, I don't want to spend $15/month for an RDS instance. Some options are:

- Use json files stored in S3
- Free mongodb sandbox at Mongolab https://mlab.com
- Amazon RDS (mysql, etc)
  - Reserved instances can lower the price significantly
  - Share between multiple apps

# Getting started

## Creating an Elastic Beanstalk environment

- Sign in to the AWS console https://aws.amazon.com/console
- Under the Services menu, go to EC2
  - In the left side bar, scroll down to Network and Security and click Key Pairs
  - Create a key pair to use for the next step
- Under the Services menu, click Elastic Beanstalk
- Create a new application
- Create a new environment (web server)
  - Node.js platform
  - Sample application
  - Click "Configure more options"
  - In the Instances box, click Modify
    - Select t3.nano for least expensive (about $4/month)
  - In the Capacity box, make sure it's single instance
  - In the Security box, click Modify
    - Select the EC2 keypair you created earlier
    - This will ensure you can easily ssh to your instance
  - Click Create Environment
- After the environment is created, you should be able to see the sample application at the env url shown in the Elastic Beanstalk env page

## Verify your DNS is setup and working

- If you're using Route 53, see https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-beanstalk-environment.html
- Confirm you can reach the sample application via your domain name 

## Configure nginx and https with auto-updating letsencrypt certs

- Clone this repository into a new project directory
  - git clone git@github.com:rydama/hello-eb-node.git your-project
  - to reinit as a fresh repo, `rm -rf .git && git init`
- Install the Elastic Beanstalk command line tools
  - https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html
- eb init
  - Select the application you created earlier
- Confirm ssh is working
  - eb ssh
  - Type 'yes' to the ssh authenticity warning
  - You should now be at the console for your instance
  - exit
- Configure nginx and letsencrypt
  - Find/replace YOURDOMAIN with your actual domain (i.e. winningpic.com) in the .ebextensions files
  - Find/replace you@example.com with your actual email in the `999-03-nginx-ssl-enable.config` file
- Confirm deploy is working
  - eb deploy
  - Once finished, visit your domain in the browser
  - You will see a certificate warning because we're using the letsencrypt "staging" certificate
    - i.e. in Chrome you will see "Your connection is not private" 
    - Click the Advanced button and then proceed
    - You should see the simple "Hello world" message at your domain
- If you see the Hello world message, then you are ready to switch from the staging certificate to the real certificate.
- Otherwise, view the `eb deploy` output, and the following log files to troubleshoot the problem
  - eb ssh
  - sudo less /var/log/letsencrypt/letsencrypt.log
  - sudo less /var/log/eb-activity.log

## Switching from a letsencrypt staging cert to a real cert

- Edit `999-03-nginx-ssl-enable.config` and remove the `--staging` parameter and add `--force-renewal`
- `eb deploy` to update. This should now get your real cert
- Visit your domain in the browser
- You should no longer see the certificate warning (it can take a few minutes for the cache to clear out. If you still see the warning, keep trying for a few minutes)
- Edit `999-03-nginx-ssl-enable.config` and remove the --force-renewal parameter
- `eb deploy` again to make it live
- Visit your domain in the browser to test one last time
- Letsencrypt certs expire in 90 days
  - `999-04-certbot-cron.config` installs a cron script to automatically attempt to renew your certificate once per week, so you will always have a valid cert. The cert won't actually get renewed until you are within 30 days of expiration.


## Extras

- $5 per month for mysql (could be shared by multiple apps)
- easy email forward

## Advanced

- ignore hack url at nginx https://forums.aws.amazon.com/thread.jspa?threadID=228527


## Logging

- /var/log/letsencrypt/letsencrypt.log
- /var/log/nginx/access.log
- cron log?
- healthd
