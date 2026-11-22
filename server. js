'use strict';
/**
 * سراج فون v2 - Pi Network Store
 * قاعدة البيانات: Firebase Firestore | الصور: Cloudinary | الاستضافة: Render
 *
 * المتغيرات البيئية (Environment Variables):
 *  FIREBASE_SERVICE_ACCOUNT  محتوى ملف JSON الكامل لحساب خدمة Firebase
 *  CLOUDINARY_CLOUD_NAME / CLOUDINARY_API_KEY / CLOUDINARY_API_SECRET
 *  PI_API_KEY, APP_WALLET_SEED, PI_SANDBOX (true/false)
 *  ADMIN_PASSWORD, ADMIN_PI_USERNAMES (اختياري), SESSION_SECRET
 */
const express = require('express');
const crypto = require('crypto');
const fs = require('fs');
const path = require('path');
const admin = require('firebase-admin');
const cloudinary = require('cloudinary').v2;
const StellarSdk = require('@stellar/stellar-sdk');

const PORT = process.env.PORT || 3000;
const PI_API = 'https://api.minepi.com/v2';
const PI_API_KEY = process.env.PI_API_KEY || '';
const APP_WALLET_SEED = process.env.APP_WALLET_SEED || '';
const SANDBOX = String(process.env.PI_SANDBOX || 'true') === 'true';
const ADMIN_PASSWORD = process.env.ADMIN_PASSWORD || '';
const ADMIN_PI = (process.env.ADMIN_PI_USERNAMES || '').split(',').map(s => s.trim().toLowerCase()).filter(Boolean);
const SESSION_SECRET = process.env.SESSION_SECRET || crypto.randomBytes(32).toString('hex');
const HORIZON = SANDBOX ? 'https://api.testnet.minepi.com' : 'https://api.mainnet.minepi.com';
const NET_PASS = SANDBOX ? 'Pi Testnet' : 'Pi Network';
const CLOUD = process.env.CLOUDINARY_CLOUD_NAME || '';
const CK = process.env.CLOUDINARY_API_KEY || '';
const CS = process.env.CLOUDINARY_API_SECRET || '';
const FOLDER = 'sirajphone';
if (!process.env.SESSION_SECRET) console.warn('تنبيه: SESSION_SECRET غير مضبوط');

/* ---------- Firebase ---------- */
let sa;
try { sa = JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT || ''); sa.private_key = sa.private_key.replace(/\\n/g, '\n'); }
catch (e) { console.error('FIREBASE_SERVICE_ACCOUNT غير موجود أو ليس JSON صحيحاً'); process.exit(1); }
admin.initializeApp({ credential: admin.credential.cert(sa) });
const fdb = admin.firestore();
fdb.settings({ ignoreUndefinedProperties: true });
const FV = admin.firestore.FieldValue;
const col = n => fdb.collection(n);
const listDocs = async q => (await q.get()).docs.map(d => ({ id: d.id, ...d.data() }));
const getDoc = async (c, id) => {
  id = String(id || ''); if (!id || id.includes('/')) return null;
  const d = await col(c).doc(id).get(); return d.exists ? { id: d.id, ...d.data() } : null;
};
const saveOrder = o => { const { id, ...rest } = o; return col('orders').doc(id).set(rest); };

cloudinary.config({ cloud_name: CLOUD, api_key: CK, api_secret: CS, secure: true });

const DEFAULT_SETTINGS = { shopPhone: '', refundDays: 7, deliveryNote: 'أجرة التوصيل تُدفع نقداً للمندوب عند استلام الطلب.' };
let settings = { ...DEFAULT_SETTINGS };
async function loadSettings() { const d = await fdb.doc('config/settings').get(); settings = { ...DEFAULT_SETTINGS, ...(d.exists ? d.data() : {}) }; }

const newId = p => p + crypto.randomBytes(5).toString('hex');
const round7 = n => Math.round(n * 1e7) / 1e7;
const now = () => Date.now();
const fail = (http, msg) => { const e = new Error(msg); e.http = http; return e; };

