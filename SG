const express = require('express'), { createClient } = require('@libsql/client'), cr = require('crypto');
const SECRET = process.env.SECRET || (process.env.VERCEL ? null : 'dev-secret'), KEY = process.env.EXPORT_KEY || '';
if (!SECRET) throw new Error('Falta la variable de entorno SECRET');
const db = createClient({ url: process.env.TURSO_DATABASE_URL || 'file:local.db', authToken: process.env.TURSO_AUTH_TOKEN });
const Q = async (sql, args = []) => (await db.execute({ sql, args })).rows.map(r => ({ ...r }));
const one = async (...a) => (await Q(...a))[0];
const NOW = "(datetime('now','-5 hours'))"; // hora Colombia (UTC-5)

const hash = (p, s = cr.randomBytes(8).toString('hex')) => s + ':' + cr.scryptSync(p, s, 32).toString('hex');
const sig = b => cr.createHmac('sha256', SECRET).update(b).digest('hex');

const SCHEMA = [
  `CREATE TABLE IF NOT EXISTS users(id INTEGER PRIMARY KEY,name TEXT,username TEXT UNIQUE,pass TEXT,role TEXT CHECK(role IN('admin','barbero')),active INT DEFAULT 1)`,
  `CREATE TABLE IF NOT EXISTS services(id INTEGER PRIMARY KEY,name TEXT UNIQUE,kind TEXT CHECK(kind IN('corte','extra')),price REAL,active INT DEFAULT 1)`,
  `CREATE TABLE IF NOT EXISTS products(id INTEGER PRIMARY KEY,name TEXT UNIQUE,price REAL,active INT DEFAULT 1)`,
  `CREATE TABLE IF NOT EXISTS clients(id INTEGER PRIMARY KEY,name TEXT,phone TEXT UNIQUE,cycle_days INT DEFAULT 21,created_at TEXT DEFAULT ${NOW})`,
  `CREATE TABLE IF NOT EXISTS cuts(id INTEGER PRIMARY KEY,barber_id INT REFERENCES users,client_id INT REFERENCES clients,service_id INT REFERENCES services,beard INT,mask INT,price REAL,created_at TEXT DEFAULT ${NOW})`,
  `CREATE TABLE IF NOT EXISTS sales(id INTEGER PRIMARY KEY,barber_id INT REFERENCES users,product_id INT REFERENCES products,qty INT,unit_price REAL,total REAL,created_at TEXT DEFAULT ${NOW})`,
  `CREATE TABLE IF NOT EXISTS availability(id INTEGER PRIMARY KEY,barber_id INT REFERENCES users,weekday INT,start TEXT,"end" TEXT)`,
  `CREATE TABLE IF NOT EXISTS appointments(id INTEGER PRIMARY KEY,barber_id INT REFERENCES users,client_id INT REFERENCES clients,starts_at TEXT,notes TEXT,status TEXT DEFAULT 'pendiente')`,
  `CREATE INDEX IF NOT EXISTS i_cuts ON cuts(created_at,barber_id)`,
  `CREATE INDEX IF NOT EXISTS i_sales ON sales(created_at,barber_id)`
];
// Crea tablas y datos iniciales solo la primera vez (TODO editable desde la app)
const ready = (async () => {
  if ((await Q("SELECT 1 x FROM sqlite_master WHERE name='users'")).length) return;
  await db.batch(SCHEMA, 'write');
  const u = (n, un, role) => ({ sql: 'INSERT INTO users(name,username,pass,role) VALUES(?,?,?,?)', args: [n, un, hash('cambiar123'), role] });
  await db.batch([u('Admin 1', 'admin1', 'admin'), u('Admin 2', 'admin2', 'admin'), ...[1, 2, 3, 4, 5].map(i => u('Barbero ' + i, 'barbero' + i, 'barbero'))], 'write');
  const bs = await Q("SELECT id FROM users WHERE role='barbero'");
  await db.batch([
    ...bs.flatMap(b => [1, 2, 3, 4, 5, 6].map(d => ({ sql: 'INSERT INTO availability(barber_id,weekday,start,"end") VALUES(?,?,?,?)', args: [b.id, d, '09:00', '19:00'] }))),
    ...[['Corte clásico', 'corte', 25000], ['Degradado', 'corte', 30000], ['Corte niño', 'corte', 20000], ['Barba', 'extra', 12000], ['Mascarilla', 'extra', 10000]]
      .map(x => ({ sql: 'INSERT INTO services(name,kind,price) VALUES(?,?,?)', args: x })),
    ...[['Cera', 28000], ['Aceite de barba', 35000], ['Shampoo', 22000]].map(x => ({ sql: 'INSERT INTO products(name,price) VALUES(?,?)', args: x }))
  ], 'write');
})();

