## 📖 OSI Layer Mapping — Lab 1 Devices

| Device | OSI Layer | Why |
|--------|-----------|-----|
| Copper cable | Layer 1 — Physical | Carries raw electrical bits |
| Switch SW1 | Layer 2 — Data Link | Reads MAC addresses, forwards frames |
| Router R1 | Layer 3 — Network | Reads IP addresses, routes packets |
| PC (OS/NIC) | Layers 1–4 | Creates and processes all data |
| Ping utility | Layer 7 — Application | User-facing network tool |

## 🧠 Key Insight
When a ping travels from PC1 to PC2:
- It becomes a **packet** at Layer 3 (IP addresses added)
- It becomes a **frame** at Layer 2 (MAC addresses added)  
- It becomes **bits** at Layer 1 (sent on the wire)
This process is called **encapsulation**.