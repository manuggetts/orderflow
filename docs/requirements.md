# OrderFlow — Requirements Specification

**Version:** 1.0  
**Status:** Draft  
**Project:** OrderFlow

## 1. Project Overview

OrderFlow is a distributed order processing platform designed to demonstrate Event-Driven Architecture (EDA), microservices, asynchronous communication and distributed system reliability.

The platform simulates an e-commerce order processing workflow, including inventory reservation, payment processing, order confirmation and cancellation.

## 2. Project Objectives

- Implement an event-driven architecture using Apache Kafka.
- Develop independent microservices using Java and Spring Boot.
- Explore asynchronous communication between services.
- Implement distributed transaction patterns.
- Handle message duplication and processing failures.
- Apply automated testing and observability practices.

## 3. Scope

### 3.1 Included in MVP

- Order creation and consultation through REST API.
- Inventory availability validation and reservation.
- Payment processing simulation.
- Automatic order confirmation or cancellation.
- Automatic inventory release after payment rejection.
- Asynchronous communication through Kafka.
- Message idempotency and failure handling.

### 3.2 Out of Scope

- Graphical user interface.
- User authentication and authorization.
- Real payment gateway integration.
- Real email delivery.
- Cloud deployment.

## 4. System Actors

| Actor | Responsibility |
|---|---|
| API Consumer | Creates orders and checks their status. |
| Order Service | Manages order lifecycle. |
| Inventory Service | Controls inventory and reservations. |
| Payment Service | Simulates payment processing. |
| Notification Service | Handles order status notifications. |

## 5. Functional Requirements

| ID | Requirement |
|---|---|
| RF01 | The system must allow order creation through REST API. |
| RF02 | The system must allow order status consultation. |
| RF03 | The system must validate inventory availability. |
| RF04 | The system must reserve available inventory. |
| RF05 | The system must simulate payment processing. |
| RF06 | The system must confirm orders after successful payment. |
| RF07 | The system must cancel orders when processing fails. |
| RF08 | The system must automatically release reserved inventory after payment rejection. |
| RF09 | The system must publish and consume domain events through Kafka. |
| RF10 | The system must notify customers about order status changes. |

## 6. Non-Functional Requirements

| ID | Requirement |
|---|---|
| RNF01 | Inter-service communication must use asynchronous messaging for the main business workflow. |
| RNF02 | Services must maintain independent responsibilities and persistence. |
| RNF03 | Event processing must be idempotent. |
| RNF04 | The system must support retry and dead-letter handling. |
| RNF05 | The system must provide automated tests. |
| RNF06 | The development environment must run locally using Docker. |
| RNF07 | Critical distributed operations must have documented consistency guarantees. |

## 7. Business Rules

| ID | Rule |
|---|---|
| BR01 | Every new order starts with PENDING status. |
| BR02 | An order can only proceed to payment after inventory reservation succeeds. |
| BR03 | An order is CONFIRMED only after payment approval. |
| BR04 | An order must be CANCELLED if inventory reservation fails. |
| BR05 | An order must be CANCELLED if payment is rejected. |
| BR06 | Reserved inventory must be automatically released after payment rejection.