const A = f => (q, r) => f(q, r).catch(e => r.status(500).json({ error: e.message }));
const me = async q => {
  const t = (q.headers.cookie || '').split('; ').find(c => c.startsWith('t='))?.slice(2) || '';
  const [id, exp, s] = t.split('.');
  if (!s || s !== sig(id + '.' + exp) || +exp < Date.now()) return;
  return one('SELECT id,name,role FROM users WHERE id=? AND active=1', [id]);
};
const auth = (q, r, n) => me(q).then(u => (q.u = u) ? n() : r.status(401).json({ error: 'No autorizado' }), e => r.status(500).json({ error: e.message }));
const adm = (q, r, n) => q.u.role == 'admin' ? n() : r.status(403).json({ error: 'Solo administradores' });
const isA = q => q.u.role == 'admin' ? 1 : 0;

const app = express();
app.use(express.json());
if (!process.env.VERCEL) app.use(express.static('public'));
app.use((q, r, n) => ready.then(n, e => r.status(500).json({ error: 'Base de datos: ' + e.message })));

app.post('/api/login', A(async (q, r) => {
  const u = await one('SELECT * FROM users WHERE username=? AND active=1', [String(q.body.username || '').trim().toLowerCase()]);
  const ok = u && cr.timingSafeEqual(Buffer.from(hash(String(q.body.password || ''), u.pass.split(':')[0])), Buffer.from(u.pass));
  if (!ok) return r.status(401).json({ error: 'Usuario o clave incorrectos' });
  const b = u.id + '.' + (Date.now() + 30 * 864e5);
  r.setHeader('Set-Cookie', `t=${b}.${sig(b)}; HttpOnly; Secure; SameSite=Strict; Path=/; Max-Age=2592000`);
  r.json({ id: u.id, name: u.name, role: u.role });
}));
app.post('/api/logout', (q, r) => { r.setHeader('Set-Cookie', 't=; Path=/; Max-Age=0'); r.json({}); });
app.get('/api/me', auth, (q, r) => r.json(q.u));

app.get('/api/catalog', auth, A(async (q, r) => {
  const [services, products, barbers] = await Promise.all([Q('SELECT * FROM services'), Q('SELECT * FROM products'), Q("SELECT id,name FROM users WHERE role='barbero' AND active=1")]);
  r.json({ services, products, barbers });
}));
app.put('/api/catalog/:t', auth, adm, A(async (q, r) => {
  const t = q.params.t; if (!['services', 'products'].includes(t)) return r.sendStatus(404);
  const ins = t == 'services' ? "INSERT INTO services(name,price,active,kind) VALUES(?,?,?,'corte')" : 'INSERT INTO products(name,price,active) VALUES(?,?,?)';
  await db.batch(q.body.items.filter(i => i.name).map(i => i.id
    ? { sql: `UPDATE ${t} SET name=?,price=?,active=? WHERE id=?`, args: [i.name, i.price, i.active, i.id] }
    : { sql: ins, args: [i.name, i.price, i.active] }), 'write');
  r.json({ ok: 1 });
}));

