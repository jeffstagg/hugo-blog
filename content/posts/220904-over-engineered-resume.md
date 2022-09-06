+++
author = "Jeff Stagg"
title = "Introducing the Over-Engineered Resume"
date = 2022-09-04
description = "Every coder needs a project to hack away at. We're going to over-engineer a resume to play with some code."
tags = [
"resume",
"portfolio-building",
"code samples"
]
categories = [
"tech",
]
series = ["Over-Engineered Resume"]
+++

## Introducing the Over-Engineered Resume
Resumes are boring, and it can be challenging to really demonstrate expertise in a craft through a document outlining your experience. Have you ever wanted to show off something with code, but all your work is zipped up behind an NDA? Ever wanted a project to hack away at and try out some new technology?

I want to build out a resume. And I don't necessarily mean a document to email somebody in hopes I've said the right words to let them know I'm not an idiot and can handle their project. I mean accomplishing the true goal of a resume: to demonstrate skill. I want to be able to give someone a QR code and have them bring up my resume on their device. I want to be able to screenshare during an interview and show off code, automated processes, security configurations, monitors and logs, everything you'd expect to demonstrate when showing a new team member around your software.

## The Plan
Let's decide on an MVP:
- View standard resume information
	- name / contact info
	- short bio
	- experience
	- projects
	- education
- Multi-tiered app following CLEAN Architecture (Rob Martin) pattern
- Continuous integration / continuous deployment to cloud
- Containerized and deployed with kubernetes

We can start here, and add items as we go, but this will be a baseline that we'll strive for before we start hacking away with more features.

## Next Up
We'll get started by setting up repositories and building out our development environment. I want to be able to hack away on this project from any of my computers running Windows or Linux, without having to install a lot of software. Development should be convenient, and we should strive to make our environment as developer-friendly as we can.