/* ---------- الجلسات ---------- */
function sign(payload) {
  const body = Buffer.from(JSON.stringify({ ...payload, exp: now() + 30 * 864e5 })).toString('base64url');
  return body + '.' + crypto.createHmac('sha256', SESSION_SECRET).update(body).digest('base64url');
}
function verify(token) {
  if (!token) return null;
  const [body, sig] = token.split('.');
  if (!body || !sig) return null;
  const exp = crypto.createHmac('sha256', SESSION_SECRET).update(body).digest('base64url');
  if (sig.length !== exp.length || !crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(exp))) return null;
  try { const p = JSON.parse(Buffer.from(body, 'base64url').toString()); return p.exp > now() ? p : null; } catch { return null; }
}
const safeEq = (a, b) => crypto.timingSafeEqual(crypto.createHash('sha256').update(String(a)).digest(), crypto.createHash('sha256').update(String(b)).digest());

/* ---------- Pi API ---------- */
async function pi(url, { method = 'GET', body, token } = {}) {
  const r = await fetch(PI_API + url, {
    method,
    headers: { 'Content-Type': 'application/json', Authorization: token ? `Bearer ${token}` : `Key ${PI_API_KEY}` },
    body: body ? JSON.stringify(body) : undefined
  });
  const text = await r.text();
  let data; try { data = JSON.parse(text); } catch { data = { raw: text }; }
  if (!r.ok) { const e = new Error(data.error_message || data.message || data.error || `Pi API ${r.status}`); e.http = 502; throw e; }
  return data;
}

/* ---------- Express ---------- */
const app = express();
app.use(express.json({ limit: '300kb' }));
app.use((req, res, next) => { req.auth = verify((req.headers.authorization || '').replace('Bearer ', '')); next(); });
const ah = fn => (req, res) => Promise.resolve(fn(req, res)).catch(e => {
  console.error(req.method, req.path, e.message);
  res.status(e.http || 500).json({ error: e.http ? e.message : 'خطأ في الخادم' });
});
const needUser = (req, res, next) => req.auth && req.auth.uid ? next() : res.status(401).json({ error: 'سجّل الدخول أولاً' });
const needAdmin = (req, res, next) => req.auth && req.auth.admin ? next() : res.status(403).json({ error: 'غير مصرح' });

app.get('/', (req, res) => res.sendFile(path.join(__dirname, 'index.html')));
app.get('/validation-key.txt', (req, res) => {
  const f = path.join(__dirname, 'validation-key.txt');
  fs.existsSync(f) ? res.sendFile(f) : res.status(404).end();
});
app.get('/healthz', (req, res) => res.send('ok'));

app.get('/api/config', (req, res) => res.json({ sandbox: SANDBOX, shop: 'سراج فون', ...settings }));

app.post('/api/login', ah(async (req, res) => {
  const { accessToken } = req.body || {};
  if (!accessToken) throw fail(400, 'accessToken مفقود');
  const me = await pi('/me', { token: accessToken });
  const old = await getDoc('users', me.uid);
  const u = { uid: me.uid, username: me.username, walletAddress: me.wallet_address || (old && old.walletAddress) || null, createdAt: (old && old.createdAt) || now(), lastLogin: now() };
  await col('users').doc(me.uid).set(u);
  const isAdmin = ADMIN_PI.includes(String(u.username).toLowerCase());
  res.json({ token: sign({ uid: u.uid, username: u.username, admin: isAdmin }), user: { uid: u.uid, username: u.username, walletAddress: u.walletAddress }, admin: isAdmin });
}));
app.post('/api/admin/login', ah(async (req, res) => {
  if (!ADMIN_PASSWORD || !safeEq((req.body || {}).password || '', ADMIN_PASSWORD)) throw fail(401, 'كلمة المرور غير صحيحة');
  res.json({ token: sign({ admin: true, username: 'admin' }), admin: true });
}));

