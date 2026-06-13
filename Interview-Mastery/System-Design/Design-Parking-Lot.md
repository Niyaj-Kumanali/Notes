# Design Parking Lot

## Overview

- **Definition** — An object-oriented design for a parking lot system that manages floors, spots, entry/exit gates, ticketing, payment processing, and real-time display boards
- **Why It Exists** — Parking lots need automated, concurrent management of spot assignment, fee calculation, and traffic flow to reduce human intervention and maximize utilization
- **Historical Context** — Traditional parking lots used manual ticketing; automated systems emerged with RFID and digital payments in the 2000s; modern lots integrate with mobile apps and dynamic pricing
- **Key Concepts** — **Spot assignment strategy** determines which spot a vehicle takes; **Concurrency control** ensures two gates do not assign the same spot; **Ticket** records entry time for fee calculation; **Display board** shows real-time availability per floor/spot type

## Core Concepts

- ParkingLot is the top-level class managing floors, entry gates, exit gates, and the display board
  - Maintains a list of floors and gates
  - Coordinates spot assignment across entry gates
- Floor contains spots and tracks occupancy per spot size
  - Each floor has a unique ID, a list of spots, and counters for available spots per size
  - Floor also tracks total capacity and current occupancy
- Spot has a unique ID, size category, floor number, and availability status
  - Sizes: SMALL (compact cars), MEDIUM (sedans), LARGE (SUVs, trucks), HANDICAPPED
  - Status: AVAILABLE, OCCUPIED, RESERVED, OUT_OF_ORDER
- Ticket records entry time, spot ID, vehicle info, and exits with total charge
  - Fields: ticketId, vehicleId, spotId, entryTime, exitTime, totalCharge, paymentStatus
  - Fee calculation: hourly rate * hours parked, with minimum charge and daily maximum
- Vehicle has a license plate number and size category
  - Vehicle size determines which spots the vehicle can use
- EntryGate issues tickets and coordinates with ParkingLot for spot assignment
  - Checks availability on the display board before allowing entry
  - Calls ParkingLot.findSpot(vehicleSize) to get an available spot
  - Creates a ticket and opens the gate
- ExitGate processes payment and frees the spot after exit
  - Reads the ticket, calculates the fee, processes payment, frees the spot
  - Updates the display board after successful exit
- PaymentProcessor handles multiple payment types
  - Cash, credit card, UPI, mobile wallet
  - Each payment type has its own processing logic (auth, capture, refund)
  - Supports hourly rate lookup with possible peak/off-peak pricing
- Spot assignment strategy
  - Nearest to elevator/entrance minimizes walking distance for customers
  - Fill small spots before large ones optimizes large spot availability for bigger vehicles
  - Even distribution across floors prevents congestion at one floor
- Concurrency requires locks or atomic operations on spot availability
  - Multiple gates operating simultaneously can race to assign the same spot
  - Use database transactions with row-level locks, or distributed locks (Redis Redlock) for multi-server deployments
- Display board shows available spots per floor, per size, and total
  - Updated on every entry and exit event
  - Can be a Singleton since there is typically one board per lot
- When the lot is full, the display board shows zero availability and entry gates deny new vehicles
  - Optionally, a queue/waiting list can be maintained for peak hours

## Common Mistakes

- **Race condition on spot assignment with multiple entry gates**
  - Two entry gates read the same spot as available and both assign it, causing double-booking
  - **Why it looks correct:** Checking spot availability before assigning seems sufficient in single-threaded code
  - Use atomic operations: SELECT ... FOR UPDATE in SQL, or optimistic locking with version number. In distributed systems, use a Redis lock per spot or a distributed transaction coordinator. The assignment and status update must be in a single atomic operation.
- **Using monolithic fee calculation without considering edge cases**
  - Simple hourly rate multiplied by hours fails for partial hours, overnight parking, lost tickets, or peak pricing
  - **Why it looks correct:** Hourly rate * hours is the most straightforward formula
  - Implement a FeeCalculationStrategy interface with pluggable rules: hourly rate with minute granularity, daily maximum, grace period, lost ticket penalty, promotional discounts. Test with edge cases (1 minute, 23 hours, 7 days, overnight).
- **Not decoupling display board updates from gate operations**
  - The display board is updated synchronously during entry/exit, slowing down gate operations if the board service is slow
  - **Why it looks correct:** The board should reflect real-time data, so updating inline seems natural
  - Use an event-driven approach: gates publish entry/exit events to a message queue; a display board consumer reads events and updates the board asynchronously. The board can tolerate a few seconds of latency.

