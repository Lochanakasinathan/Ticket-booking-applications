# Ticket-booking-applications
import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';
import './index.css';

createRoot(document.getElementById('root')).render(<App />);
import React, { useEffect, useState } from 'react';
import EventList from './EventList';
import EventDetails from './EventDetails';
import Bookings from './Bookings';

function App() {
  const [events, setEvents] = useState([]);
  const [selected, setSelected] = useState(null);

  const fetchEvents = async () => {
    const res = await fetch('/api/events');
    const data = await res.json();
    setEvents(data);
  };

  useEffect(() => { fetchEvents(); }, []);

  return (
    <div className="container">
      <h1>Simple Ticket Booking</h1>
      <div className="main">
        <div className="left">
          <EventList events={events} onSelect={setSelected} />
        </div>
        <div className="right">
          {selected ? (
            <EventDetails event={selected} onBooked={() => { fetchEvents(); }} />
          ) : (
            <div>Select an event to see details</div>
          )}
          <hr />
          <Bookings onChange={() => fetchEvents()} />
        </div>
      </div>
    </div>
  );
}

export default App;
import React from 'react';

export default function EventList({ events, onSelect }) {
  return (
    <div>
      <h2>Events</h2>
      {events.length === 0 ? <div>Loading...</div> : (
        <ul className="event-list">
          {events.map(e => (
            <li key={e.id} className="event-item" onClick={() => onSelect(e)}>
              <div className="title">{e.title}</div>
              <div className="meta">{e.available_seats} seats left • ₹{(e.price_cents?e.price_cents/100:e.price).toFixed(2)}</div>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
import React, { useState } from 'react';

export default function EventDetails({ event, onBooked }) {
  const [name, setName] = useState('');
  const [seats, setSeats] = useState(1);
  const [msg, setMsg] = useState(null);
  const price = event.price || (event.price_cents ? event.price_cents/100 : 0);

  async function book(e) {
    e.preventDefault();
    setMsg(null);
    try {
      const res = await fetch('/api/book', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ eventId: event.id, name, seats: Number(seats) })
      });
      const data = await res.json();
      if (!res.ok) {
        setMsg({ type: 'error', text: data.error || 'Booking failed' });
        return;
      }
      setMsg({ type: 'success', text: Booked ${data.seats} seat(s). Booking ID: ${data.id} });
      setName('');
      setSeats(1);
      onBooked?.();
    } catch (err) {
      setMsg({ type: 'error', text: 'Network error' });
    }
  }

  return (
    <div>
      <h2>{event.title}</h2>
      <p>{event.description}</p>
      <p><strong>Price:</strong> ₹{price.toFixed(2)} • <strong>Available:</strong> {event.available_seats}</p>

      <form onSubmit={book} className="booking-form">
        <input value={name} onChange={e => setName(e.target.value)} placeholder="Your name" required />
        <input type="number" min="1" max={event.available_seats} value={seats} onChange={e => setSeats(e.target.value)} required />
        <button type="submit" disabled={event.available_seats <= 0}>Book</button>
      </form>

      {msg && <div className={msg.type === 'error' ? 'msg error' : 'msg success'}>{msg.text}</div>}
    </div>
  );
}
import React, { useEffect, useState } from 'react';

export default function Bookings({ onChange }) {
  const [bookings, setBookings] = useState([]);

  async function load() {
    const res = await fetch('/api/bookings');
    const data = await res.json();
    setBookings(data);
  }
  useEffect(() => { load(); }, []);

  async function cancel(id) {
    if (!window.confirm('Cancel booking?')) return;
    const res = await fetch(/api/bookings/${id}, { method: 'DELETE' });
    if (res.ok) {
      await load();
      onChange?.();
    } else {
      alert('Cancel failed');
    }
  }

  return (
    <div>
      <h3>Your Bookings</h3>
      {bookings.length === 0 ? <div>No bookings yet</div> : (
        <ul className="bookings">
          {bookings.map(b => (
            <li key={b.id}>
              <div><strong>{b.customer_name}</strong> — {b.seats} seat(s) — ₹{(b.total_price||0).toFixed(2)}</div>
              <div className="small">{new Date(b.created_at).toLocaleString()}</div>
              <button onClick={() => cancel(b.id)}>Cancel</button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
body { font-family: Arial, sans-serif; margin: 20px; }
.container { max-width: 1000px; margin: 0 auto; }
.main { display: flex; gap: 20px; }
.left { flex: 1; }
.right { flex: 1.2; }
.event-list { list-style: none; padding: 0; }
.event-item { padding: 10px; border: 1px solid #ddd; margin-bottom: 8px; cursor: pointer; }
.event-item:hover { background: #f9f9f9; }
.title { font-weight: bold; }
.meta { font-size: 0.9em; color: #666; }
.booking-form input { margin-right: 8px; padding: 6px; }
.booking-form button { padding: 6px 10px; }
.msg { margin-top: 8px; padding: 8px; border-radius: 4px; }
.msg.error { background: #ffdede; color: #900; }
.msg.success { background: #e6ffea; color: #065 }
.bookings { list-style: none; padding: 0; }
.bookings li { padding: 8px; border: 1px solid #eee; margin-bottom: 6px; display:flex; justify-content: space-between; align-items:center; }
.small { color: #666; font-size: 0.85em; }
{
  "name": "ticket-server",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "better-sqlite3": "^8.0.0",
    "body-parser": "^1.20.2",
    "cors": "^2.8.5",
    "express": "^4.18.2",
    "uuid": "^9.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');
const Database = require('better-sqlite3');
const { v4: uuidv4 } = require('uuid');
const path = require('path');

const app = express();
app.use(cors());
app.use(bodyParser.json());

// DB file in server/data/tickets.db
const DB_PATH = path.join(__dirname, 'data', 'tickets.db');
const db = new Database(DB_PATH);

// Initialize DB (create tables & seed if needed)
function initDb() {
  db.exec(`
    PRAGMA foreign_keys = ON;
    CREATE TABLE IF NOT EXISTS events (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      description TEXT,
      total_seats INTEGER NOT NULL,
      available_seats INTEGER NOT NULL,
      price_cents INTEGER NOT NULL
    );
    CREATE TABLE IF NOT EXISTS bookings (
      id TEXT PRIMARY KEY,
      event_id INTEGER NOT NULL,
      customer_name TEXT NOT NULL,
      seats INTEGER NOT NULL,
      total_price_cents INTEGER NOT NULL,
      created_at TEXT NOT NULL,
      FOREIGN KEY(event_id) REFERENCES events(id) ON DELETE CASCADE
    );
  `);

  // Seed sample events if table empty
  const row = db.prepare('SELECT COUNT(*) AS c FROM events').get();
  if (row.c === 0) {
    const insert = db.prepare(INSERT INTO events (title,description,total_seats,available_seats,price_cents) VALUES (?, ?, ?, ?, ?));
    insert.run('Indie Rock Concert', 'Outdoor concert with 3 bands', 200, 200, 1500);
    insert.run('Standup Night', 'Comedy evening — 2 shows', 120, 120, 800);
    insert.run('Movie Premiere', 'Premiere screening with Q&A', 80, 80, 300);
    console.log('Seeded events');
  }
}
initDb();

/**
 * Endpoints
 */

// List events
app.get('/api/events', (req, res) => {
  const events = db.prepare('SELECT id, title, description, total_seats, available_seats, price_cents FROM events').all();
  res.json(events.map(e => ({ ...e, price: e.price_cents / 100 })));
});

// Event details
app.get('/api/events/:id', (req, res) => {
  const id = Number(req.params.id);
  const ev = db.prepare('SELECT id, title, description, total_seats, available_seats, price_cents FROM events WHERE id = ?').get(id);
  if (!ev) return res.status(404).json({ error: 'Event not found' });
  ev.price = ev.price_cents / 100;
  delete ev.price_cents;
  res.json(ev);
});

// Create booking (transactional)
app.post('/api/book', (req, res) => {
  const { eventId, name, seats } = req.body;
  if (!eventId || !name || !seats || seats <= 0) {
    return res.status(400).json({ error: 'Missing or invalid fields (eventId, name, seats)' });
  }
  const event = db.prepare('SELECT id, available_seats, price_cents FROM events WHERE id = ?').get(eventId);
  if (!event) return res.status(404).json({ error: 'Event not found' });
  if (event.available_seats < seats) {
    return res.status(409).json({ error: 'Not enough seats available' });
  }

  // Transaction: deduct seats and insert booking
  const tx = db.transaction(() => {
    db.prepare('UPDATE events SET available_seats = available_seats - ? WHERE id = ?').run(seats, eventId);
    const id = uuidv4();
    const total_price_cents = seats * event.price_cents;
    db.prepare('INSERT INTO bookings (id,event_id,customer_name,seats,total_price_cents,created_at) VALUES (?,?,?,?,?,?)')
      .run(id, eventId, name, seats, total_price_cents, new Date().toISOString());
    return id;
  });

  try {
    const bookingId = tx();
    const booking = db.prepare('SELECT id, event_id, customer_name, seats, total_price_cents, created_at FROM bookings WHERE id = ?').get(bookingId);
    booking.total_price = booking.total_price_cents / 100;
    delete booking.total_price_cents;
    res.status(201).json(booking);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Booking failed' });
  }
});

// List bookings
app.get('/api/bookings', (req, res) => {
  const rows = db.prepare('SELECT id, event_id, customer_name, seats, total_price_cents, created_at FROM bookings ORDER BY created_at DESC').all();
  res.json(rows.map(r => ({ ...r, total_price: r.total_price_cents / 100 })));
});

// Cancel booking (refund seats)
app.delete('/api/bookings/:id', (req, res) => {
  const id = req.params.id;
  const booking = db.prepare('SELECT id, event_id, seats FROM bookings WHERE id = ?').get(id);
  if (!booking) return res.status(404).json({ error: 'Booking not found' });

  const tx = db.transaction(() => {
    db.prepare('DELETE FROM bookings WHERE id = ?').run(id);
    db.prepare('UPDATE events SET available_seats = available_seats + ? WHERE id = ?').run(booking.seats, booking.event_id);
  });

  try {
    tx();
    res.json({ ok: true });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Cancel failed' });
  }
});

// serve static (optional) — helpful if you build the React app into server/public
app.use(express.static(path.join(__dirname, 'public')));

const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(Ticket server running on http://localhost:${PORT}));
