---
layout: post
title: "Building and Deploying Micro-services to Openshift"
date: 2018-01-21
---

Working in Enterprise software, Java / Springboot micro-services for 
distributed / high availability software is now the standard.
Because of how often I come across this I thought I'd take a moment to quickly 
run through how to deploy your own micro-services on Minishift 
( a personal Openshift cluster).


In my spare time I like to play around with questions from [Project Euler](https://projecteuler.net/archives).
So I'll be using a Gradle/Springboot project with solutions to a few of the problems
[available here](https://bitbucket.org/fosuoy/project-euler-solutions-java)


For this example (if you want to follow with) you'll need:
 - A working internet connection ;)
 - A copy of the [oc binary](https://github.com/openshift/origin/releases)
 - Access to a sample Gradle / Springboot project [(I'll be using mine)](https://bitbucket.org/fosuoy/project-euler-solutions-java)


You can find guides on getting started with the oc binary from Red Hat, personally
I've found set up to be very straightforward, placing the binary in my ~/.local/bin
directory.
The only thing to keep in mind is to configure an insecure docker registry in the 172.30.0.0/16 address space.


For example see the following daemon.json:


```
yousof@laptop:~$ sudo cat /etc/docker/daemon.json
{ 
    "dns":["8.8.8.8", "8.8.4.4"],
    "insecure-registries":["172.30.0.0/16"]
}
```


You will need to restart your docker daemon if you apply the above configuration:


`systemctl restart docker`


Separately I found another issue with Openshift where I was unable to check code 
out of github within the Openshift environment with search urls in my resolv.conf.


So to be on the safe side you can edit the runtime config and remove any search 
instructions from your: /etc/resolv.conf, however it's likely this won't affect you.


Moving on - start your Openshift cluster:


`oc cluster up`


and login:


`oc login -u developer -p developer`


Peronally, I am familiar with Gradle / Springboot and so Red Hat's Wildfly build 
image which looks for a maven pom.xml will not be useful for us.


To example on what the above means, apart from just being a container orchestration
service like Kubernetes, pulling and deploying images from a docker registry - 
Openshift is made for large enterprises to smooth out their deployment pipelines, 
and allow multiple teams / applications to work on different applications on 
multi-tenant servers.


The part of their service that involves pipelines also includes a very useful 
feature called Source-To-Image (S2I) which pulls some source from a git repo, 
builds it in a builder image, then pushes a new image with the compiled / 
deployment ready image to the Openshift registry. Webhooks monitor the repo for 
new commits so that the new source can be compiled / deployed as soon as it's 
ready.


Thankfully there is a gradle/java s2i image on docker hub
[build image](appuio/s2i-gradle-java) 
for Gradle applications, it is straightforward to create your own but I won't go 
into that here.


In order to add this new builder image, follow along:


```
yousof@laptop:~$ oc login -u developer -p developer
Login successful.

You have one project on this server: "myproject"

Using project "myproject".

yousof@laptop:~$ oc -n myproject import-image appuio/s2i-gradle-java --confirm
The import completed successfully.

...
<You should see details of the import here>
```

So all that has happened above is that you have logged in and imported a new s2i
image. In order to use it in the UI of Openshift 
(a more pleasant experience if you're not administrating the instance!) you'll 
have to properly tag the image stream:


```
yousof@laptop:~$ oc -n myproject edit is s2i-gradle-java
...
 17 spec:                                                                            
 18   lookupPolicy:                                                                  
 19     local: false                                                                 
 20   tags:                                                                          
 21   - annotations: null                                                            
 22     from:                                                                        
 23       kind: DockerImage                                                          
 24       name: appuio/s2i-gradle-java
 25     generation: 1                                                                
 26     importPolicy: {}                                                             
 27     name: latest                                                                 
 28     referencePolicy:                                                             
 29       type: Source 
...
Change to:
 17 spec:                                                                            
 18   lookupPolicy:                                                                  
 19     local: false                                                                 
 20   tags:                                                                          
 21   - annotations:                                                                 
 22       description: OpenJDK builder image!                                        
 23       tags: builder,java 
...
```

Now you can fire up the UI and take a look at what we've done!


Go to your [Openshift Console which should be hosted at localhost:8443](https//localhost:8443) 


Login using the same credentials (user: developer, password: developer)


![Login_screen]({{ BASE_PATH }}/images/openshift_tutorial/1/Selection_001.png)


Navigate to your project


![Project_screen]({{ BASE_PATH }}/images/openshift_tutorial/1/Selection_002.png)


Create a new project:


![Project_screen_2]({{ BASE_PATH }}/images/openshift_tutorial/1/Selection_003.png)


Select Java:


![Java_screen]({{ BASE_PATH }}/images/openshift_tutorial/1/Selection_004.png)


Select the image we just added (present here because of the added annotations):


![Java_screen_2]({{ BASE_PATH }}/images/openshift_tutorial/1/Selection_005.png)


I have a project ready here:

https://bitbucket.org/fosuoy/project-euler-solutions-java


![Java_screen_3]({{ BASE_PATH }}/images/openshift_tutorial/1/Selection_006.png)


I can give you a break here to explore the interface.
It will take some time while Openshift downloads the image, checks out the code 
ompiles and pushes the new image to the registry.


Personally I've found the UI for Openshift to be a bit nicer than Kubernetes but
functionally they are very similar - you can follow along the build container log
followed by the deployed container and see the standard Springboot start up logs.


Finally you can checkout Applications -> Routes to see the running application.
In my case:


```
yousof@laptop:~$ curl http://springboot-app-myproject.127.0.0.1.nip.io/
Current urls:
/problem_001/ 
/problem_002/ 
/problem_003/ 
/problem_028/ 

yousof@laptop:~$ curl http://springboot-app-myproject.127.0.0.1.nip.io/problem_001 | jq '.'
{
  "answer": "233168",
  "blurb": "If we list all the natural numbers below 10 that are multiples of 3 or 5, we get 3, 5, 6 and 9. The sum of these multiples is 23.\nFind the sum of all the multiples of 3 or 5 below 1000.\n",
  "url": "https://projecteuler.net/problem=1"
}
```


Where above you can see the use of one of my favourite tools jq.


I feel a lot has been covered here and I hope to expand on some of it in future - 
that is cool things you can do with jq, or using gradle / springboot to quickly create
JSON based APIs as well as loadbalancing / splitting of routes within Openshift.


All good topics to follow up on in future!
