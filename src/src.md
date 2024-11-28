# Pasta de código-fonte

// Importando dependências
const express = require("express");
const mongoose = require("mongoose");
const geolib = require("geolib"); // Biblioteca para cálculos geográficos

const app = express();
app.use(express.json());

// Conexão com o banco de dados MongoDB
mongoose.connect("mongodb://localhost:27017/pizza-express", {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

// Schemas e Models
const StoreSchema = new mongoose.Schema({
  name: String,
  latitude: Number,
  longitude: Number,
  pizzasInQueue: Number, // Simula a capacidade atual da unidade
});
const Store = mongoose.model("Store", StoreSchema);

const OrderSchema = new mongoose.Schema({
  customerName: String,
  customerAddress: String,
  customerLatitude: Number,
  customerLongitude: Number,
  storeId: mongoose.Schema.Types.ObjectId,
  status: { type: String, default: "Pending" }, // Pending, In Progress, Delivered
  createdAt: { type: Date, default: Date.now },
});
const Order = mongoose.model("Order", OrderSchema);

// Endpoint: Registrar uma nova loja
app.post("/stores", async (req, res) => {
  const { name, latitude, longitude, pizzasInQueue } = req.body;

  const store = new Store({ name, latitude, longitude, pizzasInQueue });
  await store.save();

  res.status(201).json({ message: "Store added successfully", store });
});

// Endpoint: Registrar um novo pedido
app.post("/orders", async (req, res) => {
  const { customerName, customerAddress, customerLatitude, customerLongitude } =
    req.body;

  // Buscar todas as lojas e encontrar a mais próxima
  const stores = await Store.find();
  const closestStore = stores.reduce((prev, curr) => {
    const prevDistance = geolib.getDistance(
      { latitude: customerLatitude, longitude: customerLongitude },
      { latitude: prev.latitude, longitude: prev.longitude }
    );
    const currDistance = geolib.getDistance(
      { latitude: customerLatitude, longitude: customerLongitude },
      { latitude: curr.latitude, longitude: curr.longitude }
    );

    return prevDistance < currDistance ? prev : curr;
  });

  // Criar o pedido
  const order = new Order({
    customerName,
    customerAddress,
    customerLatitude,
    customerLongitude,
    storeId: closestStore._id,
  });
  await order.save();

  // Atualizar fila de produção da loja
  closestStore.pizzasInQueue += 1;
  await closestStore.save();

  res.status(201).json({
    message: "Order created successfully",
    order,
    assignedStore: closestStore.name,
  });
});

// Endpoint: Atualizar status do pedido
app.patch("/orders/:id/status", async (req, res) => {
  const { status } = req.body;

  const order = await Order.findById(req.params.id);
  if (!order) {
    return res.status(404).json({ message: "Order not found" });
  }

  order.status = status;
  await order.save();

  // Reduzir fila de produção ao concluir o pedido
  if (status === "Delivered") {
    const store = await Store.findById(order.storeId);
    store.pizzasInQueue -= 1;
    await store.save();
  }

  res.json({ message: "Order status updated", order });
});

// Endpoint: Listar pedidos por loja
app.get("/stores/:id/orders", async (req, res) => {
  const orders = await Order.find({ storeId: req.params.id });
  res.json(orders);
});

// Inicialização do servidor
const PORT = 3000;
app.listen(PORT, () => {
  console.log(`🚀 Servidor rodando na porta ${PORT}`);
});
