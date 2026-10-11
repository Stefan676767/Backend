# Import Python's built-in SQLite database tool
import sqlite3
# Import FastAPI framework modules and web error handlers
from fastapi import FastAPI, HTTPException
# Import Pydantic's BaseModel to natively validate incoming JSON data
from pydantic import BaseModel
# Import the CORS middleware to let separate HTML apps talk to this sandbox API
from fastapi.middleware.cors import CORSMiddleware

# Initialize the main FastAPI application instance
app = FastAPI()

# SECURITY SETUP: Allows your separate HTML app to read/write data to this sandbox engine
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"], # Allows external files to talk to OneCompiler's background port
    allow_credentials=True,
    allow_methods=["*"], # Allows GET and POST requests
    allow_headers=["*"],
)

# Name your local SQLite tracking file
DB_NAME = "independent_text_db.db"

# Create a helper function to set up your database structures
def init_db():
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    # Create a table that maps an auto-incrementing ID to a text note string
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS shared_notes (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            note_content TEXT
        )
    """)
    conn.commit()
    conn.close()

# Initialize the database table structure immediately as the file compiles
init_db()

# --- DEFINING THE JSON DATA STRUCTURE ---
# Instead of using HTML Forms, we use a clean Pydantic model to receive JSON text.
# This completely eliminates the need for the "python-multipart" library!
class NotePayload(BaseModel):
    text: str

# --- 1. THE ROOT HEALTH ROUTE ---
@app.get("/")
def home():
    # Satisfies OneCompiler's automatic background health checker
    return {"status": "Server is alive", "mode": "JSON-Only Engine"}

# --- 2. THE SAVE TEXT ROUTE (POST) ---
@app.post("/save/")
def save_note(payload: NotePayload): # Accepts the Pydantic data structure natively
    # Check if the user passed a blank text string
    if not payload.text.strip():
        raise HTTPException(status_code=400, detail="Text content cannot be blank.")
        
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    # Insert the string into your SQLite database row
    cursor.execute("INSERT INTO shared_notes (note_content) VALUES (?)", (payload.text,))
    new_id = cursor.lastrowid
    conn.commit()
    conn.close()
    
    # Return a clean JSON validation map back to your frontend code
    return {"status": "Saved", "assigned_id": new_id}

# --- 3. THE FETCH TEXT ROUTE (GET) ---
@app.get("/get/{note_id}")
def get_note(note_id: int):
    conn = sqlite3.connect(DB_NAME)
    cursor = conn.cursor()
    cursor.execute("SELECT note_content FROM shared_notes WHERE id = ?", (note_id,))
    row = cursor.fetchone()
    conn.close()
    
    if not row:
        raise HTTPException(status_code=404, detail="Note ID not found in database.")
        
    # Return the text content matching that ID row number
    return {"id": note_id, "content": row[0]}
