import psycopg
import time

conninfo = "dbname=yourdb user=youruser password=yourpass host=localhost port=5432 sslmode=disable"

def listen():
    with psycopg.connect(conninfo, autocommit=True) as conn:
        with conn.cursor() as cur:
            cur.execute("LISTEN my_channel;")
            print("Listening...")

            while True:
                conn.poll()
                while conn.notifies:
                    notify = conn.notifies.pop(0)
                    print(f"Received on '{notify.channel}': {notify.payload}")
                time.sleep(1)

listen()
