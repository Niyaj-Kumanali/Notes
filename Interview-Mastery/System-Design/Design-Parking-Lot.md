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

## Use Cases

- **Multi-level parking garage management** — commercial parking structures with multiple floors, entry/exit gates, and payment kiosks
  - System tracks spot occupancy in real-time. Displays available spots per floor at entrance. Handles peak-hour traffic with queue management at gates.
  - **Avoid when:** parking is unstructured (open lots, no gates) — a simple ticket-based system without spot-level tracking is cheaper.

- **Airport parking** — long-term and short-term zones with different rates, lost ticket handling, and valet services
  - Zone-specific pricing and availability. License plate lookup for lost tickets. Reservation system for guaranteed spots during holiday seasons.
  - **Avoid when:** the lot is small (<50 spots) — manual tracking with a whiteboard is simpler and more cost-effective.

- **Event parking** — concert venues, stadiums, or convention centers with pre-paid reservations
  - Pre-booking with time slots. Variable pricing based on event proximity. QR code validation at entry gates. Overflow lot management for sell-out events.
  - **Avoid when:** parking is free and first-come-first-served — a simple count of available spots is sufficient.

- **Residential/commercial building parking** — apartment complexes or office buildings with assigned spots and visitor parking
  - Tenant spot assignments with vehicle registration. Visitor parking with time limits. Guest pass generation via web/mobile app.
  - **Avoid when:** parking is unassigned and first-come — a simple permit system without digital assignment is cheaper.

- **Automated valet parking systems** — robotic parking where a machine parks the car in a dense storage grid
  - System tracks exact car location in the grid. Optimizes placement for retrieval time. Handles staging area for drop-off and pick-up.
  - **Avoid when:** the cost of automation exceeds the land value — traditional parking is more economical for most locations.

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

**Q: Your parking lot charges different rates for different hours (peak: $5/hr, off-peak: $2/hr). A car enters at 3:50 PM and exits at 4:10 PM during a peak-to-off-peak transition. How do you calculate the fee?**

- Prorate by actual time spent in each rate period: 10 minutes at peak rate + 10 minutes at off-peak rate
- Use the rate at entry time for the entire stay (simpler but may cause disputes at rate boundaries)
- Store rate periods as overlapping or adjacent intervals with a rule engine that evaluates each minute of parking
- **Interview follow-up:** How would you handle multiple rate changes (e.g., weekday vs weekend, holiday surcharges)?

**Q: A driver loses their parking ticket. How does the system handle lost ticket scenarios?**

- Calculate the maximum possible fee: charge the maximum daily rate for the entire day the car could have been parked
- Look up the vehicle by license plate from entry cameras — if the vehicle is found in the system, use the actual entry time
- Require manual verification at a staffed booth for lost tickets with identity verification
- **Interview follow-up:** How do you prevent fraud where a driver claims a lost ticket but actually entered recently to pay a lower fee?

**Q: Your parking lot has monthly reserved spots that should always be available for specific users. During peak hours, regular customers park in reserved spots. How do you enforce reservation?**

- Mark reserved spots with a RESERVED status in the system — the spot assignment algorithm skips reserved spots for non-reserved vehicles
- Implement license plate recognition at entry: if a reserved spot holder arrives, their spot is guaranteed; if a non-reserved vehicle parks there, issue a ticket
- Use physical barriers (retractable bollards or gates) for reserved spots that only authorized users can lower via RFID or app
- **Interview follow-up:** How do you handle the case where a reserved spot holder does not show up — should the spot be released to general parking after a grace period?

**Q: Your parking lot uses a payment kiosk at the exit. During peak hours, the exit line backs up because each payment takes 30 seconds. How do you reduce exit congestion?**

- Implement prepayment kiosks inside the parking lot: customers pay before returning to their car, then scan the receipt at exit for a quick check
- Support automatic payment via license plate recognition: the camera reads the plate at exit, charges the linked account, and opens the gate
- Use a mobile app for contactless payment — customers pay through the app and scan a QR code at exit
- **Interview follow-up:** How do you handle the case where a customer's automatic payment fails (insufficient funds, expired card)?

**Q: Your parking lot has valet parking. How does the valet system integrate with the parking lot design?**