/* ---------- المنتجات ---------- */
const pubProduct = p => ({ id: p.id, name: p.name, category: p.category, description: p.description, price: p.price, stock: p.stock, active: p.active, images: p.images || [], rating: p.rating || 0, reviewsCount: p.reviewsCount || 0, updatedAt: p.updatedAt });
app.get('/api/products', ah(async (req, res) => {
  const l = await listDocs(col('products').where('active', '==', true));
  res.json(l.sort((a, b) => b.createdAt - a.createdAt).map(pubProduct));
}));

function cleanImages(arr) {
  if (!Array.isArray(arr)) return [];
  const pre = `https://res.cloudinary.com/${CLOUD}/`;
  return arr.slice(0, 6).filter(i => i && typeof i.url === 'string' && i.url.startsWith(pre) && typeof i.pid === 'string' && i.pid.startsWith(FOLDER + '/'))
    .map(i => ({ url: i.url.slice(0, 400), pid: i.pid.slice(0, 200) }));
}
const destroyImgs = pids => Promise.all(pids.filter(p => String(p).startsWith(FOLDER + '/')).map(p => cloudinary.uploader.destroy(p).catch(() => {})));

app.post('/api/admin/upload-sign', needAdmin, ah(async (req, res) => {
  if (!CLOUD || !CK || !CS) throw fail(500, 'بيانات Cloudinary غير مضبوطة في الخادم');
  const timestamp = Math.round(Date.now() / 1000);
  res.json({ cloudName: CLOUD, apiKey: CK, timestamp, folder: FOLDER, signature: cloudinary.utils.api_sign_request({ timestamp, folder: FOLDER }, CS) });
}));
app.post('/api/admin/images/delete', needAdmin, ah(async (req, res) => {
  await destroyImgs(Array.isArray(req.body.pids) ? req.body.pids.slice(0, 12) : []); res.json({ ok: true });
}));

function cleanProduct(b, old = {}) {
  const t = (v, n) => String(v ?? '').trim().slice(0, n);
  const p = { ...old, name: t(b.name, 120), category: t(b.category, 40) || 'هواتف', description: t(b.description, 2000), price: round7(Number(b.price)), stock: Math.max(0, parseInt(b.stock) || 0), active: b.active !== false, updatedAt: now() };
  if (p.name.length < 2) throw fail(400, 'اسم المنتج مطلوب');
  if (!(p.price > 0)) throw fail(400, 'السعر بعملة Pi يجب أن يكون أكبر من صفر');
  if (b.images !== undefined) p.images = cleanImages(b.images);
  p.images = p.images || [];
  return p;
}
const saveProduct = p => { const { id, ...rest } = p; return col('products').doc(id).set(rest); };

app.get('/api/admin/products', needAdmin, ah(async (req, res) => {
  res.json((await listDocs(col('products'))).sort((a, b) => b.createdAt - a.createdAt).map(pubProduct));
}));
app.post('/api/admin/products', needAdmin, ah(async (req, res) => {
  const p = cleanProduct(req.body, { id: newId('p'), createdAt: now(), rating: 0, reviewsCount: 0 });
  await saveProduct(p); res.json(pubProduct(p));
}));
app.put('/api/admin/products/:id', needAdmin, ah(async (req, res) => {
  const old = await getDoc('products', req.params.id);
  if (!old) throw fail(404, 'غير موجود');
  const p = cleanProduct(req.body, old);
  const keep = new Set(p.images.map(i => i.pid));
  await destroyImgs((old.images || []).map(i => i.pid).filter(x => !keep.has(x)));
  await saveProduct(p); res.json(pubProduct(p));
}));
app.delete('/api/admin/products/:id', needAdmin, ah(async (req, res) => {
  const old = await getDoc('products', req.params.id);
  if (old) {
    await destroyImgs((old.images || []).map(i => i.pid));
    const rs = await col('reviews').where('productId', '==', old.id).get();
    const batch = fdb.batch(); rs.docs.forEach(d => batch.delete(d.ref)); batch.delete(col('products').doc(old.id)); await batch.commit();
  }
  res.json({ ok: true });
}));

