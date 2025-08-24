# Overview

A repository outlining journaling details of projects and career highlights.

This repository is not an exhaustive list of my work, but rather, learning moments, milestones, or work that has sentimental value. 
All documents outline design, architecture, and implementation details, as well as personal reflections and anecdotes.

## `backend/`
- [Client and REST API for creation and configuration of Dask Custers within a cloud-native Kubernetes environment](backend/dask-client.md) (2024)
- [Client and REST API for serving browser embedded IDE instances via secure reverse proxy from within a kubernetes cluster using websockets](backend/ide-proxy.md) (2024)
- [Server-Side Event (SSE) stream with REST API endpoints for efficient live updates via Kubernetes Informers](backend/informers.md) (2024)
- [SaaS platform server design and core contributions; various anecdotes and information.](backend/server.md) (2024)

## `cicd/`
- [ArgoCD configuration, hosting, deployment](cicd/argocd.md) (2022)
- [Continuous delivery with GitOps, and CI integration](cicd/cd.md) (2023)
- [The Empathetic Monorepo: CI pipeline design](cicd/ci.md) (2023)
- [Total CI/CD system failure; A tale of leadership, initiative, recovery, and limited sleep](cicd/ci-cache-artifacts.md) (2023)
- [Gitlab runner configuration and deployment with Helm and ArgoCD](cicd/gitlab-runners.md) (2022)

## `geospatial/`
- [A REST API, spatial database, and Django client for management of large LiDAR datasets.](spatial/eclipse.md) (2020)
- [A LiDAR QA/QC pipeline tool developed early in my career for the province of BC. An awaking for my love of programming.](spatial/liqcs.md) (2018)
- [A tool for evaluation of the absolute vertical accuracy of LiDAR data, utilizing eigen-based surface detection, statistical hypothesis testing, and spatial indexing.](spatial/vertigo.md) (2020)

## `gijo/` (2018)
This directory contains all documentation, personal notes, diagrams, and implementation details of the `gijo` chat protocol and client/server application.
 
GiJo represents a pivotal moment in my technical journey—a C-based networking application that sparked my passion for software architecture and system design. 
Originally developed as a school project, its design principles and implementation of low-level networking concepts demonstrate stand the test of time with 
regards to programming capabilities and architectural thinking. 

This project implements a custom-designed chat protocol using the Berkeley Sockets API, enabling reliable 
communication between networked clients while demonstrating advanced of concurrent programming principles and network protocol design.

The architecture embodies several key design principles:
1. Separation of Concerns: Clear boundaries between networking, protocol implementation, and application logic
2. Modularity: Self-contained components with well-defined interfaces
3. Protocol Simplicity: A carefully designed messaging system that handles only what's necessary
4. Resource Efficiency: Careful memory management and performance considerations
5. Polymorphism in C: Advanced object-oriented patterns implemented in pure C
6. Dual-Protocol Support: TCP for reliable messaging and UDP for real-time voice


## `infrastructure/`
- [Revival of cloud-infrastructure deployments for full idempotent automation using bicep IaC for Azure](infrastructure/bicep.md)
- [Designing templated manifest configurations for dynamic deployment of multi-environment systems via CI/CD integration.](infrastructure/deployments.md)
- [Configuring a remote debugger for kubernetes deployed applications with CI/CD integration for development environments.](infrastructure/k8s-debugging.md)





