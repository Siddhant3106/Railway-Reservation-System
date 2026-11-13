# Railway Reservation System

## Objective
This project is a railway reservation system. The main goal is to simulate how a real train booking system works: a user creates an account, searches for trains between stations, checks seat availability, books tickets, and later views or downloads their reservations.

---

## What this system does
A typical user flow is:

1. Register an account
2. Log in with username and password
3. Select source and destination stations
4. Search for available trains on a selected date
5. Choose train and ticket type (AC or General)
6. Enter passenger details
7. Pay and confirm booking
8. Generate a ticket and save the booking in the user account
9. View past and upcoming bookings

This is very similar to a real-world railway booking platform where booking is based on seat count, route selection, date logic, and user-specific history.

---

## Implementation
From a database viewpoint, the project is a relational reservation system. The important idea is that booking information is stored in multiple related tables instead of a single flat table.

The system keeps information in normalized tables so that:

- users can have multiple tickets
- one ticket can have multiple passengers
- one train can have multiple bookings
- one reservation is linked to a train, ticket, and user
- ticket history can be retrieved using joins

This is the same pattern used in real reservation systems.

---

## Main schema / tables
The application is built around several key tables. Although the exact table names in the code are used directly by the app, the conceptual schema is as follows.

### 1. Login
Stores user login information.

Fields conceptually include:

- UserId
- Pwd
- Mobile_no
- Email
- User_Status

Purpose:

- user authentication
- registration validation
- login session tracking

SQL example:

```sql
SELECT Pwd
FROM Login
WHERE UserId = %s;
```

and

```sql
INSERT INTO Login (UserId, Pwd, Mobile_no, Email)
VALUES (%s, %s, %s, %s);
```

### 2. Stations
Stores station records and route order.

Fields conceptually include:

- station_name
- order_in_route

Purpose:

- determine whether a route is valid
- compare source and destination order
- calculate travel direction

This table is essential because the application checks whether a selected source and destination lie on a valid train route and in a valid direction.

### 3. Trains
Stores train master records.

Fields conceptually include:

- train_number
- train_name
- src
- destination
- price_per_ticket

Purpose:

- identify each train
- associate a train with its route and fare
- support route-based searching

### 4. Train availability
Stores seat availability for each train by day of week.

Fields conceptually include:

- train_number
- day_of_week
- available_ac_seats
- available_gen_seats
- version_number

Purpose:

- determine how many seats remain for a journey date
- handle class-wise availability (AC/General)
- support concurrency-safe booking logic

This is one of the most important tables in the system because it controls whether a user can book a ticket.

### 5. Passengers
Stores passenger data for each traveler.

Fields conceptually include:

- passenger_name
- age
- mobile_no
- adhaar_number
- sex

Purpose:

- capture traveler identity
- connect passengers to reservations
- allow one ticket to include multiple passengers

### 6. Tickets
Stores ticket-level reservation data.

Fields conceptually include:

- ticket_ID
- ticket_cluster
- ticket_type
- confirmation_status
- departure_date
- arrival_date
- departure_time
- arrival_time
- ticket_source
- ticket_destination
- coach_no
- berth_no
- amount

Purpose:

- represent each booked seat as a ticket
- store travel schedule and class
- group multiple passengers under a ticket cluster
- generate printable ticket details

### 7. Booking
Links passengers to tickets.

Fields conceptually include:

- adhaar_number
- ticket_ID

Purpose:

- many passengers can book under one ticket cluster
- each booking can be traced back to a passenger

### 8. Reservation
Links tickets to trains.

Fields conceptually include:

- ticket_ID
- train_number

Purpose:

- identify which train was booked for a ticket
- keep train-specific booking history

### 9. MyTickets
Links users to tickets.

Fields conceptually include:

- UserId
- ticket_ID

Purpose:

- maintain each user’s ticket history
- show current and past bookings on the user dashboard

---

## Relationship model
The database is designed with relationships like this:

- One user can have many tickets
- One ticket can belong to many passengers
- One ticket belongs to one train
- One train can have many ticket reservations
- One passenger can be linked to multiple tickets over time

This is a classic relational database design for a reservation system.

The application uses joins such as:

```sql
SELECT DISTINCT tr.train_number, tr.train_name, t.ticket_cluster, t.ticket_type,
                t.departure_date, t.arrival_date, t.departure_time, t.arrival_time,
                t.ticket_source, t.ticket_destination
FROM MyTickets mt
JOIN tickets t ON mt.ticket_ID = t.ticket_ID
JOIN booking b ON b.ticket_ID = t.ticket_ID
JOIN reservation r ON r.ticket_ID = t.ticket_ID
JOIN trains tr ON tr.train_number = r.train_number
JOIN passengers p ON p.adhaar_number = b.adhaar_number
WHERE mt.UserId = %s;
```

This query reconstructs a user’s booking history from multiple related tables.

---

## Process flow of the system

### 1. Registration process
The system checks if the username or email already exists and if the values are unique, the system inserts the user into the `Login` table.

This is the registration layer of the system.

### 2. Login process
The system verifies the user by matching the entered password with the stored value. If the password matches, a session is created and the user is redirected to the booking section.

### 3. Route search process
The user chooses a source and destination station and journey date. The app then searches trains that match the route and date.

The system uses station ordering to decide whether a train passes through the selected stations in the correct direction. The code also compares `order_in_route` values to decide whether the train can connect the selected route.

### 4. Availability check
When the user chooses a train, the app checks whether the train has seats on that day for the selected class.

This is important because the table is checked for available seats before booking is confirmed.

### 5. Booking creation
When the user confirms the booking, the system inserts passenger details and then creates the reservation.

The app performs a seat update and also assigns a coach and berth number based on the remaining seat inventory.

Then it creates the ticket record after which, the related rows are stored in `MyTickets`, `booking`, and `reservation`.

---

## Transactions and consistency
This system uses database transactions to ensure that a booking is either fully successful or fully rejected.

When a booking is made, the code ensures that the system does not leave behind partial data when a failure happens.

For example, if an error occurs after reducing seats but before inserting the ticket record, the rollback prevents an inconsistent booking state.

---

## Ticket generation and history
Once the booking is confirmed, the system stores the data needed to print or display the final ticket.

The ticket page uses queries like:

```sql
SELECT p.passenger_name, p.age, p.sex, t.coach_no, t.berth_no, t.ticket_ID
FROM tickets t
JOIN booking b ON b.ticket_ID = t.ticket_ID
JOIN passengers p ON p.adhaar_number = b.adhaar_number
WHERE ticket_cluster = %s;
```

This retrieves the passengers attached to the same ticket cluster, which allows the app to generate a ticket with multiple travelers.

The user’s booking list is pulled from `MyTickets` and related tables so that both current and past bookings can be displayed.

---