/* ---------- التقييمات ---------- */
async function recompute(pid) {
  const rs = await listDocs(col('reviews').where('productId', '==', pid));
  await col('products').doc(pid).update({ rating: rs.length ? round7(rs.reduce((s, r) => s + r.rating, 0) / rs.length) : 0, reviewsCount: rs.length }).catch(() => {});
}
app.get('/api/products/:id/reviews', ah(async (req, res) => {
  const l = await listDocs(col('reviews').where('productId', '==', req.params.id));
  res.json(l.sort((a, b) => b.createdAt - a.createdAt).map(r => ({ id: r.id, username: r.username, rating: r.rating, comment: r.comment, createdAt: r.createdAt })));
}));
const PAID_OK = ['paid', 'processing', 'shipped', 'delivered'];
app.post('/api/products/:id/reviews', needUser, ah(async (req, res) => {
  const p = await getDoc('products', req.params.id);
  if (!p) throw fail(404, 'المنتج غير موجود');
  const orders = await listDocs(col('orders').where('userUid', '==', req.auth.uid));
  if (!orders.some(o => o.productId === p.id && PAID_OK.includes(o.status))) throw fail(403, 'يمكن تقييم المنتج بعد شرائه فقط');
  const rating = Math.round(Number(req.body.rating));
  if (!(rating >= 1 && rating <= 5)) throw fail(400, 'التقييم من 1 إلى 5');
  const comment = String(req.body.comment || '').trim().slice(0, 500);
  await col('reviews').doc(`${p.id}_${req.auth.uid}`).set({ productId: p.id, userUid: req.auth.uid, username: req.auth.username, rating, comment, createdAt: now() });
  await recompute(p.id); res.json({ ok: true });
}));
app.delete('/api/admin/reviews/:id', needAdmin, ah(async (req, res) => {
  const r = await getDoc('reviews', req.params.id);
  if (r) { await col('reviews').doc(r.id).delete(); await recompute(r.productId); }
  res.json({ ok: true });
}));

/* ---------- الطلبات ---------- */
function cleanCustomer(c = {}) {
  const t = (v, n) => String(v || '').trim().slice(0, n);
  const out = { name: t(c.name, 80), phone: t(c.phone, 20).replace(/[\s-]/g, ''), governorate: t(c.governorate, 40), city: t(c.city, 80), address: t(c.address, 250), notes: t(c.notes, 250) };
  if (c.lat != null && c.lng != null && isFinite(c.lat) && isFinite(c.lng)) { out.lat = +Number(c.lat).toFixed(6); out.lng = +Number(c.lng).toFixed(6); }
  if (out.name.length < 3) throw fail(400, 'أدخل الاسم الكامل');
  if (!/^(\+964|00964|0)?7\d{9}$/.test(out.phone)) throw fail(400, 'رقم الهاتف غير صحيح (مثال 07701234567)');
  if (!out.governorate || out.address.length < 5) throw fail(400, 'أدخل المحافظة والعنوان بالتفصيل');
  return out;
}
app.post('/api/orders', needUser, ah(async (req, res) => {
  const p = await getDoc('products', req.body.productId);
  if (!p || !p.active) throw fail(404, 'المنتج غير متوفر');
  const qty = Math.max(1, Math.min(10, parseInt(req.body.qty) || 1));
  if (p.stock < qty) throw fail(400, 'الكمية المطلوبة غير متوفرة');
  const customer = cleanCustomer(req.body.customer);
  const o = {
    id: 'SF-' + crypto.randomBytes(3).toString('hex').toUpperCase(), userUid: req.auth.uid, username: req.auth.username,
    productId: p.id, productName: p.name, qty, unitPi: p.price, totalPi: round7(p.price * qty), customer,
    status: 'pending_payment', refund: { status: 'none' }, createdAt: now(), history: [{ s: 'pending_payment', t: now() }]
  };
  await saveOrder(o); res.json({ order: o });
}));
app.get('/api/orders/mine', needUser, ah(async (req, res) => {
  const l = await listDocs(col('orders').where('userUid', '==', req.auth.uid));
  res.json(l.filter(o => !['pending_payment', 'cancelled'].includes(o.status)).sort((a, b) => b.createdAt - a.createdAt).map(o => ({ ...o, userUid: undefined })));
}));
app.post('/api/orders/:id/refund', needUser, ah(async (req, res) => {
  const o = await getDoc('orders', req.params.id);
  if (!o || o.userUid !== req.auth.uid) throw fail(404, 'الطلب غير موجود');
  if (!PAID_OK.includes(o.status)) throw fail(400, 'لا يمكن طلب الاسترجاع لهذا الطلب');
  if (o.refund.status !== 'none') throw fail(400, 'تم تقديم طلب استرجاع لهذا الطلب مسبقاً');
  const days = Number(settings.refundDays) || 7;
  if (now() - (o.deliveredAt || o.paidAt) > days * 864e5) throw fail(400, `انتهت مدة الاسترجاع (${days} أيام)`);
  const reason = String(req.body.reason || '').trim().slice(0, 500);
  if (reason.length < 5) throw fail(400, 'اكتب سبب الاسترجاع');
  o.refund = { status: 'requested', reason, requestedAt: now() }; await saveOrder(o); res.json({ ok: true });
}));

