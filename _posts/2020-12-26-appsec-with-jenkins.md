---
layout: post
published: true
categories:
  - personal
mathjax: false
featured: false
comments: true
title: AppSec with Jenkins
---
## AppSec with Jenkins

Forked OWASP's [Vulnerable-Web-Application](https://github.com/OWASP/Vulnerable-Web-Application) for this excersise. The plan is to have security tools run in Jenkins every time we have a new build from the project.

Our security team will track diff's of the security tooling output to detect builds that introduced new vulnerabilities. 
![jenkins_git_url.JPG]({{site.baseurl}}/images/jenkins_git_url.JPG)


Jenkins will run the new build with docker and execute scan(s).
![jenkins_commands.JPG]({{site.baseurl}}/images/jenkins_commands.JPG)


We will be using the [Web Application Security Scanner](https://github.com/Arachni/arachni) for this excersise. The target will be Vulnerable-Web-Application's level 1 XSS page.
![vuln_xss_page.JPG]({{site.baseurl}}/images/vuln_xss_page.JPG)


As expected, Arachni has detected the XSS vulnerability. 
![arachni_ouptut.JPG]({{site.baseurl}}/images/arachni_ouptut.JPG)

We can use this output to make subsequent code changes to remediate the vulnerability.