## Real-World Scenarios

### Multi-Story Mall Parking Lot

- 5 floors, 200 spots per floor (small, medium, large, handicapped)
- 4 entry gates, 4 exit gates (can be bidirectional)
- Display boards at each floor entrance and at the street entrance
- Peak hours (weekends, holidays) require queue management

### Airport Parking Garage

- Long-term and short-term parking zones with different rates
- Payment at kiosk before returning to car or at exit gate
- Lost ticket handling with verification (license plate lookup)

## Scenario-Based Questions

**Q: During peak hours, 3 cars arrive at 3 entry gates simultaneously. All 3 gates see the same spot as available and assign it. How do you fix this?**

- Use a database transaction with SELECT ... FOR UPDATE on the spot row — the second and third transactions wait for the first to complete, then see the spot as occupied
- In a distributed system, use a distributed lock per spot (or per floor) with Redis Redlock — the gate acquires the lock before reading and updating spot status
- Alternatively, use optimistic locking with a version column: UPDATE spots SET status='OCCUPIED', version=version+1 WHERE spot_id=X AND version=Y; if affected rows = 0, another gate took it, retry with the next available spot
- **Interview follow-up:** How would you handle the scenario where a gate acquires the lock but the vehicle never enters (e.g., drives away)?

**Q: The parking lot display board shows 5 available spots, but when a car arrives, all spots are actually occupied. What went wrong?**

- The display board is not updated synchronously with spot assignments — likely updated via a delayed batch process
- An exit occurred but the board update failed silently, showing stale data
- Fix: use event-driven architecture with at-least-once delivery. The gate publishes an event on entry/exit; the board consumer processes the event and updates availability. Add reconciliation: periodically scan actual spot occupancy and correct the board display.
- **Interview follow-up:** If you cannot fix the synchronicity, should the board show slightly stale data (optimistic) or always under-report (pessimistic)?

## Interview Questions

- **Design the classes for a parking lot system.**
  - ParkingLot (floors, gates, display board, findSpot()), Floor (id, spots, availableCountBySize), Spot (id, size, status), Ticket (id, vehicleId, spotId, entryTime, exitTime, charge), Vehicle (licensePlate, size), EntryGate (assignSpot, issueTicket), ExitGate (processPayment, freeSpot), PaymentProcessor (processPayment by type), DisplayBoard (update, show), SpotAssignmentStrategy (interface: findSpot).
- **How do you handle concurrency in parking lot spot assignment?**
  - Use pessimistic locking (SELECT ... FOR UPDATE) in relational databases to prevent two gates from reading the same status. In distributed environments, use optimistic locking (version column, retry on conflict) or distributed locks (Redis). The key is that spot status read and update must be atomic.
- **Explain the strategy pattern for spot assignment.**
  - Define a SpotAssignmentStrategy interface with a method findSpot(vehicleSize, floors) -> Spot. Implement NearestStrategy (spot closest to elevator), UtilizationStrategy (fill small spots first), DistributionStrategy (even floor distribution). The ParkingLot accepts a strategy at construction time, allowing runtime swapping.
- **How would you calculate parking fees for edge cases?**
  - Use a FeeCalculationStrategy with rules: base hourly rate, partial hour rounding (up or prorated), daily maximum cap, overnight flat rate, grace period (15 minutes free), lost ticket penalty, peak/off-peak multipliers. Calculate with minute granularity, not hourly boundaries.

## Developer Recommendations

- **Use atomic spot assignment with pessimistic locking**
  - The spot availability check and status update must be a single atomic operation
  - In relational DB: BEGIN, SELECT ... FOR UPDATE, UPDATE, COMMIT
  - In distributed: distributed lock (Redis or ZooKeeper) around the assignment logic
  - **Production story:** A parking lot system double-booked 12 spots in one day because two entry gates read the same spot as free in a race condition — pessimistic locking eliminated the issue
- **Decouple display board updates from gate operations via events**
  - Publish entry/exit events to a queue; the display board consumer processes events asynchronously
  - Use at-least-once delivery semantics with idempotent processing to handle duplicates
  - **Production story:** Synchronous display updates added 200ms to each gate operation during peak hours — moving to async event processing reduced gate latency to 50ms
- **Implement pluggable fee calculation strategies**
  - Different parking lots have different pricing (hourly, daily, event-based, valet)
  - Use strategy pattern: define common interface, inject the appropriate strategy at configuration time
- **Design for bidirectional gates from day one**
  - A single gate may serve as both entry and exit during different times of day
  - Gate class should support both modes with clear state management
