---
title: How to Study for the Certified Kubernetes Exams - CKAD and CKA
date: 2019-02-01T00:00:00+00:00
tags: [ "technical", "linux", "kubernetes", "ckad", "cka" ]
draft: false
---

## Intro

At work, we've asked our team members to pass the official Linux Foundation certification exams for Kubernetes.  The exams are challenging, practical, and realistic. They serve perfectly as a *minimum* bar for our engineers who work with Kubernetes.  The exams don't cover *everything* that is typically needed to manage a Kubernetes cluster (e.g. helm, helmfile, etc.), but they do ensure that an engineer demonstrates basic competence.

## The CKAD and CKA Exams

The exams are:

* **Certified Kubernetes Application Developer (CKAD)** covers the basics of how to configure, deploy, and debug applications within a Kubernetes cluster.
* **Certified Kubernetes Administrator (CKA)** covers all of the material from the CKAD, as well as the basics of how to configure, deploy, and debug the Kubernetes cluster-level infrastructure itself including masters, nodes, kubelet, scheduler, etcd.  The CKA is more difficult than the CKAD.

The certifications are taken online with a live human proctor via webcam, screensharing, and web browser.  The CKAD is a 2 hour exam with a 66% score required to pass, while the CKA is a 3 hour exam with a 74% score required to pass.  Both exams require the user to interact and solve problems on real Kubernetes deployments at the command line using the `kubectl` utility.  The exams are realistic, and do define a minimum bar.  The exam fee is $300 before applying easily found discount codes.

## How Hard are the Exams?

Individuals on our team have taken anywhere between 1 day and 1 week to study.  We all passed the exams on first attempts.  We all have significant experience with Kubernetes.

One team member prepared for 1 day and was able to pass both exams, taking the harder CKA first.  The other team members took up to 1 week to prepare, but felt much more confident after taking the exams.

Please note that there isn't much risk to failing the exam, because an exam re-take is included without additional cost if you have registered for the exam directly on https://www.cncf.io.

## How to Study

* Read through the exam handbook and /
  * https://www.cncf.io/certification/candidate-handbook
  * https://www.cncf.io/certification/tips/
* Take the CKAD before the CKA.
  * The CKAD is easier, and requires a lower passing score
  * The CKA includes everything on the CKAD, plus cluster admin tasks
* Get comfortable with the linux terminal.
    * Each exam is live at a single linux terminal
    * Know how to type quickly
    * Know how to use a terminal multiplexor such as `screen` or `tmux`
    * Know how to use an editor such as `vi` or `emacs`
* Run through the CKAD exercises here:
  * https://github.com/dgkanatsios/CKAD-exercises
* (CKA-only) Practice Kubernetes the Hard Way
  * https://github.com/kelseyhightower/kubernetes-the-hard-way
* If you wish to experience the web browser linux shell which is the primary interface to the exam, you may setup GateOne.
  * Docker image: https://hub.docker.com/r/dcwangmit01/gateone
  * Source code: https://github.com/dcwangmit01/gateone
    * Forked, because the original wouldn't build and seems to be unmaintained.


## Exam Tips

* Get comfortable with `kubectl` and its tab completion.
* (CKA-only) Get comfortable with `ssh`
* (CKA-only) Get comfortable with `systemd` tools such as `systemctl` and `journalctl`.
* Know how to use `kubectl run ... --dry-run -o yaml` to generate yaml snippets without referring to the documentation for pods, deployments, etc.
* Be familiar with `kubectl explain ...` 
* Save your nodes and yaml snippets as you go along, since you can reuse them for later questions (e.g. `question1-pod.yaml`)
* Work quickly and only double-check if necessary.
  * Each exam is conducted under serious time pressure.
  * Most of us ran out of time and were unable to address every single exam question.
* You are allowed bookmarks in your Chrome web browser.

## Links

* CKA Registration Link
  * https://www.cncf.io/certification/cka/
* CKAD Registration Link
  * https://www.cncf.io/certification/ckad/
* Discount codes for Linux Foundation Exams
  * We were able to get 15% off
  * https://www.retailmenot.com/view/training.linuxfoundation.org