// ---- Espejo en tiempo real a Google Sheets (opcional; si falla, el registro ya quedó en Turso) ----
const GE = process.env.GOOGLE_SERVICE_ACCOUNT_EMAIL, GK = (process.env.GOOGLE_PRIVATE_KEY || '').replace(/\\n/g, '\n'), GS = process.env.GOOGLE_SHEET_ID;
const ts = () => new Date(Date.now() - 5 * 36e5).toISOString().replace('T', ' ').slice(0, 19);
const ph = p => { p = String(p || '').replace(/\D/g, ''); return p ? "'" + p : ''; };
const b64 = o => Buffer.from(JSON.stringify(o)).toString('base64url');
let tok = { v: '', exp: 0 };
async function gtoken() {
  if (tok.exp > Date.now()) return tok.v;
  const n = Math.floor(Date.now() / 1e3), h = b64({ alg: 'RS256', typ: 'JWT' }) + '.' + b64({ iss: GE, scope: 'https://www.googleapis.com/auth/spreadsheets', aud: 'https://oauth2.googleapis.com/token', iat: n, exp: n + 3600 });
  const jwt = h + '.' + cr.sign('RSA-SHA256', Buffer.from(h), GK).toString('base64url');
  const r = await fetch('https://oauth2.googleapis.com/token', { method: 'POST', headers: { 'Content-Type': 'application/x-www-form-urlencoded' }, body: new URLSearchParams({ grant_type: 'urn:ietf:params:oauth:grant-type:jwt-bearer', assertion: jwt }) }).then(x => x.json());
  if (!r.access_token) throw new Error(r.error_description || 'No se pudo autenticar con Google');
  tok = { v: r.access_token, exp: Date.now() + (r.expires_in - 60) * 1e3 }; return tok.v;
}
const gfetch = async (path, opt = {}) => {
  const r = await fetch(`https://sheets.googleapis.com/v4/spreadsheets/${GS}${path}`, { ...opt, headers: { Authorization: 'Bearer ' + await gtoken(), 'Content-Type': 'application/json' } });
  if (!r.ok) throw new Error(await r.text()); return r.json();
};
const mirror = async (tab, row) => {
  if (!GE || !GK || !GS) return;
  try {
    await Promise.race([gfetch(`/values/${encodeURIComponent(tab + '!A1')}:append?valueInputOption=USER_ENTERED&insertDataOption=INSERT_ROWS`, { method: 'POST', body: JSON.stringify({ values: [row] }) }),
      new Promise((_, j) => setTimeout(() => j(new Error('timeout')), 5000))]);
  } catch (e) { console.error('Sheets:', e.message); }
};
app.get('/api/sheets-test', auth, adm, A(async (q, r) => {
  if (!GE || !GK || !GS) return r.json({ ok: false, error: 'Faltan variables GOOGLE_* en Vercel' });
  try { r.json({ ok: true, hoja: (await gfetch('?fields=properties.title')).properties.title }); } catch (e) { r.json({ ok: false, error: e.message }); }
}));

// ---- Registro de cortes y ventas ----
const client = async (n, p) => {
  p = String(p || '').replace(/\D/g, ''); if (!p) return null; if (p.length == 10) p = '57' + p;
  const had = await one('SELECT id FROM clients WHERE phone=?', [p]);
  await db.execute({ sql: "INSERT INTO clients(name,phone) VALUES(?,?) ON CONFLICT(phone) DO UPDATE SET name=COALESCE(NULLIF(excluded.name,''),name)", args: [n || '', p] });
  if (!had) await mirror('Clientes', [ts(), n || '', "'" + p]);
  return had ? had.id : (await one('SELECT id FROM clients WHERE phone=?', [p])).id;
};
app.post('/api/cuts', auth, A(async (q, r) => {
  const b = q.body, s = await one("SELECT * FROM services WHERE id=? AND kind='corte' AND active=1", [b.service_id]);
  if (!s) return r.status(400).json({ error: 'Servicio inválido' });
  const E = Object.fromEntries((await Q("SELECT name,price FROM services WHERE kind='extra'")).map(x => [x.name, x.price]));
  const price = s.price + (b.beard ? E.Barba || 0 : 0) + (b.mask ? E.Mascarilla || 0 : 0); // precio calculado en servidor
  const bid = +b.barber_id || q.u.id;
  const i = await db.execute({ sql: 'INSERT INTO cuts(barber_id,client_id,service_id,beard,mask,price) VALUES(?,?,?,?,?,?)', args: [bid, await client(b.name, b.phone), s.id, b.beard ? 1 : 0, b.mask ? 1 : 0, price] });
  const id = Number(i.lastInsertRowid);
  await mirror('Cortes', [ts(), id, (await one('SELECT name FROM users WHERE id=?', [bid])).name, b.name || '', ph(b.phone), s.name, b.beard ? 'Sí' : 'No', b.mask ? 'Sí' : 'No', price]);
  r.json({ id, price });
}));
app.post('/api/sales', auth, A(async (q, r) => {
  const b = q.body, p = await one('SELECT * FROM products WHERE id=? AND active=1', [b.product_id]), n = Math.max(1, +b.qty || 1);
  if (!p) return r.status(400).json({ error: 'Producto inválido' });
  const bid = +b.barber_id || q.u.id;
  const i = await db.execute({ sql: 'INSERT INTO sales(barber_id,product_id,qty,unit_price,total) VALUES(?,?,?,?,?)', args: [bid, p.id, n, p.price, p.price * n] });
  const id = Number(i.lastInsertRowid);
  await mirror('Ventas', [ts(), id, (await one('SELECT name FROM users WHERE id=?', [bid])).name, p.name, n, p.price, p.price * n]);
  r.json({ id, total: p.price * n });
}));