/* ---------- مدفوعات Pi (مستخدم → تطبيق) ---------- */
async function releaseStock(oid) {
  const oref = col('orders').doc(oid);
  await fdb.runTransaction(async t => {
    const os = await t.get(oref); const o = os.data();
    if (!o || !o.stockReserved) return;
    const pref = col('products').doc(o.productId); const ps = await t.get(pref);
    if (ps.exists) t.update(pref, { stock: ps.data().stock + o.qty });
    t.update(oref, { stockReserved: false });
  });
}
const cancelOrder = async oid => { await releaseStock(oid); await col('orders').doc(oid).update({ status: 'cancelled', history: FV.arrayUnion({ s: 'cancelled', t: now() }) }); };
const orderOfPayment = pay => getDoc('orders', (pay.metadata || {}).orderId);

app.post('/api/payments/approve', needUser, ah(async (req, res) => {
  const id = String(req.body.paymentId || '');
  const pay = await pi('/payments/' + encodeURIComponent(id));
  const o = await orderOfPayment(pay);
  if (!o || o.userUid !== req.auth.uid || pay.user_uid !== req.auth.uid) throw fail(403, 'الدفعة لا تخص هذا الحساب');
  if (o.status !== 'pending_payment') throw fail(400, 'حالة الطلب لا تسمح بالدفع');
  if (Math.abs(Number(pay.amount) - o.totalPi) > 1e-6) throw fail(400, 'المبلغ لا يطابق قيمة الطلب');
  const oref = col('orders').doc(o.id), pref = col('products').doc(o.productId);
  await fdb.runTransaction(async t => {
    const [os, ps] = await Promise.all([t.get(oref), t.get(pref)]);
    const cur = os.data();
    if (cur.status !== 'pending_payment') throw fail(400, 'حالة الطلب لا تسمح بالدفع');
    if (!cur.stockReserved) {
      const p = ps.data();
      if (!ps.exists || !p.active || p.stock < cur.qty) throw fail(400, 'نفدت الكمية');
      t.update(pref, { stock: p.stock - cur.qty });
    }
    t.update(oref, { stockReserved: true, paymentId: id });
  });
  try { await pi(`/payments/${id}/approve`, { method: 'POST' }); }
  catch (e) { await releaseStock(o.id); throw e; }
  res.json({ ok: true });
}));

