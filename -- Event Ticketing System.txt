-- Event Ticketing System

CREATE DATABASE event_ticketing_db;
USE event_ticketing_db;

-- 1. Creare tabele
CREATE TABLE buyers (
    buyer_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    registered_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO buyers (name, email, phone) VALUES
('Ion Popescu', 'ion.popescu@example.com', '0722345678'),
('Maria Ionescu', 'maria.ionescu@example.com', '0745678901'),
('Andrei Vasilescu', 'andrei.vasilescu@example.com', '0756345678');

CREATE TABLE venues (
    venue_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    location VARCHAR(150) NOT NULL,
    capacity INT CHECK (capacity > 0)
);
INSERT INTO venues (name, location, capacity) VALUES
('Arena Națională', 'București, România', 55000),
('Sala Palatului', 'București, România', 2500),
('Teatrul Național', 'Cluj-Napoca, România', 1500);

CREATE TABLE events (
    event_id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(100) NOT NULL,
    description TEXT,
    date DATE NOT NULL,
    time TIME,
    venue_id INT NOT NULL,
    total_sales DECIMAL(10,2) DEFAULT 0,
    FOREIGN KEY (venue_id) REFERENCES venues(venue_id)
);
INSERT INTO events (title, description, date, time, venue_id) VALUES
('Concert Coldplay', 'Concert extraordinar al trupei Coldplay.', '2025-06-20', '19:00', 1),
('Spectacol de teatru', 'Piesa de teatru "O viață de om".', '2025-07-05', '18:30', 3),
('Conferința Tech', 'Conferință de tehnologie la Sala Palatului.', '2025-06-15', '09:00', 2);

CREATE TABLE tickets (
    ticket_id INT PRIMARY KEY AUTO_INCREMENT,
    event_id INT NOT NULL,
    seat_number VARCHAR(10),
    category ENUM('Standard', 'VIP', 'Student') DEFAULT 'Standard',
    price DECIMAL(10,2) NOT NULL,
    available BOOLEAN DEFAULT TRUE,
    FOREIGN KEY (event_id) REFERENCES events(event_id)
);
INSERT INTO tickets (event_id, seat_number, category, price) VALUES
(1, 'A1', 'VIP', 300.00),
(1, 'A2', 'Standard', 150.00),
(2, 'B1', 'Standard', 80.00),
(3, 'C1', 'Student', 50.00),
(3, 'C2', 'VIP', 200.00);

-- Tabel intermediar pentru relatia M:M intre buyers si tickets
CREATE TABLE buyer_tickets (
    buyer_id INT,
    ticket_id INT,
    purchase_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (buyer_id, ticket_id),
    FOREIGN KEY (buyer_id) REFERENCES buyers(buyer_id),
    FOREIGN KEY (ticket_id) REFERENCES tickets(ticket_id)
);
INSERT INTO buyer_tickets (buyer_id, ticket_id, purchase_date) VALUES
(1, 1, '2025-05-10 15:30'),
(2, 3, '2025-05-11 16:00'),
(3, 5, '2025-05-12 14:00');

CREATE TABLE payments (
    payment_id INT PRIMARY KEY AUTO_INCREMENT,
    buyer_id INT,
    amount DECIMAL(10,2) NOT NULL,
    method ENUM('Card', 'Cash', 'Online') DEFAULT 'Card',
    payment_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (buyer_id) REFERENCES buyers(buyer_id)
);
INSERT INTO payments (buyer_id, amount, method, payment_date) VALUES
(1, 300.00, 'Card', '2025-05-10 15:35'),
(2, 80.00, 'Cash', '2025-05-11 16:05'),
(3, 200.00, 'Online', '2025-05-12 14:05');

CREATE TABLE venue_details (
    id INT PRIMARY KEY AUTO_INCREMENT,
    venue_id INT,
    venue_name VARCHAR(100),
    venue_location VARCHAR(150)
);

-- Inserarea datelor în tabela 'venue_details'
INSERT INTO venue_details (venue_id, venue_name, venue_location) VALUES
(1, 'Arena Națională', 'București, România'),
(2, 'Sala Palatului', 'București, România'),
(3, 'Teatrul Național', 'Cluj-Napoca, România');

-- Tabele adăugate intenționat ne-normalizate complet
CREATE TABLE event_summary (
    id INT PRIMARY KEY AUTO_INCREMENT,
    event_id INT,
    event_title VARCHAR(100),
    venue_name VARCHAR(100),
    venue_location VARCHAR(150),
    tickets_sold INT,
    total_income DECIMAL(10,2)
);
INSERT INTO event_summary (event_id, venue_name, venue_location, tickets_sold, total_income) VALUES
(1, 'Arena Națională', 'București, România', 1, 300.00),
(2, 'Teatrul Național', 'Cluj-Napoca, România', 1, 80.00),
(3, 'Sala Palatului', 'București, România', 1, 200.00);

CREATE TABLE buyer_contacts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    buyer_id INT,
    full_contact VARCHAR(255)
);