// ---- Reportes y proyecciones (barbero: solo lo suyo) ----
const fit = y => { // regresión lineal simple sobre valores mensuales
  const n = y.length; if (n < 2) return () => y[0] || 0;
  const mx = (n - 1) / 2, my = y.reduce((a, b) => a + b) / n; let a = 0, c = 0;
  y.forEach((v, i) => { a += (i - mx) * (v - my); c += (i - mx) ** 2; });
  return k => Math.max(0, my + a / c * (k - mx));
};
async function proj(u) {
  const m = (await Q(`SELECT m,SUM(n) n,SUM(r) r FROM(SELECT substr(created_at,1,7) m,1 n,price r,barber_id b FROM cuts UNION ALL SELECT substr(created_at,1,7),0,total,barber_id FROM sales) WHERE :u IS NULL OR b=:u GROUP BY m ORDER BY m DESC LIMIT 7`, { u })).reverse();
  const cur = new Date(Date.now() - 5 * 36e5).toISOString().slice(0, 7), part = m.length > 1 && m[m.length - 1].m == cur;
  if (part) m.pop(); // el mes en curso está incompleto: no entra a la regresión
  const fr = fit(m.map(x => x.r)), fn = fit(m.map(x => x.n)), base = m.length - 1 + (part ? 1 : 0);
  return { meses: m.length, p: [1, 3, 6].map(k => ({ k, cuts: Math.round(fn(base + k)), rev: Math.round(fr(base + k)) })) };
}
app.get('/api/stats', auth, A(async (q, r) => {
  const u = isA(q) ? null : q.u.id, p = { f: q.query.from + ' 00:00:00', t: q.query.to + ' 23:59:59', u }, W = 'created_at BETWEEN :f AND :t';
  const [per, c, pr] = await Promise.all([
    Q(`SELECT u.id,u.name,
      (SELECT COUNT(*) FROM cuts WHERE barber_id=u.id AND ${W}) cuts,
      (SELECT COALESCE(SUM(price),0) FROM cuts WHERE barber_id=u.id AND ${W}) cut_rev,
      (SELECT COALESCE(SUM(total),0) FROM sales WHERE barber_id=u.id AND ${W}) sales_rev
      FROM users u WHERE u.role='barbero' AND (:u IS NULL OR u.id=:u)`, p),
    one(`SELECT COUNT(DISTINCT client_id) n FROM cuts WHERE ${W} AND (:u IS NULL OR barber_id=:u)`, p),
    proj(u)]);
  r.json({ per, clients: c.n, proj: pr });
}));

app.get('/api/clients', auth, adm, A(async (q, r) => r.json(await Q(
  'SELECT c.*,(SELECT MAX(created_at) FROM cuts WHERE client_id=c.id) last,(SELECT COUNT(*) FROM cuts WHERE client_id=c.id) visits FROM clients c'))));

// ---- Agenda y disponibilidad ----
app.get('/api/appointments', auth, A(async (q, r) => r.json(await Q(
  `SELECT a.id,a.starts_at,a.status,u.name barber,c.name client,c.phone FROM appointments a JOIN users u ON u.id=a.barber_id LEFT JOIN clients c ON c.id=a.client_id
   WHERE a.starts_at>=? AND a.status='pendiente' AND (?=1 OR a.barber_id=?) ORDER BY a.starts_at`, [q.query.from || '', isA(q), q.u.id]))));
