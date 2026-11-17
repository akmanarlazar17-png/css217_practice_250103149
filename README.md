# Facade Pattern Lab — Airline Flight Booking

A Java implementation of the **Facade Design Pattern** applied to a five-class
airline booking subsystem. The project hides seat inventory, baggage pricing,
currency conversion, payment processing, and ticket issuance behind a single
`FlightBookingFacade` entry point, with automatic seat-lock rollback and
guaranteed resource cleanup.

---

## Table of Contents

1. [Overview](#overview)
2. [Pattern Rationale](#pattern-rationale)
3. [Project Structure](#project-structure)
4. [Prerequisites](#prerequisites)
5. [Build & Run](#build--run)
6. [Usage](#usage)
7. [Public API Reference](#public-api-reference)
8. [Architecture](#architecture)
9. [Loyalty Tier Rules (Phase 2)](#loyalty-tier-rules-phase-2)
10. [Test Scenarios](#test-scenarios)
11. [Expected Output](#expected-output)
12. [Design Notes](#design-notes)
13. [Verification Checklist](#verification-checklist)
14. [Known Subsystem Quirks](#known-subsystem-quirks)
15. [License](#license)

---

## Overview

Booking a flight ticket is one high-level business action that requires
coordinating five independent subsystems, each with its own initialisation
requirement, validation rule, and execution order. Calling them directly from
client code produces tight coupling and fragile, order-sensitive logic spread
across the application.

This project demonstrates how a **Facade** wraps that orchestration into a
single method:

```java
String ticket = facade.bookFlight(
    "Amina Suleimenova", "KC101", "14A",
    320.00, 2, 23.0, "EUR", "4532015112830366");
 Objective

A five-class airline booking subsystem (SeatInventorySystem, BaggagePricingEngine, CurrencyConverter, PaymentProcessor, TicketIssuer) each carries its own initialisation requirement, validation rule, and ordering constraint. The goal of this lab was to introduce a single FlightBookingFacade entry point that:

Hides all five subsystems from the client,
Owns the correct call ordering and resource lifecycle,
Rolls back partially-completed work when a later step fails,
Absorbs a mid-project specification change (Phase 2 loyalty tiers) without altering the client or any subsystem.
2. Background — the problem the pattern solves

Calling the subsystems directly from a client produces three distinct coupling problems:

Ordering coupling — SeatInventorySystem throws unless connectToReservationServer() runs first; PaymentProcessor throws unless initializeGateway() runs first.
Resource coupling — the seat server connection is a real resource that must be closed on every path, including failure.
Failure coupling — a seat lock acquired before a later failure (e.g. bad card number) is stranded unless somebody explicitly calls releaseSeatLock(...).
The Facade pattern addresses all three: the client learns one method, one return type, and nothing else. The five subsystems remain independently testable and unmodified.

3. Subsystem analysis (pre-implementation)

Before writing any facade code, each subsystem was inspected for its hidden contract.

Subsystem	Precondition enforced	Failure type	Side effect to manage
SeatInventorySystem	connectToReservationServer() before any query/lock	IllegalArgumentException	Connection must be closed; row 13 blocked
BaggagePricingEngine	none (stateless)	IllegalArgumentException (negative input)	none
CurrencyConverter	supported currency code only	IllegalArgumentException (unsupported / null)	none
PaymentProcessor	initializeGateway() before applyPromoCode / chargeCard	IllegalArgumentException	Promo reset after each charge; card must be ≥ 12 digits
TicketIssuer	non-null lock token and receipt id	IllegalStateException	none
Derived ordering pipeline: connect → lock → price baggage → convert → init gateway → (apply promo) → charge → issue ticket → release/disconnect.
