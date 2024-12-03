```


# Pasta de código-fonte - em Python


from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List
import sqlite3
import math

# Inicializando a aplicação
app = FastAPI()

# Configurando o banco de dados SQLite
conn = sqlite3.connect("pizza_express.db", check_same_thread=False)
cursor = conn.cursor()

# Criação das tabelas
cursor.execute("""
CREATE TABLE IF NOT EXISTS stores (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    latitude REAL,
    longitude REAL,
    pizzas_in_queue INTEGER DEFAULT 0
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS orders (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    customer_name TEXT,
    customer_address TEXT,
    customer_latitude REAL,
    customer_longitude REAL,
    store_id INTEGER,
    status TEXT DEFAULT 'Pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (store_id) REFERENCES stores (id)
)
""")
conn.commit()

# Modelos de entrada e saída
class Store(BaseModel):
    name: str
    latitude: float
    longitude: float
    pizzas_in_queue: int = 0

class Order(BaseModel):
    customer_name: str
    customer_address: str
    customer_latitude: float
    customer_longitude: float

class UpdateOrderStatus(BaseModel):
    status: str

# Função para calcular distância (Haversine formula)
def calculate_distance(lat1, lon1, lat2, lon2):
    R = 6371  # Raio da Terra em km
    d_lat = math.radians(lat2 - lat1)
    d_lon = math.radians(lon2 - lon1)
    a = math.sin(d_lat / 2) ** 2 + math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * math.sin(d_lon / 2) ** 2
    c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))
    return R * c

# Endpoints
@app.post("/stores/", response_model=dict)
def create_store(store: Store):
    cursor.execute("""
    INSERT INTO stores (name, latitude, longitude, pizzas_in_queue)
    VALUES (?, ?, ?, ?)
    """, (store.name, store.latitude, store.longitude, store.pizzas_in_queue))
    conn.commit()
    return {"message": "Store created successfully"}

@app.post("/orders/", response_model=dict)
def create_order(order: Order):
    # Buscar todas as lojas
    cursor.execute("SELECT * FROM stores")
    stores = cursor.fetchall()

    if not stores:
        raise HTTPException(status_code=404, detail="No stores available")

    # Encontrar a loja mais próxima
    closest_store = min(
        stores,
        key=lambda store: calculate_distance(
            order.customer_latitude,
            order.customer_longitude,
            store[2],  # Latitude
            store[3]   # Longitude
        )
    )

    # Criar pedido
    cursor.execute("""
    INSERT INTO orders (customer_name, customer_address, customer_latitude, customer_longitude, store_id)
    VALUES (?, ?, ?, ?, ?)
    """, (order.customer_name, order.customer_address, order.customer_latitude, order.customer_longitude, closest_store[0]))
    conn.commit()

    # Atualizar fila da loja
    cursor.execute("""
    UPDATE stores SET pizzas_in_queue = pizzas_in_queue + 1 WHERE id = ?
    """, (closest_store[0],))
    conn.commit()

    return {
        "message": "Order created successfully",
        "store_assigned": closest_store[1],  # Nome da loja
        "estimated_time_minutes": 15 + closest_store[5] * 2  # Estimativa baseada na fila
    }

@app.patch("/orders/{order_id}/status", response_model=dict)
def update_order_status(order_id: int, update: UpdateOrderStatus):
    cursor.execute("SELECT * FROM orders WHERE id = ?", (order_id,))
    order = cursor.fetchone()
    if not order:
        raise HTTPException(status_code=404, detail="Order not found")

    cursor.execute("""
    UPDATE orders SET status = ? WHERE id = ?
    """, (update.status, order_id))
    conn.commit()

    # Reduzir fila da loja se o pedido for concluído
    if update.status.lower() == "delivered":
        cursor.execute("""
        UPDATE stores SET pizzas_in_queue = pizzas_in_queue - 1 WHERE id = ?
        """, (order[5],))  # store_id
        conn.commit()

    return {"message": "Order status updated successfully"}

@app.get("/stores/{store_id}/orders", response_model=List[dict])
def list_orders_by_store(store_id: int):
    cursor.execute("""
    SELECT * FROM orders WHERE store_id = ?
    """, (store_id,))
    orders = cursor.fetchall()

    if not orders:
        raise HTTPException(status_code=404, detail="No orders found for this store")

    return [
        {
            "id": order[0],
            "customer_name": order[1],
            "customer_address": order[2],
            "status": order[6],
            "created_at": order[7]
        }
        for order in orders
    ]

# Inicialização do servidor
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="127.0.0.1", port=8000)

```


