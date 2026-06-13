# Design Parking Lot

- Requirements: multiple floors, spots of different sizes (small, medium, large), entry and exit gates, ticketing, payment, hourly rate, display board showing available spots
- ParkingLot manages floors, entry gates, exit gates, and the display board
- Floor contains spots and tracks occupancy per spot size
- Spot has a unique ID, size, floor number, and availability status
- Ticket records entry time, spot ID, and exits with total charge calculated
- Vehicle has a license plate number and size category
- EntryGate issues tickets and coordinates with ParkingLot for spot assignment
- ExitGate processes payment and frees the spot after exit
- PaymentProcessor handles cash, card, and UPI transactions
- Spot assignment strategy can be nearest to elevator to minimize walking distance
- Assignment strategy can also optimize utilization by filling small spots before large ones
- Concurrency requires locks or atomic operations on spot availability when multiple gates operate simultaneously
- When the lot is full, the display board shows zero availability and entry gates deny new vehicles
- Hourly rate varies by spot size and duration