async function finalizePayment(id, txid, uid) {
  const pay = await pi('/payments/' + encodeURIComponent(id));
  const o = await orderOfPayment(pay);
  if (!o) throw fail(404, 'الطلب غير موجود');
  if (uid && o.userUid !== uid) throw fail(403, 'غير مصرح');
  if (o.status === 'pending_payment') {
    txid = txid || (pay.transaction || {}).txid;
    if (!txid) throw fail(400, 'txid مفقود');
    if (!(pay.status || {}).developer_completed) await pi(`/payments/${id}/complete`, { method: 'POST', body: { txid } });
    Object.assign(o, { status: 'paid', txid, paidAt: now(), paymentId: id }); o.history.push({ s: 'paid', t: now() });
    await saveOrder(o);
  }
  return o;
}
app.post('/api/payments/complete', needUser, ah(async (req, res) => {
  const o = await finalizePayment(String(req.body.paymentId || ''), req.body.txid, req.auth.uid); res.json({ ok: true, orderId: o.id });
}));
app.post('/api/payments/cancel', needUser, ah(async (req, res) => {
  const pay = await pi('/payments/' + encodeURIComponent(String(req.body.paymentId || '')));
  const o = await orderOfPayment(pay);
  if (o && o.userUid === req.auth.uid && o.status === 'pending_payment') await cancelOrder(o.id);
  res.json({ ok: true });
}));
app.post('/api/payments/incomplete', ah(async (req, res) => {
  const id = String((((req.body || {}).payment) || {}).identifier || '');
  if (!id) throw fail(400, 'identifier مفقود');
  const pay = await pi('/payments/' + encodeURIComponent(id)); const st = pay.status || {};
  if (st.developer_completed) return res.json({ ok: true });
  if ((pay.transaction || {}).txid) await finalizePayment(id, pay.transaction.txid);
  else if (!st.cancelled && !st.user_cancelled) {
    await pi(`/payments/${id}/cancel`, { method: 'POST' }).catch(() => {});
    const o = await orderOfPayment(pay); if (o && o.status === 'pending_payment') await cancelOrder(o.id);
  }
  res.json({ ok: true });
}));

/* ---------- الاسترجاع (تطبيق → مستخدم) ---------- */
async function processRefund(o) {
  if (!PI_API_KEY || !APP_WALLET_SEED) throw fail(500, 'يجب ضبط PI_API_KEY و APP_WALLET_SEED لتنفيذ الاسترجاع');
  const r = o.refund; r.status = 'processing'; await saveOrder(o);
  try {
    let pay;
    if (r.paymentId) pay = await pi('/payments/' + r.paymentId);
    else {
      pay = await pi('/payments', { method: 'POST', body: { payment: { amount: o.totalPi, memo: `استرجاع طلب ${o.id}`, metadata: { orderId: o.id, type: 'refund' }, uid: o.userUid } } });
      r.paymentId = pay.identifier; await saveOrder(o);
    }
    let txid = (pay.transaction || {}).txid;
    if (!txid) {
      const kp = StellarSdk.Keypair.fromSecret(APP_WALLET_SEED);
      const server = new StellarSdk.Horizon.Server(HORIZON);
      const acc = await server.loadAccount(kp.publicKey());
      const fee = await server.fetchBaseFee();
      const tx = new StellarSdk.TransactionBuilder(acc, { fee: String(fee), networkPassphrase: NET_PASS, timebounds: await server.fetchTimebounds(180) })
        .addOperation(StellarSdk.Operation.payment({ destination: pay.to_address, asset: StellarSdk.Asset.native(), amount: String(pay.amount) }))
        .addMemo(StellarSdk.Memo.text(pay.identifier)).build();
      tx.sign(kp);
      const result = await server.submitTransaction(tx); txid = result.hash || result.id;
    }
    if (!(pay.status || {}).developer_completed) await pi(`/payments/${r.paymentId}/complete`, { method: 'POST', body: { txid } });
    Object.assign(r, { status: 'refunded', txid, toAddress: pay.to_address, refundedAt: now(), lastError: null });
    o.status = 'refunded'; o.history.push({ s: 'refunded', t: now() }); await saveOrder(o);
  } catch (e) { r.status = 'requested'; r.lastError = e.message; await saveOrder(o); throw e; }
}

