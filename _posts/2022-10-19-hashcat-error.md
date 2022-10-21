---
title: HASHCAT Opencl Error
date: 2022-10-19 23:23:00 +0800 
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [error_fix]     
---

>#PROBLEM DESCRIPTION

This kind of error comes up when you dont have the latest version of OPEN_CL installed or maybe there is a lack of LLVM 4.0 on your system.

>#PROBLEM SOLUTION

So if this error pops up when you try to execute "hashcat" on any file:

>clBuildProgram(): CL_BUILD_PROGRAM_FAILURE 
{: .prompt-danger }

>error: unknown target CPU 'generic' 
{: .prompt-danger }

>Device #1: Kernel /usr/share/hashcat/OpenCL/shared.cl build failed. 
{: .prompt-danger }

We have to follow these steps:

> 1. Go to http://registrationcenter-download.intel.com/akdlm/irc_nas/12556/opencl_runtime_16.1.2_x64_rh_6.4.0.37.tgz and download the file into any directory in your system 
{: .prompt-tip}

> 2. tar -vxf (name_of_the_file_you_just_downloaded) 
{: .prompt-tip} 

> 3. cd (name_of_the_file_you_just_downloaded) 
{: .prompt-tip} 

> 4. ./install.sh
{: .prompt-tip}

> 5. Follow the stepts, and then execute hashcat again. Now it should work.
{: .prompt-tip}