app.post('/api/appointments', auth, A(async (q, r) => {
  const b = q.body, bar = isA(q) ? +b.barber_id : q.u.id;
  if (!bar || !b.starts_at) return r.status(400).json({ error: 'Falta barbero o fecha' });
  await db.execute({ sql: 'INSERT INTO appointments(barber_id,client_id,starts_at) VALUES(?,?,?)', args: [bar, await client(b.name, b.phone), b.starts_at] });
  r.json({ ok: 1 });
}));
app.delete('/api/appointments/:id', auth, A(async (q, r) => {
  await db.execute({ sql: "UPDATE appointments SET status='cancelada' WHERE id=? AND (?=1 OR barber_id=?)", args: [q.params.id, isA(q), q.u.id] }); r.json({ ok: 1 });
}));
app.get('/api/availability', auth, A(async (q, r) => r.json(await Q('SELECT id,barber_id,weekday,start,"end" AS "end" FROM availability WHERE ?=1 OR barber_id=?', [isA(q), q.u.id]))));
app.put('/api/availability', auth, adm, A(async (q, r) => {
  await db.batch([{ sql: 'DELETE FROM availability WHERE barber_id=?', args: [q.body.barber_id] },
    ...q.body.rows.map(x => ({ sql: 'INSERT INTO availability(barber_id,weekday,start,"end") VALUES(?,?,?,?)', args: [q.body.barber_id, x.weekday, x.start, x.end] }))], 'write');
  r.json({ ok: 1 });
}));

// ---- Usuarios (crear / actualizar / restablecer clave / desactivar) ----
app.post('/api/users', auth, adm, A(async (q, r) => {
  const b = q.body, un = String(b.username || '').trim().toLowerCase(); if (!un || !b.name) return r.status(400).json({ error: 'Falta nombre o usuario' });
  const e = await one('SELECT id FROM users WHERE username=?', [un]);
  if (e) await db.execute({ sql: 'UPDATE users SET name=?,role=?,active=?' + (b.password ? ',pass=?' : '') + ' WHERE id=?', args: [b.name, b.role, b.active ? 1 : 0, ...(b.password ? [hash(b.password)] : []), e.id] });
  else await db.execute({ sql: 'INSERT INTO users(name,username,pass,role) VALUES(?,?,?,?)', args: [b.name, un, hash(b.password || 'cambiar123'), b.role] });
  r.json({ ok: 1 });
}));

// ---- Exportación CSV (sesión admin, o ?key=EXPORT_KEY para Google Sheets IMPORTDATA) ----
const EX = {
  cuts: `SELECT c.id,c.created_at fecha,u.name barbero,cl.name cliente,cl.phone telefono,s.name servicio,c.beard barba,c.mask mascarilla,c.price precio FROM cuts c JOIN users u ON u.id=c.barber_id JOIN services s ON s.id=c.service_id LEFT JOIN clients cl ON cl.id=c.client_id ORDER BY c.id`,
  sales: `SELECT s.id,s.created_at fecha,u.name barbero,p.name producto,s.qty cantidad,s.unit_price precio,s.total FROM sales s JOIN users u ON u.id=s.barber_id JOIN products p ON p.id=s.product_id ORDER BY s.id`,
  clients: `SELECT c.name nombre,c.phone telefono,c.cycle_days ciclo_dias,(SELECT COUNT(*) FROM cuts WHERE client_id=c.id) cortes,(SELECT MAX(created_at) FROM cuts WHERE client_id=c.id) ultimo_corte,date((SELECT MAX(created_at) FROM cuts WHERE client_id=c.id),'+'||c.cycle_days||' days') proximo_corte FROM clients c`,
  resumen: `SELECT substr(c.created_at,1,7) mes,u.name barbero,COUNT(*) cortes,SUM(c.price) ingresos_cortes,(SELECT COALESCE(SUM(total),0) FROM sales s WHERE s.barber_id=u.id AND substr(s.created_at,1,7)=substr(c.created_at,1,7)) ventas FROM cuts c JOIN users u ON u.id=c.barber_id GROUP BY mes,u.id`
};
const csv = rows => rows.length ? [Object.keys(rows[0]).join(','), ...rows.map(o => Object.values(o).map(v => '"' + String(v ?? '').replace(/"/g, '""') + '"').join(','))].join('\n') : '';
app.get('/api/export/:n', A(async (q, r) => {
  const u = await me(q); if (!((KEY && q.query.key === KEY) || u?.role == 'admin')) return r.sendStatus(401);
  const sql = EX[q.params.n.replace('.csv', '')]; if (!sql) return r.sendStatus(404);
  r.type('text/csv; charset=utf-8').send('\ufeff' + csv(await Q(sql)));
}));

module.exports = app;
if (require.main === module) app.listen(process.env.PORT || 3000, () => console.log('Barbería en http://localhost:' + (process.env.PORT || 3000)));
