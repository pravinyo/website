---
title: "Comprehensive Roadmap for Low-Level and High-Level Design Interview Preparation"
author: pravin_tripathi
date: 2024-06-09 01:00:00 +0530
readtime: true
img_path: /assets/img/system-design-roadmap/
attachment_path: /assets/document/attachment/system-design-roadmap/
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, system-design, interview]
image:
  path: header.jpg
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using deepai.org
---

System design interviews are a critical part of the hiring process for software engineering roles, especially for senior positions. These interviews assess your ability to design scalable, maintainable, and efficient systems to solve real-world problems. In this blog, we’ll cover a **combined roadmap** for both **Low-Level Design (LLD)** and **High-Level Design (HLD)** interview preparation, along with strategies, expectations, and steps to excel in these interviews.

---

## What is a System Design Interview?

In a system design interview, you are given a real-world problem and expected to design a system to solve it. Since no system is perfect, you must also identify trade-offs, pros, and cons of your design. The goal is to evaluate your ability to:

1. **Analyze requirements** and define system constraints.
2. **Design scalable and maintainable systems**.
3. **Identify trade-offs** and justify your design decisions.

Whether it’s low-level or high-level design, the core focus is on your problem-solving skills, understanding of system architecture, and ability to communicate your ideas effectively.

---

## Low-Level Design (LLD) Interviews

Low-Level Design focuses on the **implementation details** of a system. It involves designing class structures, defining entities, and applying design principles and patterns to create a robust and extensible system.

### Expectations in LLD Interviews:
1. **Requirement Gathering**: Identify and define system requirements and constraints.
2. **Entity Identification**: Define entities (classes) and their relationships.
3. **Design Principles**: Apply principles like **SOLID** to ensure good design.
4. **Design Patterns**: Use patterns like **Strategy**, **Builder**, **Singleton**, etc., to solve common design problems.
5. **System Maintainability**: Ensure your design is extensible, loosely coupled, and easy to maintain.

---

### LLD Learning Strategy

#### 1. **Understand Core Concepts**
   - Learn the fundamentals of **object-oriented programming (OOP)**.
   - Understand **design principles** like SOLID, DRY, and KISS.
   - Study **design patterns** such as:
     - **Strategy**
     - **Builder**
     - **Singleton**
     - **Factory**
     - **Command**
     - **Composition over Inheritance**

#### 2. **Practice Problems**
   - Solve real-world problems like designing a **Parking Lot**, **Elevator System**, or **Library Management System**.
   - Focus on identifying trade-offs and comparing your design with others.

#### 3. **Steps in an LLD Interview**
   - **Step 1: Define Model Classes**: Identify entities and create classes for them.
   - **Step 2: Define Properties**: Add fields and attributes to each class.
   - **Step 3: Define Behavior**: Implement methods for each class. Start with top-level methods and drill down to discover additional methods as needed.

---

## High-Level Design (HLD) Interviews

High-Level Design focuses on the **architecture and scalability** of a system. It involves designing services, choosing data storage solutions, and ensuring the system can handle high loads and remain fault-tolerant.

### Expectations in HLD Interviews:
1. **Requirement Gathering**: Define system constraints and use cases.
2. **Component Identification**: Identify core components like databases, queues, caches, etc.
3. **Scalability and Availability**: Design for high scalability, fault tolerance, and concurrency control.
4. **Trade-offs**: Justify your design choices and discuss pros and cons.

---

### HLD Learning Strategy

#### 1. **Learn Core Concepts**
   - Study **scalability** techniques like partitioning, sharding, and replication.
   - Understand **high availability** concepts like quorums and leader election.
   - Learn about **backend components**:
     - **Databases**: SQL vs. NoSQL, database schemas, and indexing.
     - **Queues**: Kafka, RabbitMQ, and their use cases.
     - **Caches**: Redis, Memcached, and caching strategies.
     - **Blob Storage**: S3, Google Cloud Storage, etc.

#### 2. **Practice Problems**
   - Solve problems like designing **URL Shorteners**, **Social Media Platforms**, or **E-commerce Systems**.
   - Focus on creating component diagrams and defining high-level flows.

#### 3. **Steps in an HLD Interview**
   - **Step 1: Gather Requirements**: Clarify use cases, constraints, and assumptions.
   - **Step 2: Create a High-Level Design**: Sketch components (e.g., services, databases, caches) and their interactions.
   - **Step 3: Design Core Components**: Dive deep into critical components (e.g., URL hashing, database schemas).
   - **Step 4: Scale the Design**: Address bottlenecks using techniques like load balancing, caching, and sharding.

---

