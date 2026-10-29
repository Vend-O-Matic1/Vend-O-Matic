'use strict';
// Vend-O-Matic: zero-dependency beverage vending machine service (Node built-ins only).
const http = require('http');

const PORT = parseInt(process.env.PORT, 10) || 8080;
const PRICE = 2;           // two quarters per beverage
const INITIAL_STOCK = 5;   // five of each beverage
const ITEM_COUNT = 3;      // three beverages (ids 0, 1, 2)

const machine = {
  coins: 0,
  inventory: Array(ITEM_COUNT).fill(INITIAL_STOCK),
};

function send(res, status, headers = {}, body) {
  const out = { ...headers };
  let payload = '';
  if (body !== undefined) {
    payload = JSON.stringify(body);
    out['Content-Type'] = 'application/json';
    out['Content-Length'] = Buffer.byteLength(payload);
  }
  res.writeHead(status, out);
  res.end(payload);
}

function readJson(req) {
  return new Promise((resolve, reject) => {
    let data = '';
    req.on('data', (c) => { data += c; if (data.length > 1e5) req.destroy(); });
    req.on('end', () => {
      if (!data) return resolve(undefined);
      try { resolve(JSON.parse(data)); } catch (e) { reject(e); }
    });
    req.on('error', reject);
  });
}

function parseId(raw) {
  if (!/^\d+$/.test(raw)) return null;
  const id = parseInt(raw, 10);
  return id < ITEM_COUNT ? id : null;
}

async function handle(req, res) {
  const path = req.url.split('?')[0].replace(/\/+$/, '') || '/';
  const method = req.method;

  // PUT / : insert one quarter
  if (path === '/' && method === 'PUT') {
    let body;
    try { body = await readJson(req); } catch { return send(res, 400, {}, { error: 'invalid JSON' }); }
    if (!body || body.coin !== 1) return send(res, 400, { 'X-Coins': machine.coins }, { error: 'only quarters (coin: 1) accepted' });
    machine.coins += 1;
    return send(res, 204, { 'X-Coins': machine.coins });
  }

  // DELETE / : cancel, return all coins
  if (path === '/' && method === 'DELETE') {
    const returned = machine.coins;
    machine.coins = 0;
    return send(res, 204, { 'X-Coins': returned });
  }

  if (path === '/inventory' && method === 'GET') {
    return send(res, 200, {}, machine.inventory);
  }

  const m = path.match(/^\/inventory\/([^/]+)$/);
  if (m) {
    const id = parseId(m[1]);
    if (id === null) return send(res, 404, { 'X-Coins': machine.coins }, { error: 'no such item' });

    if (method === 'GET') return send(res, 200, {}, machine.inventory[id]);

    if (method === 'PUT') {
      if (machine.inventory[id] <= 0) {
        // out of stock: coins stay in the machine
        return send(res, 404, { 'X-Coins': machine.coins });
      }
      if (machine.coins < PRICE) {
        return send(res, 403, { 'X-Coins': machine.coins });
      }
      machine.inventory[id] -= 1;
      const change = machine.coins - PRICE;
      machine.coins = 0;
      return send(res, 200, {
        'X-Coins': change,
        'X-Inventory-Remaining': machine.inventory[id],
      }, { quantity: 1 });
    }
  }

  return send(res, 404, {}, { error: 'not found' });
}

const server = http.createServer((req, res) => {
  handle(req, res).catch(() => send(res, 500, {}, { error: 'internal error' }));
});

server.listen(PORT, () => console.log(`Vend-O-Matic listening on http://localhost:${PORT}`));# Vend-O-Matic
