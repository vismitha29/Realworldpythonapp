
from flask import Flask, request, jsonify
import sqlite3
from datetime import datetime
from contextlib import closing

app = Flask(__name__)
DB = "food_donation.db"

STATUSES = ["Available", "Claimed", "Picked Up", "Delivered"]


def connect():
    con = sqlite3.connect(DB)
    con.row_factory = sqlite3.Row
    return con


def initialize_db():
    with closing(connect()) as con:
        con.executescript("""
        CREATE TABLE IF NOT EXISTS users (
            user_id TEXT PRIMARY KEY,
            name TEXT NOT NULL,
            role TEXT NOT NULL CHECK(role IN ('Donor', 'Volunteer')),
            phone TEXT NOT NULL
        );

        CREATE TABLE IF NOT EXISTS donations (
            donation_id INTEGER PRIMARY KEY AUTOINCREMENT,
            donor_id TEXT NOT NULL,
            food_name TEXT NOT NULL,
            quantity INTEGER NOT NULL CHECK(quantity > 0),
            location TEXT NOT NULL,
            available_until TEXT NOT NULL,
            contact TEXT NOT NULL,
            status TEXT NOT NULL DEFAULT 'Available',
            volunteer_id TEXT,
            FOREIGN KEY(donor_id) REFERENCES users(user_id),
            FOREIGN KEY(volunteer_id) REFERENCES users(user_id)
        );
        """)
        con.commit()


def error(message, code=400):
    return jsonify({"error": message}), code


@app.route("/")
def home():
    return jsonify({
        "message": "Food Donation Management API",
        "status": "Running"
    })


# Register donor or volunteer
@app.route("/users", methods=["POST"])
def create_user():
    data = request.get_json(silent=True) or {}

    required = ["user_id", "name", "role", "phone"]
    if not all(data.get(k) for k in required):
        return error("All user fields are required.")

    if data["role"] not in ["Donor", "Volunteer"]:
        return error("Role must be Donor or Volunteer.")

    try:
        with closing(connect()) as con:
            con.execute("""
                INSERT INTO users VALUES (?, ?, ?, ?)
            """, (
                data["user_id"],
                data["name"],
                data["role"],
                data["phone"]
            ))
            con.commit()

        return jsonify({
            "message": "User registered successfully",
            "user_id": data["user_id"]
        }), 201

    except sqlite3.IntegrityError:
        return error("User ID already exists.")


# Create donation
@app.route("/donations", methods=["POST"])
def create_donation():
    data = request.get_json(silent=True) or {}

    required = [
        "donor_id", "food_name", "quantity",
        "location", "available_until", "contact"
    ]

    if not all(data.get(k) is not None and data.get(k) != ""
               for k in required):
        return error("All donation fields are required.")

    try:
        quantity = int(data["quantity"])
        if quantity <= 0:
            return error("Quantity must be greater than zero.")

        available_until = datetime.fromisoformat(
            data["available_until"]
        )
        if available_until <= datetime.now():
            return error("Availability time must be in the future.")

    except (ValueError, TypeError):
        return error("Invalid quantity or date/time format.")

    with closing(connect()) as con:
        donor = con.execute(
            "SELECT * FROM users WHERE user_id=? AND role='Donor'",
            (data["donor_id"],)
        ).fetchone()

        if not donor:
            return error("Valid donor not found.", 404)

        cursor = con.execute("""
            INSERT INTO donations
            (donor_id, food_name, quantity, location,
             available_until, contact)
            VALUES (?, ?, ?, ?, ?, ?)
        """, (
            data["donor_id"],
            data["food_name"],
            quantity,
            data["location"],
            available_until.isoformat(),
            data["contact"]
        ))

        donation_id = cursor.lastrowid
        con.commit()

    return jsonify({
        "message": "Donation created successfully",
        "donation_id": f"DON{donation_id:03d}",
        "status": "Available"
    }), 201


# View available donations
@app.route("/donations/available", methods=["GET"])
def available_donations():
    with closing(connect()) as con:
        rows = con.execute("""
            SELECT * FROM donations
            WHERE status='Available'
            AND available_until > ?
            ORDER BY donation_id
        """, (datetime.now().isoformat(),)).fetchall()

    return jsonify([dict(row) for row in rows])


# View all donations
@app.route("/donations", methods=["GET"])
def all_donations():
    with closing(connect()) as con:
        rows = con.execute(
            "SELECT * FROM donations ORDER BY donation_id"
        ).fetchall()

    return jsonify([dict(row) for row in rows])


# Volunteer claims a donation
@app.route("/donations/<int:donation_id>/claim", methods=["POST"])
def claim_donation(donation_id):
    data = request.get_json(silent=True) or {}
    volunteer_id = data.get("volunteer_id")

    if not volunteer_id:
        return error("Volunteer ID is required.")

    with closing(connect()) as con:
        volunteer = con.execute("""
            SELECT * FROM users
            WHERE user_id=? AND role='Volunteer'
        """, (volunteer_id,)).fetchone()

        if not volunteer:
            return error("Valid volunteer not found.", 404)

        donation = con.execute("""
            SELECT * FROM donations WHERE donation_id=?
        """, (donation_id,)).fetchone()

        if not donation:
            return error("Donation not found.", 404)

        if donation["status"] != "Available":
            return error("Donation is not available for claiming.")

        if donation["available_until"] <= datetime.now().isoformat():
            return error("Donation availability has expired.")

        con.execute("""
            UPDATE donations
            SET status='Claimed', volunteer_id=?
            WHERE donation_id=? AND status='Available'
        """, (volunteer_id, donation_id))

        con.commit()

    return jsonify({
        "message": "Donation claimed successfully",
        "donation_id": f"DON{donation_id:03d}",
        "status": "Claimed",
        "volunteer_id": volunteer_id
    })


# Progress donation status
@app.route("/donations/<int:donation_id>/status", methods=["PUT"])
def update_status(donation_id):
    data = request.get_json(silent=True) or {}
    new_status = data.get("status")

    if new_status not in STATUSES:
        return error("Invalid donation status.")

    with closing(connect()) as con:
        donation = con.execute(
            "SELECT * FROM donations WHERE donation_id=?",
            (donation_id,)
        ).fetchone()

        if not donation:
            return error("Donation not found.", 404)

        current_index = STATUSES.index(donation["status"])
        new_index = STATUSES.index(new_status)

        if new_index != current_index + 1:
            return error(
                "Invalid transition. Status must progress one step at a time."
            )

        con.execute("""
            UPDATE donations SET status=?
            WHERE donation_id=?
        """, (new_status, donation_id))

        con.commit()

    return jsonify({
        "message": "Donation status updated successfully",
        "donation_id": f"DON{donation_id:03d}",
        "status": new_status
    })


if __name__ == "__main__":
    initialize_db()
    app.run(debug=True)
    # Realworldpythonapp