## Combined Roadmap for LLD and HLD Preparation

### 1. **Learn the Fundamentals**
   - **LLD**: Focus on OOP, design principles, and patterns.
   - **HLD**: Study scalability, availability, and backend components.

### 2. **Practice Real-World Problems**
   - **LLD**: Practice designing small systems like **Parking Lots** or **Elevator Systems**.
   - **HLD**: Practice designing large-scale systems like **Instagram**, **Netflix**, or **URL Shorteners**.

### 3. **Understand Trade-offs**
   - Analyze the pros and cons of your designs.
   - Compare your solutions with others to identify improvements.

### 4. **Mock Interviews**
   - Simulate real interview scenarios with peers or mentors.
   - Focus on clear communication and justifying your design choices.

### 5. **Resources**
   - **Books**: *Designing Data-Intensive Applications* by Martin Kleppmann.
   - **Videos**: Watch system design tutorials on platforms like YouTube.
   - **Open Source**: Study the documentation of tools like Kafka, Redis, and databases.

---

## Example Problems for Practice

### Low-Level Design Problems:
1. Design a **Parking Lot System**.
2. Create a **Library Management System**.
3. Design an **Elevator Control System**.
4. Build a **Vending Machine System**.

### High-Level Design Problems:
1. Design **Instagram** or **Twitter**.
2. Create a **URL Shortening Service**.
3. Build a **Distributed Job Scheduler**.
4. Design a **Payment Gateway**.
5. Create a **Netflix-like Streaming Service**.

---

## Another Format for System Design Interviews

You need to understand that there is a flow to the System design interview. Which mostly looks like this:

1. **Requirement clarification** - functional & non-functional
2. **Estimations** - Storage, Bandwidth, etc.
3. **Data flow**
4. **High-level component design**
5. **Detailed design**
6. **Identify and address issues** (System bottlenecks)

### Detailed Interview Steps

### 1. Problem Statement
- Functional Requirements
- Non-Functional Requirements

### 2. Back of the Envelope Estimate
- Estimating Queries Per Second
- Read vs. Write Characteristics
- Query Distribution Ratio
- Breaking Down Query Distribution
- Estimating Data Storage

### 3. API Design
- Define different API endpoints to support the requirements

### 4. Database Design
- Identifying Queries
- Record Size Estimation
- Schema Design
- Sharding Approach
- Selecting the right Database
- Data Partitioning and related problems

### 5. High Level Design
- Create component level design to show what system will look like
- Use tools like [Excalidraw](https://excalidraw.com/) for diagrams

### 6. Technology-Specific Designs

#### Design with Sorted Set Redis
- Read/Write Access Pattern
- Query Pattern

#### Design with Cassandra
- Read/Write Access Pattern
- How token-aware driver works
- Query Pattern

### 7. Maintaining System Reliability
- Address points to support this part of the system

## References

### Basic/Fundamental of System Design
- [System Design Fundamentals: A Complete Guide for Beginners](https://dev.to/kaustubhyerkade/system-design-fundamentals-a-complete-guide-for-beginners-3n95) or [view pdf]({{page.attachment_path}}/system_design_fundamentals_ a_complete_guide_for_ beginners.pdf)
- [System Design Fundamentals (YouTube)](https://www.youtube.com/watch?v=uzeJb7ZjoQ4&t=873s&ab_channel=Exponent)
- [The System Design Primer](https://github.com/donnemartin/system-design-primer?tab=readme-ov-file#the-system-design-primer)

### Roadmap
- [Interview Prep: High Level Design Roadmap](https://medium.com/@sandeep.kumar.ece16/interview-prep-high-level-design-roadmap-9b1218373d3c) or [view pdf]({{page.attachment_path}}/interview_prep _ high_level_design_roadmap_ by_sandeep_kumar.pdf)
- [Low Level Design Roadmap](https://blog.uditagarwal.com/2022-06-25-low-level-design-roadmap/) or [view pdf]({{page.attachment_path}}/roadmap-for-low-level-design-interviews-preparation.pdf)
- [High Level Design Roadmap](https://blog.uditagarwal.com/2022-07-21-high-level-design-roadmap/) or [view pdf]({{page.attachment_path}}/roadmap-for-high-level-system-design-interviews-preparation.pdf)

---

## Final Tips for Success

1. **Ask Questions**: Clarify requirements and constraints before starting.
2. **Think Aloud**: Communicate your thought process clearly.
3. **Iterate**: Start with a basic design and refine it step by step.
4. **Focus on Trade-offs**: No design is perfect; justify your choices.
5. **Practice, Practice, Practice**: The more you practice, the better you’ll get.