CREATE TABLE ticket_stats (
    id INT PRIMARY KEY AUTO_INCREMENT,
    ticket_id INT,
    event_title VARCHAR(100),
    buyer_name VARCHAR(100),
    category ENUM('Standard', 'VIP', 'Student'),
    price DECIMAL(10,2)
);
INSERT INTO ticket_stats (ticket_id, category, price) VALUES
(1, 'VIP', 300.00),
(2, 'Standard', 150.00),
(3, 'Standard', 80.00),
(4, 'Student', 50.00),
(5, 'VIP', 200.00);

-- 2. View: bilete cumparate de fiecare buyer cu detalii eveniment si locatie
CREATE VIEW buyer_ticket_details AS
SELECT 
    b.buyer_id,
    b.name AS buyer_name,
    b.email,
    t.ticket_id,
    t.category,
    t.price AS ticket_price,
    e.event_id,
    e.title AS event_title,
    e.date AS event_date,
    e.time AS event_time,
    v.name AS venue_name,
    v.location AS venue_location
FROM buyers b
JOIN buyer_tickets bt ON b.buyer_id = bt.buyer_id
JOIN tickets t ON bt.ticket_id = t.ticket_id
JOIN events e ON t.event_id = e.event_id
JOIN venues v ON e.venue_id = v.venue_id;

-- 3. Trigger: actualizare total_sales la vanzarea unui bilet
DELIMITER $$

CREATE TRIGGER update_event_sales
AFTER INSERT ON buyer_tickets
FOR EACH ROW
BEGIN
    DECLARE ticket_price DECIMAL(10,2);
    DECLARE event_id_var INT;

    SELECT t.price, t.event_id INTO ticket_price, event_id_var
    FROM tickets t
    WHERE t.ticket_id = NEW.ticket_id;

    UPDATE events
    SET total_sales = total_sales + ticket_price
    WHERE event_id = event_id_var;

    UPDATE tickets
    SET available = FALSE
    WHERE ticket_id = NEW.ticket_id;
END$$

DELIMITER ;

-- 4. Procedura stocată: afișează sumarul unui eveniment dat
DELIMITER $$

CREATE PROCEDURE get_event_summary(IN input_event_id INT)
BEGIN
    SELECT 
        e.event_id,
        e.title,
        e.date,
        e.time,
        v.name AS venue_name,
        COUNT(bt.ticket_id) AS tickets_sold,
        SUM(t.price) AS total_income
    FROM events e
    JOIN venues v ON e.venue_id = v.venue_id
    LEFT JOIN tickets t ON e.event_id = t.event_id
    LEFT JOIN buyer_tickets bt ON t.ticket_id = bt.ticket_id
    WHERE e.event_id = input_event_id
    GROUP BY e.event_id;
END$$

DELIMITER ;
