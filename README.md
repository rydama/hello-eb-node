# AWS Elastic Beanstalk with Node and SSL

A quick start for creating a web app on AWS with the following characteristics:

- Ideally suited for low-traffic side projects
- Quickly and painlessly deploy apps with https
- Dirt cheap to start ($5/month). Easy to scale up as needed.
- Free, auto-renewing SSL certificate with Let's Encrypt
- Provisions automatically from a brand new Elastic Beanstalk environment, so you can quickly be up and running again if the environment is rebuilt or terminated.

This repository and README are the result of many many hours of troubleshooting on AWS to get to this set of instructions that works and meets the requirements above.

Why was it so difficult? AWS does seem to offer easier ways to get https, but it is expensive or has other problems:

- AWS does support an auto-renew certificate manager, but only if your HTTPS is on their elastic load balancer (additional $18/month)
- API gateway and lambda could work, but there is a cold-start penalty (5 seconds or so in my experience) that is a non-starter for my applications

# Getting started

- These steps assume familiarity with basic use of the linux shell/mac terminal

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

## Configure nginx and https with auto-updating Let's Encrypt certs

- Clone this repository into a new project directory
  - `git clone git@github.com:rydama/hello-eb-node.git your-project`
  - to reinit as a fresh repo, `rm -rf .git && git init`
- Install the Elastic Beanstalk command line tools
  - https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/eb-cli3-install.html
- Run `eb init`
  - Select the application you created earlier
- Confirm ssh is working
  - Run `eb ssh`
  - Type `yes` to the ssh authenticity warning
  - You should now be at the shell for your EC2 instance
  - `exit` from the shell
- Configure nginx and let's encrypt
  - Find/replace `YOURDOMAIN` with your actual domain (i.e. winningpic.com) in the `.ebextensions` files
  - Find/replace `you@example.com` with your actual email in the `999-03-nginx-ssl-enable.config` file
- Confirm deploy is working
  - By default the Elastic Beanstalk deployment uses the master branch in your local github repository. Be sure to commit your changes before any invocation of `eb deploy` below.
  - Commit the above changes to the `master` branch in your local repository
  - Run `eb deploy`
  - Once finished, visit your domain in the browser
  - You will see a certificate warning because we're using the Let's Encrypt "staging" certificate
    - i.e. in Chrome you will see "Your connection is not private" 
    - Click the Advanced button and then proceed
    - You should see the simple "Hello world" message at your domain
- If you see the Hello world message, then you are ready to switch from the staging certificate to the real certificate.
- Otherwise, view the `eb deploy` output, and the following log files to troubleshoot the problem
  - Run `eb ssh`
  - In the EC2 shell:
  - `sudo less /var/log/letsencrypt/letsencrypt.log`
  - `sudo less /var/log/eb-activity.log`

## Switching from a Let's Encrypt staging cert to a real cert

- Edit `999-03-nginx-ssl-enable.config` and remove the `--staging` parameter and add `--force-renewal`
- `eb deploy` to update. This should now get your real cert
- Visit your domain in the browser
- You should no longer see the certificate warning (it can take a few minutes for the cache to clear out. If you still see the warning, keep trying for a few minutes)
- Edit `999-03-nginx-ssl-enable.config` and remove the --force-renewal parameter
- `eb deploy` again to make it live
- Visit your domain in the browser to test one last time
- Let's Encrypt certs expire in 90 days, so `999-04-certbot-cron.config` installs a cron script to automatically attempt to renew your certificate once per week, so you will always have a valid cert. The cert won't actually get renewed until you are within 30 days of expiration.


## Advanced

- Configure nginx to ignore hack urls
  - https://forums.aws.amazon.com/thread.jspa?threadID=228527


## Logging

If you have trouble getting things to work, run `eb ssh` and view the following log files:

- `/var/log/eb-activity.log`
- `/var/log/letsencrypt/letsencrypt.log`
- `/var/log/nginx/access.log`

# What about DB/storage?

Databases can be expensive. For a side project, I don't want to spend $15/month for an RDS instance. Some options are:

- Use json files stored in S3
- Free mongodb sandbox at Mongolab https://mlab.com
- Amazon RDS (mysql, etc)
  - Reserved instances can lower the price significantly
  - Share between multiple apps