- Valet mode: the attendant checks in the vehicle, the system assigns a spot, the attendant parks the vehicle, and the ticket is stored centrally
- When the customer returns, the attendant retrieves the vehicle using the ticket ID, and the system processes payment
- Valet spots are typically located in a dedicated area near the entrance for quick turnaround
- **Interview follow-up:** How do you track which valet attendant parked which car for accountability?

**Q: Your parking lot uses a sensor per spot to detect occupancy. Some sensors malfunction, reporting occupied spots as empty or vice versa. How do you handle sensor errors?**

- Implement sensor health monitoring: if a sensor reports rapid state changes or no changes for an extended period, flag it for maintenance
- Cross-validate sensor data with entry/exit gate counts — if 100 cars entered and 95 exited, approximately 5 spots should be occupied
- When a sensor reports a spot as empty but the assignment system shows it as occupied, trust the assignment system and flag the sensor for inspection
- **Interview follow-up:** How do you handle the case where a car is parked straddling two spots, confusing both sensors?

**Q: Your parking lot supports electric vehicle (EV) charging stations. How do you integrate charging into the parking system?**

- Designate specific spots with EV charging and mark them in the spot management system
- Track charging station usage: when a vehicle parks at an EV spot, start a charging session; the fee includes both parking and charging costs
- Implement a policy: EV spots have a grace period after charging completes — if the vehicle remains past the grace period, apply a surcharge to encourage turnover
- **Interview follow-up:** How do you handle the case where a non-EV vehicle parks in an EV charging spot?

**Q: Your parking lot system needs to support dynamic pricing based on real-time demand. When occupancy exceeds 80%, rates should increase to discourage entry. How do you implement this?**

- Implement a dynamic pricing service that reads current occupancy from the display board and adjusts rates in real time
- The rate schedule is evaluated at entry time: if occupancy > 80%, apply a surge multiplier to the base rate
- Display the current rate on the entry board so drivers can decide whether to enter based on price
- **Interview follow-up:** How do you prevent rapid rate fluctuations that confuse customers — should there be a minimum time between rate changes?

## Interview Questions

- **Design the classes for a parking lot system.**
  - ParkingLot (floors, gates, display board, findSpot()), Floor (id, spots, availableCountBySize), Spot (id, size, status), Ticket (id, vehicleId, spotId, entryTime, exitTime, charge), Vehicle (licensePlate, size), EntryGate (assignSpot, issueTicket), ExitGate (processPayment, freeSpot), PaymentProcessor (processPayment by type), DisplayBoard (update, show), SpotAssignmentStrategy (interface: findSpot).
- **How do you handle concurrency in parking lot spot assignment?**
  - Use pessimistic locking (SELECT ... FOR UPDATE) in relational databases to prevent two gates from reading the same status. In distributed environments, use optimistic locking (version column, retry on conflict) or distributed locks (Redis). The key is that spot status read and update must be atomic.
- **Explain the strategy pattern for spot assignment.**
  - Define a SpotAssignmentStrategy interface with a method findSpot(vehicleSize, floors) -> Spot. Implement NearestStrategy (spot closest to elevator), UtilizationStrategy (fill small spots first), DistributionStrategy (even floor distribution). The ParkingLot accepts a strategy at construction time, allowing runtime swapping.
- **How would you calculate parking fees for edge cases?**
  - Use a FeeCalculationStrategy with rules: base hourly rate, partial hour rounding (up or prorated), daily maximum cap, overnight flat rate, grace period (15 minutes free), lost ticket penalty, peak/off-peak multipliers. Calculate with minute granularity, not hourly boundaries.
- **How do you design the parking lot system to handle multiple entry and exit gates?**
  - Each gate operates independently and communicates with a central ParkingLot coordinator via a shared database or distributed lock. Entry gates call findSpot() atomically. Exit gates free the spot and update the display board. The coordinator ensures consistency across gates using transactions or optimistic locking.
- **Explain the role of a "display board" in the parking lot system.**
  - The display board shows real-time availability per floor, per spot type, and total. It helps drivers decide which floor to use before entering. It is updated on every entry and exit event, ideally through an event-driven async pipeline to avoid blocking gate operations.
- **How does the strategy pattern apply to spot assignment?**
  - The SpotAssignmentStrategy interface defines a method findSpot(vehicle, floors) → Spot. Implementations include NearestToElevatorStrategy, FillSmallestFirstStrategy, EvenDistributionStrategy. The ParkingLot is configured with a strategy at startup and can swap strategies at runtime (e.g., switch to even distribution during peak hours).
