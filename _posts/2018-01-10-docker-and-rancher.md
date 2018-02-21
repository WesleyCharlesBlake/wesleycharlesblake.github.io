---
layout: default
title: "Docker and Rancher"
---

### Rancher — A software platform for deploying a private container service.

I have been on a drive to improve our development workflow (and culture). To keep up with demands, any tech company needs to make adjustments to not only to their core products, but ideally to the operational side of things. From concept, to release to market, and all the facets that form a part of your organisation are affected by how we can deploy, maintain, and grow any product.

Scalability is a key term that I throw around daily, and its something that shows its face more frequently and will become growths best friend. In my company, we have a highly distributed application and a multi tiered infrastructure. This has caused me many head aches and late nights to ensure that Ops can manage normal day-to-day tasks, as well as provision for the future. However a “Top-Down” influence in the product, lead to some short sightedness without consideration for rapid growth of our user base, which ultimately impacts the infrastructure, deployment, scalability and so general operations became a nightmare.

This is a huge technical debt that I now have clear. As top level management do not understand a ROI on automation, scalability, and simpler work flows. They see it as cost and unnecessary expense to the company, as we would need to pull engineers off current (billable) tasks, to be able to rectify this situation. However the reality of this issue is a huge set back. Work flow is disrupted, features take too long to role out, time is spent(wasted) on tasks that yield no outcome.

As the sole member of DevOps here, it is my job to figure out a cost effective way to address all of these problems, and create a scalable platform.
The answer was in plain sight.

* docker — for production
* docker-compose — for development

Rancher — for managing clusters, staging/qa/prod environments
These 3 tools will become the start of my saving grace. And here’s what I did to solve our unique “business decision” problems

Docker is an awesome tool for distributed applications. It allows you to run, build and ship your product. All we needed to do was build a docker image (for each particular version we have in production) and use this image as base for each application. So immediately a lot of work has been cut out. I know longer have to provision an environment-per-version-per-developer. A bonus is that your environments can be locked down to a container, so the story of “but it works in dev, it must be a hosting issue!” is no longer a valid argument! (a big win for DevOps and Sysadmins!)

Now the next challenge this brought along, was educating the developers on how to dev locally using docker. This is where `docker-compose` shows its big guns.
Using Compose is basically a three-step process.
Define your app’s environment with a `Dockerfile` so it can be reproduced anywhere.

Define the services that make up your app in `docker-compose.yml` so they can be run together in an isolated environment.
Lastly, run docker-compose up and Compose will start and run your entire app.
So this allowed me to provision our core/base image, and the devs could easily extend on this using Compose, by adding to their docker-compose.yml file in their current project.

>Our docker-compose.yml in the project could look like this:
```
web:
  build: .
  ports:
   - "5000:5000"
  volumes:
   - .:/code
  links:
   - redis
redis:
  image: redis
```
So now that production, and dev has been taken care of, how would I make sure I can keep an eye on everything, scale up resources where needed, ensure an effective, redundant and highly available infrastructure? Rancher came to my rescue.
I had a look at various other tools such as Google’s Kubernetes, and docker swarm. But what I found in Rancher was the flexibility to cater for our distributed applications (and it was free). I found it simple to use, and it met my needs pretty well.

First thing was to set up a host to run your rancher server.
I simply spun up a small bare-bones 2GB droplet on Digital Ocean using Ubuntu 14.04 64bit. I would suggest a decent amount of ram (1gb is minimum). Ubuntu and Rancher work really well together, so unless you plan on getting really stuck into your distro, I would stick to Ubuntu.
Once the VM was running, ssh into it, and install docker.
```
$ wget -qO- https://get.docker.com/ | sh
```
Then all thats needed is install and run Rancher. What makes Rancher neat, is that is packaged and run in a docker container (so a docker service to run all your docker services.).

All that is needed is to pull the docker image and run it:
```
$ sudo docker run -d — restart=always -p 8080:8080 rancher/server # Tail the logs to show Rancher
$ sudo docker logs -f containerid
```
When the logs show ….
```
Startup Succeeded, Listening on port 8080
```
Rancher UI is up and running. As simple as that. Visit http://<host_ip>:8080 (eg: http://rancher.mydomain.com:8080) to access Rancher. (you could also run the setup on your local machine to test it out locally, in which case you would visit http://localhost:8080)

