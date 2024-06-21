+++
title = "Cecil Gwak, Security Research Engineer"
description = "I implement what I research since theory will take you only so far"
+++

During my academic journey in graduate school, I was engaged in a project focused on ensuring 5G MEC (Multi-access Edge Computing) security, utilizing containers and virtual machines (VMs) on the edge. To achieve this, I identified the necessary protection methods required throughout the Software Development Life Cycle (SDLC), with particular emphasis on runtime security for containers.

During runtime, a program operates as an active process rather than merely existing as a binary file. Therefore, it needs robust, privileged, yet safe protection mechanisms. One such tool is eBPF (extended Berkeley Packet Filter), which provides the capability to enforce security policies effectively.

I implemented several security solutions using eBPF to enforce these security policies, thereby monitoring and protecting the entire cloud-native infrastructure. Here, you can see how to enhance security during a typical SDLC, along with my patents and papers that detail the implementation of these improvements. If you would like to see more of my achievements, you can find them here. ([notion page](https://cuboid-authority-f7b.notion.site/bec4d8208f754ccfb6044cce8c9c1a3d?pvs=4)).

![secure SDLC](https://github.com/g-song-i/g-song-i.github.io/assets/57793091/e352eb22-8e12-4008-924d-93a9163bd54e)

During my tenure at this company, I have conducted extensive research on setting up cloud infrastructure with various Cloud Service Providers (CSPs) such as AWS, Microsoft Azure, and Google Cloud Platform (GCP). Additionally, I explored cloud-native infrastructure using managed Kubernetes services provided by these CSPs, including Azure Kubernetes Service (AKS), Google Kubernetes Engine (GKE), and Amazon Elastic Kubernetes Service (EKS), as well as on-premises Kubernetes solutions.

I also delved into compliance requirements, both global and domestic such as CIS, PCI-DSS, ISMS-P and so on. Ultimately, I developed a prototype of a Cloud Security Posture Management (CSPM) tool to demonstrate the practical applications of my research.

My technical stack and background, which were essential in my research for CSPM, include but are not limited to:

![develop CSPM](https://github.com/g-song-i/g-song-i.github.io/assets/57793091/cc5298f7-69a7-4c0b-8a8c-c619b040c9bf)

Beyond this, I've researched other components of Cloud-Native Application Protection Platforms (CNAPP) to support a broad range of protection for cloud assets, including Cloud Infrastructure Entitlement Management (CIEM), Cloud Workload Protection Platforms (CWPP), and Application Security Posture Management (ASPM).

My current research objective is to research and develop PoC for CIEM (Cloud Identity and Entitlement Management). This project targets **Google Cloud** and **Google Workspace**, so far.
* Here, I research and develop functions for CIEM solutions:
    - **List up members and their roles, permissions**
    - **List up resources the role can access**, and what permissions actually they use when they access the resource
* Moreover, I set up all the software components myself to implement functions I listed up
    - **Configured the backend server using Go** for efficient data handling and operations.
    - **Developed the frontend server with JavaScript**, optimizing client-server interactions.
    - **Established API communications** between frontend and backend.
    - **Deployed MongoDB** for comprehensive data management, including database design and data insertion/retrieval operations

![develop CIEM](https://github.com/g-song-i/g-song-i.github.io/assets/57793091/54133400-e4dc-4324-a9cd-dd34fecebeae)