---
layout: post
title: Using CWL and Toil to Wrap an Ad-hoc Astronomy Data Processing Pipeline
excerpt_separator:  <!--more-->
description: I recently taught some students how to use the Common Workflow Language (CWL) to do reproducible and automated scientific data processing pipelines. Here's what I learnt.
categories:
    - technology
tags:
    - science
    - astronomy
    - reproducible
    - automation
    - data
    - big data
    - workshop
    
---
<figure>
    <img class="img-responsive" src="/assets/images/jedi.jpg" alt="Blasting Students with Science"/>
    <figcaption style="text-align: center">Blasting Students with Science</figcaption>
</figure>


I was recently invited to give a workshop on reproducible scientific workflows to students as part of the Inter-university Institute for Data Intensive Astronomy's ([IDIA](http://www.idia.ac.za)) "[JEDI](http://www.idia.ac.za/workshop/jedi-madagascar)" programme. The overall purpose of this workshop was to introduce students from the African continent to various topics that are being dealt with in the data science space. A large focus here was machine learning.

This post details some of my experiences with preparing the original pipeline, working CWL around it and also teaching people how to do it.

<!--more-->
## Background

I was originally brought into knowledge about the event by my boss, who had the idea of getting me involved to give students a workshop on something slightly higher level than domain specific science. Due to the growing concept of research cloud computing and my involvement in the South African Data Intensive Research Cloud (SADIRC) joint project, we settled on the idea of giving students a crash course in infrastructure-as-code.

We were met with some hesitation after pitching the idea to some of the organisers. Direct scientists often don't understand the usefulness of knowing some higher-level stuff about the environments that they run their code on.

Eventually I thought about reproducible scientific processes. I had been involved in turning bioinformatics pipelines into reproducible workflows in a [hackathon held by the H3Africa](https://aasopenresearch.org/articles/1-9/v1), where I was exposed to the [Common Workflow Language](https://www.commonwl.org/) (CWL).

They agreed to have me give a workshop on using CWL to create reproducible data processing workflows, using an existing astronomy calibration and imaging pipeline created by [Prof. Russ Taylor](http://idia.ac.za/prof-russ-taylor).

## The Original Pipeline

Whew, that was a mouthful. Prepare for more!

> To preface, I am not an expert with CWL and had fairly little experience with using it up until this point. I merely have a passion for automated and reproducible scientific work and have some experience with other workflow frameworks.

I received copies of the code used for the original pipeline. This code was all written in Python and some parts of it took advantage of a toolkit for astronomy called the _Common Astronomy Software Applications_ ([CASA](https://casa.nrao.edu/)).

The original code was all written with in [Jupyter](https://jupyter.org/), which is a fantastic frontend and tool for distributing Python code. Theoretically this should all be normal and fine, until you have a look at what was actually being done here.

The pipeline is broken up into multiple steps, for which each do something onto the resulting work from the previous step. The first step, calibration, takes data from a telescope and calibrates it to a known good source. Metadata about the datasets that are available for processing, including the amount of observations in each dataset, are defined by a simple `datasets.py` file, which contains classes to define them. This is imported into every part of the pipeline.

![Simple Breakdown](/assets/images/russ_pipeline_simple_1.png){:class="img-responsive"}

Each part of the pipeline has shared parameters that apply to the dataset and observations in question, as well as their own parameters which are unique to that step. This is repeated in every Jupyter notebook.

The pipeline also makes use of the [slurm](https://slurm.schedmd.com/) scheduler, where it is run against various virtual machines that act as an HPC environment ontop of the IDIA cloud (don't get me started...). The scripts being written in Jupyter do not allow this integration directly. To remedy this, the Jupyter notebooks effectively generate a large string of Python code syntax and tune it with the settings specified in the parameters set at the top of the given notebook. This string is written out as a plain `.py` file to the disk. A definition file for the scheduler is also written out and is run using the `os.system()` function from within the notebook, which then runs this script in a CASA [Singularity](https://singularity.lbl.gov/) container.  

This is an interesting solution to say the least! The major advantage here is that you can prepare the dataset/observation and horizontally scale that out with additional slurm jobs.

## Preparing for CWL-ification

CWL expects work units to fit into their idea of a "tool". Basically, a tool is a script or binary which takes in some pre-defined input and outputs some expected results. This does not necessarily have to be in terms of the results of the analysis you're processing, but rather the files or text that you are expecting as output from this process.

This was the first challenge. None of the existing notebooks could be directly used as a tool for CWL and they were also doing too many things to be considered a coherent "tool" that focussed on doing one thing.

Each of the pipeline steps needed to be reworked. With maybe more than a month before the actual workshop, I had to focus on getting things working rather than making them pretty. I ended up splitting the preparation step out into its own tool, which would output a `JSON` file that would be used by the actual pipeline step. This was converted from this Python-code-as-a-string-to-be-written-out-to-a-script form into a separate Python script that used a functions and parameters to apply parameter changes.

For each step in the pipeline 