const { default: makeWASocket, useMultiFileAuthState, DisconnectReason } = require('baileys');
const qrcode = require('qrcode-terminal');
const axios = require('axios');

const SMM_API_KEY = 'API_KEY';
const BASE_URL = 'BASE_URL';

const providerGroups = {
  
    'ABC': 'ABC's Jid',
    'DEF': 'DEF's Jid'
};

const normalizeStatus = s => s.toLowerCase().replace(/[_\s-]/g, '');

// Fetch order status
async function fetchOrderStatus(orderId) {
    try {
        const res = await axios.get(`${BASE_URL}/orders/${orderId}`, {
            headers: { 'Content-Type': 'application/json', 'X-Api-Key': SMM_API_KEY }
        });
        if (res.data?.data) return res.data.data;
    } catch (err) {
        return { error: true, id: orderId };
    }
}

// Start WhatsApp bot
async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info');
    const sock = makeWASocket({ auth: state });

    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect, qr } = update;
        if (qr) qrcode.generate(qr, { small: true });
        if (connection === 'open') console.log('✅ Bot connected successfully!');
        else if (connection === 'close') {
            const shouldReconnect = lastDisconnect?.error?.output?.statusCode !== DisconnectReason.loggedOut;
            console.log(`⚠ Connection closed. Reconnecting: ${shouldReconnect}`);
            if (shouldReconnect) startBot();
        }
    });

    sock.ev.on('messages.upsert', async ({ messages }) => {
        const msg = messages[0];
        if (!msg.key.fromMe && msg.message) {
            const jid = msg.key.remoteJid;
            const providerGroupJIDs = Object.values(providerGroups);
            if (providerGroupJIDs.includes(jid)) return;

            const text = msg.message.conversation ||
                msg.message.extendedTextMessage?.text ||
                msg.message.imageMessage?.caption ||
                msg.message.videoMessage?.caption ||
                '';
            if (!text) return;

            const lowered = text.toLowerCase();
            const actions = ["cancel", "partial", "speed", "refill"];
            const action = actions.find(a => lowered.includes(a));

            const fakeCompleted = /fake[_\s-]*complete(d)?/.test(lowered);
            const orderIds = text.match(/\b\d+\b/g) || [];

            // Check if order IDs sent without valid action
            if (orderIds.length > 0 && !action && !fakeCompleted) {
                await sock.sendMessage(jid, { 
                    text: `Please enter valid action - Order ID speed / cancel / partial / refill / fake completed \nExample: 12345 cancel or 678910 speed`
                });
                return;
            }

            if (orderIds.length === 0) return;

            const batchMessages = {};
            const customerBatch = {};

            for (const orderId of orderIds) {
                const order = await fetchOrderStatus(orderId);

                // Handle invalid orders
                if (order.error) {
                    if (!customerBatch['invalid']) customerBatch['invalid'] = [];
                    customerBatch['invalid'].push(orderId);
                    continue;
                }

                const statusNorm = normalizeStatus(order.status);
                const providerJid = order.provider ? providerGroups[order.provider] : null;
                const failJid = providerGroups['FailGroup'];

                // Initialize arrays
                if (providerJid && !batchMessages[providerJid]) batchMessages[providerJid] = { fail: [], normal: [], fake: [] };
                if (!batchMessages[failJid]) batchMessages[failJid] = { fail: [], normal: [], fake: [] };

                // Handle fake completed logic
                if (fakeCompleted) {
                    if (["completed", "partial"].includes(statusNorm)) {
                        // ✅ Only send to provider if completed or partial
                        if (providerJid) batchMessages[providerJid].fake.push(order.external_id);
                        if (!customerBatch['sent for resolution']) customerBatch['sent for resolution'] = [];
                        customerBatch['sent for resolution'].push(orderId);
                    } else {
                        // Customer gets “in progress” message, provider gets nothing
                        if (!customerBatch['in progress']) customerBatch['in progress'] = [];
                        customerBatch['in progress'].push(orderId);
                    }
                    continue; // skip normal processing
                }

                // Provider logic
                if (statusNorm === 'fail') {
                    batchMessages[failJid].fail.push(order.id);
                } else if (action === 'refill' && statusNorm === 'completed') {
                    if (providerJid) batchMessages[providerJid].normal.push(order.external_id);
                } else if (providerJid && ["pending", "inprogress", "processing", "error"].includes(statusNorm)) {
                    batchMessages[providerJid].normal.push(order.external_id);
                }

                // Customer replies
                let custKey = '';
                if (["cancel", "partial", "speed"].includes(action)) {
                    if (["pending", "inprogress", "processing", "error", "fail"].includes(statusNorm)) {
                        custKey = `added for ${action}`;
                    } else if (statusNorm === "completed") custKey = 'already completed';
                    else if (statusNorm === "canceled" || statusNorm === "cancelled") custKey = 'already cancelled';
                    else if (statusNorm === "partial") custKey = 'already partial';
                } else if (action === 'refill') {
                    if (statusNorm === 'completed') custKey = 'sent for refilling';
                    else custKey = `${order.id} is in progress, cannot refill yet.`;
                }

                if (custKey) {
                    if (!customerBatch[custKey]) customerBatch[custKey] = [];
                    customerBatch[custKey].push(order.id);
                }
            }

            // Send provider messages
            for (const [groupJid, orders] of Object.entries(batchMessages)) {
                if (orders.normal.length > 0) await sock.sendMessage(groupJid, { text: `${orders.normal.join(', ')} ${action}` });
                if (orders.fail.length > 0) await sock.sendMessage(groupJid, { text: `${orders.fail.join(', ')} fail ${action}` });
                if (orders.fake.length > 0) await sock.sendMessage(groupJid, { text: `${orders.fake.join(', ')} fake completed` });
            }

            // Send customer messages
            for (const [key, ids] of Object.entries(customerBatch)) {
                let messageText = '';
                if (key === 'invalid') messageText = `${ids.join(', ')} — order doesn't belong to you or invalid order id`;
                else if (key === 'in progress') messageText = `${ids.join(', ')} — order is in progress, cannot mark as fake completed yet`;
                else messageText = `${ids.join(', ')} — ${key}`;
                await sock.sendMessage(jid, { text: messageText });
            }
        }
    });

    sock.ev.on('creds.update', saveCreds);
}

startBot();