- **How do you implement a waiting list for a full parking lot?**
  - When the lot is full, offer the driver an option to join a queue. Store the queue in Redis (sorted set by timestamp). When a spot becomes available, dequeue the next driver and send a notification (SMS, push). The spot is reserved for a limited time (e.g., 5 minutes) before it is released to the next in queue.
- **What design patterns are used in a parking lot system?**
  - Singleton: DisplayBoard (typically one per lot). Strategy: SpotAssignmentStrategy and FeeCalculationStrategy. Observer: display board observes entry/exit events. Factory: GateFactory creates EntryGate or ExitGate. Repository: SpotRepository for database access. Command: gate operations as command objects for audit logging.
- **How do you handle payment failures at the exit gate?**
  - Retry the payment up to 3 times. If all retries fail, open the gate but flag the ticket for follow-up (invoice the customer later via their registered payment method). For cash payments, if the exact change is unavailable, round down to the nearest available amount and log the discrepancy.
- **How do you implement seasonal or event-based pricing?**
  - Use a FeeCalculationStrategy that accepts a rate schedule object. The schedule defines base rates, peak multipliers, and special event overrides. The rate for a given time is looked up from the schedule during fee calculation. Changes to the schedule are effective immediately without code deployment.
- **How do you handle the scenario where a vehicle exits without paying?**
  - License plate cameras capture the plate at entry and exit. The system flags unpaid exits and adds the plate to a "blocklist" for future entry. A reconciliation job runs daily to match exit events with payments and generates invoices for unpaid stays.
- **What is the role of "audit logging" in a parking lot system?**
  - Every entry, exit, payment, and manual override is logged with timestamp, operator ID (for manual actions), and full state before/after. Audit logs are essential for dispute resolution (a customer claims they paid but the system disagrees) and fraud detection.
- **How do you design the database schema for a parking lot?**
  - Tables: parking_lot (id, name, address), floor (id, lot_id, floor_number), spot (id, floor_id, spot_number, size, status), ticket (id, spot_id, vehicle_plate, entry_time, exit_time, total_charge, payment_status), payment (id, ticket_id, amount, method, timestamp), gate (id, lot_id, type, status). Indexes on spot.status, ticket.entry_time, ticket.vehicle_plate.
- **How do you handle multiple currency and payment method support?**
  - Store prices in a base currency (USD) and convert at the current exchange rate at payment time. PaymentProcessor interface supports multiple implementations: CashPayment, CreditCardPayment, UPIPayment, MobileWalletPayment. Each implementation handles its own auth, capture, and refund logic.
- **How do you test a parking lot system for race conditions?**
  - Write integration tests that simulate multiple concurrent entry/exit operations. Use thread-safe test harnesses that dispatch vehicles to multiple gates simultaneously. Verify that no spot is double-booked and that total occupancy matches entry/exit counts. Test with pessimistic locking enabled.
- **Explain the difference between "hard reservation" and "soft reservation" for spots.**
  - Hard reservation: a spot is guaranteed for a specific user and time slot; the system will not assign it to anyone else. Soft reservation: the system predicts availability but does not guarantee a specific spot; if all spots fill, the user is queued. Hard is better for reserved parking; soft is better for general parking.
- **How do you implement a "frequent parker" loyalty program?**
  - Track user parking sessions by license plate or account ID. Accumulate points per dollar spent. Points can be redeemed for free parking hours or discounts. Implement a PricingStrategy decorator that applies discounts based on loyalty tier. Points expire after 12 months of inactivity.
- **How would you design the system to support multiple parking lots across a city?**
  - Add a ParkingLotManagementSystem that aggregates data from all lots. Each lot operates independently but reports occupancy and availability to a central service. A mobile app queries the central service to show available spots at all lots. Cross-lot reservations allow users to book at any lot through a unified interface.
- **How does the Observer pattern apply to the display board system?**
  - The display board acts as an observer of entry/exit events. When a gate processes an entry or exit, it publishes an event (subject notifies observers). The display board receives the event and recalculates availability per floor. This decouples the board from gate logic and allows multiple boards (entrance, per-floor) to react to the same events independently.

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
