---
layout: post
title: QRA Design Goals
date: 2014-04-07
tags: ["rails", "qra"]
category: Programming
notebook: 工作日志
---

## What is QRA
QRA stands for QAD reference architecture. (Q: why is this name?). It is an important part of software foundation for QAD. Others lists:

* BPM
* QRA
* QXtend
* SF - Appshell
* R & B
* BLF

### Ojbectives of QRA
better support for web development?
allow the possibility to migrate BL and UI to other technologies.

#### QRA modularity
Module should have the following features:

* independent
* interchangeable
* foundation module should included
* QRA modules should be upgraded /deployed/ independently without having to upgrade other module

Questions:
Customization support,
Global development for multiple function groups.
build/deploy tools support.

#### QRA UI evolution
QRA should make it easy to develop UIs with a common look and feel and allow easy evolution to use a different UI technology as necessary.

#### QRA consistency
QRA should make the development and deployment of modules easy and consistent by providing standard envrionments, tools, guidelines and processes.

* qra guide line
* qra patterns
* qra standards
* foundation layer services

#### Additional Goals

* QRA Quality
* QRA backward compatibility
* Cost reduction
* Service oriented architecture

#### QRA Migration

**Migration of UI**

* Using API make is easy to use any client technology that can implement and use API's and use webservices.
* Using entity metadata and view metadata its possible to have all clients have the same look and feel whatever the technology.
* UI clients don't have any business logic to process any informaton received from the webservice. This is all handled by the BL.

**Migration of BL**

* Having BL API exposed as a webservice makes it easier to move to any technology that supports webservices.
* The BL contains all the logic and the API's are available via the webservices to the client.
* As modules are small and self-contained its possible to write/rewrite modules in any other languages that support API's and webservices.

### Goals 

improve on productivity, easy to crud, validation.

improve quality,

easy to change to use other ui tech.

