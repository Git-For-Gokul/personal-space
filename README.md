import psycopg2
import select

# Update with your database credentials
conn = psycopg2.connect(
    dbname="your_db",
    user="your_user",
    password="your_password",
    host="localhost",   # or your DB host
    port=5432,
    sslmode="require"   # Add if using SSL
)

# Enable autocommit to receive notifications
conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)

cur = conn.cursor()
channel = 'counterparty_update'

# Start listening
cur.execute(f"LISTEN {channel};")
print(f"👂 Listening for notifications on channel '{channel}'...")

try:
    while True:
        # Wait for notifications
        if select.select([conn], [], [], 5) == ([], [], []):
            print("⏳ Waiting...")
        else:
            conn.poll()
            while conn.notifies:
                notify = conn.notifies.pop(0)
                print(f"🔔 Received notification on '{notify.channel}': {notify.payload}")

except KeyboardInterrupt:
    print("\n🛑 Exiting...")
    cur.close()
    conn.close()