/* ---------- لوحة الإدارة ---------- */
const recentOrders = () => listDocs(col('orders').orderBy('createdAt', 'desc').limit(300)).then(l => l.filter(o => !['pending_payment', 'cancelled'].includes(o.status)));
app.get('/api/admin/summary', needAdmin, ah(async (req, res) => {
  const [paid, prods, users] = await Promise.all([recentOrders(), col('products').count().get(), col('users').count().get()]);
  res.json({
    products: prods.data().count, orders: paid.length, newOrders: paid.filter(o => o.status === 'paid').length,
    revenuePi: round7(paid.filter(o => o.status !== 'refunded').reduce((s, o) => s + o.totalPi, 0)),
    refundsPending: paid.filter(o => o.refund.status === 'requested').length, users: users.data().count
  });
}));
app.get('/api/admin/orders', needAdmin, ah(async (req, res) => {
  const l = await recentOrders();
  const uids = [...new Set(l.map(o => o.userUid))];
  const snaps = uids.length ? await fdb.getAll(...uids.map(u => col('users').doc(u))) : [];
  const wallet = {}; snaps.forEach(s => { if (s.exists) wallet[s.id] = s.data().walletAddress; });
  res.json(l.map(o => ({ ...o, wallet: wallet[o.userUid] || null })));
}));
app.put('/api/admin/orders/:id/status', needAdmin, ah(async (req, res) => {
  const o = await getDoc('orders', req.params.id); const s = req.body.status;
  if (!o) throw fail(404, 'غير موجود');
  if (!['processing', 'shipped', 'delivered'].includes(s)) throw fail(400, 'حالة غير صالحة');
  if (!PAID_OK.includes(o.status)) throw fail(400, 'لا يمكن تغيير حالة هذا الطلب');
  o.status = s; o.history.push({ s, t: now() }); if (s === 'delivered') o.deliveredAt = now();
  await saveOrder(o); res.json({ ok: true });
}));
app.post('/api/admin/orders/:id/refund', needAdmin, ah(async (req, res) => {
  const o = await getDoc('orders', req.params.id);
  if (!o || o.refund.status !== 'requested') throw fail(400, 'لا يوجد طلب استرجاع معلّق');
  o.refund.adminNote = String(req.body.note || '').slice(0, 300);
  if (req.body.action === 'reject') { o.refund.status = 'rejected'; o.refund.decidedAt = now(); await saveOrder(o); return res.json({ ok: true }); }
  if (req.body.action !== 'approve') throw fail(400, 'إجراء غير صالح');
  await processRefund(o); res.json({ ok: true, txid: o.refund.txid });
}));
app.get('/api/admin/settings', needAdmin, (req, res) => res.json(settings));
app.put('/api/admin/settings', needAdmin, ah(async (req, res) => {
  const b = req.body || {};
  settings = { shopPhone: String(b.shopPhone || '').slice(0, 20), refundDays: Math.max(1, Math.min(60, parseInt(b.refundDays) || 7)), deliveryNote: String(b.deliveryNote || '').slice(0, 300) };
  await fdb.doc('config/settings').set(settings); res.json(settings);
}));

/* ---------- تنظيف الطلبات المعلّقة ---------- */
setInterval(async () => {
  try {
    const l = await listDocs(col('orders').where('status', '==', 'pending_payment'));
    for (const o of l.filter(x => now() - x.createdAt > 2 * 36e5)) {
      try {
        if (o.paymentId) {
          const pay = await pi('/payments/' + o.paymentId);
          if ((pay.transaction || {}).txid) { await finalizePayment(o.paymentId, pay.transaction.txid); continue; }
          if (!(pay.status || {}).cancelled && !(pay.status || {}).user_cancelled) await pi(`/payments/${o.paymentId}/cancel`, { method: 'POST' }).catch(() => {});
        }
        await cancelOrder(o.id);
      } catch (e) { /* إعادة المحاولة لاحقاً */ }
    }
  } catch (e) { console.error('cleanup', e.message); }
}, 10 * 60 * 1000);

app.use((req, res) => res.status(404).json({ error: 'غير موجود' }));
loadSettings().then(() => app.listen(PORT, () => console.log(`سراج فون على المنفذ ${PORT} | ${SANDBOX ? 'Sandbox' : 'Mainnet'}`)))
  .catch(e => { console.error('فشل الاتصال بـ Firebase:', e.message); process.exit(1); });