> Note: Rancher will not have access control configured and your UI and API will be available to anyone who has access to your IP. We recommend configuring access control. What is even better, is that you can configure your rancher instance’s access to authenticate using an existing LDAP server. Everything is great with LDAP ;)

The first thing I did was set up my environments, and provisions hosts for each environment. This way, I could have a QA, Staging, NOC and Production environment, each with their own cluster of hosts. This meant I could keep a tight wrap on the different environments, and have piece of mind that should anything go wrong on in QA/Dev, Production will remain unaffected (a no-brainer really)

![alt](https://cdn-images-1.medium.com/max/1200/1*lFNjH8_UI8K7PawziA2RAw.png)

I found that using Digital Ocean was simple to provision hosts with Rancher, however in rancher you can access your AWS account to spin up EC2 Instances, or use your own hosts if needed. In your DO account, generate an Access token, and take note of it, as you will need this for Rancher to be able to spin up hosts when you require — The power of API’s!
![](https://cdn-images-1.medium.com/max/1200/1*AuCnnQNTxNIYHe0vKC0QXQ.png)
Next thing on the list is to add your Docker registry to rancher, to allow you to pull your images from your own private docker registry. Alternatively you could spin up a registry service in rancher (using Docker’s registry image), and run your own private registry as I did. However you will need to have a valid SSH certificate in order to be able to push your local image build to your now new docker registry.
Rancher gives you free rain to be able to mount volumes where needed, so you can keep data storage persistent.

What I really found a life saver, was spinning up new stacks and services when needed, upgrading certain stacks to newer versions (with the ability to roll back), without any impact on the production application. Rancher keeps your existing container running until you complete your upgrade, and the deletes the old service once the upgrade has been completed. A very seamless process.

##### Rancher UI — Applications dash board
Need extra services to help? You can now simply scale up your services, using the same image and config that is required. Don’t need them any more? Just scale it down again. Need more or less hosts? Its the same process.

![](https://cdn-images-1.medium.com/max/2000/1*XqNj3IgY3FFButvCqHTJiw.png)

Another great feature I make use of, is Rancher’s Load balancer. For a simple use case, you could set up a Common stack, and create a “common-lb”. Then you can resolve a domain, on port 80, to the port exposed in your container. It is really simple, and highly affective, however in more complicated scenarios, you would need a web server such as Nginx/Apache running either as a shared resource between apps, or each app could have its own webserver part of its stack — this way you can handle custom URL rewrites that may be needed (think Django and staticfiles).

I have now achieved my objectives that DevOps needed to address. Deployments are simplified, I can spend less time helping devs in the office, and more time making sure our services are running. New applications are deployed in a matter of minutes, and I have the freedom to scale any aspect of our Hosting Infrastructure to my hearts content.
Deployments are now also painless.

You could also create a “Travis-CI like” Jenkins build. To achieve this, a solid Git workflow is required. Features are developed in a feature branch. When ready, a pull request is opened, Jenkins runs a serious of pre-build tests (unit tests etc). If everything passes, Jenkins builds the image, tags it and pushes it the registry, and then deploys it to prod. (This is something I am still working on, and will publish this soon).

The one thing I have realised in all of this, is that there is not one solution that can shoe horn all your Dev and Ops requirements into one. We can get pretty close though, but each company and organisation have different needs. It takes time to implement new technology stacks. It takes a greater amount of time to change an internal work culture. People fear change, and as I have found it, that has been the biggest hurdle. I can distinctly recall migrating our VCS from svn to git. And the months of moaning, fighting, screaming and crying I had to deal with on a daily basis. But now, without Git, I would not have been able to achieve any of this, or any of our DevOps future milestones.

I also had to convince management to support me and allow me to create a system that will rectify a fundamental flaw in their initial product’s design. It has not been an easy road, but they are starting to see the vision I have, and the great benefits that will be inherited.

Do some research. Set up a proof of concept, get your hands dirty and figure out what it is that you need, and make it happen!
