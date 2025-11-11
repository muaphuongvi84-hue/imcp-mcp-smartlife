// imcp-mcp-smartlife.js
const express = require('express');
const axios = require('axios');
const crypto = require('crypto');
const fs = require('fs');
const path = require('path');

require('dotenv').config();

const app = express();
app.use(express.json());

const TUYA_BASE = process.env.TUYA_BASE || 'https://openapi.tuyaeu.com';
const CLIENT_ID = process.env.TUYA_CLIENT_ID || '';
const CLIENT_SECRET = process.env.TUYA_CLIENT_SECRET || '';
const PORT = process.env.PORT || 3000;
const DEVICE_DB_FILE = process.env.DEVICE_DB_FILE || path.join(__dirname, 'device_map.json');

let accessToken = null;
let tokenExpireAt = 0;
function nowTs() { return Date.now().toString(); }
function makeSign(clientId, clientSecret, t, accessTokenStr='') {
  const str = clientId + accessTokenStr + t;
  return crypto.createHmac('sha256', clientSecret).update(str).digest('hex').toUpperCase();
}

async function fetchAccessToken() {
  if (!CLIENT_ID || !CLIENT_SECRET) throw new Error('Missing TUYA_CLIENT_ID/SECRET');
  const t = nowTs();
  const sign = makeSign(CLIENT_ID, CLIENT_SECRET, t);
  const url = `${TUYA_BASE}/v1.0/token?grant_type=1`;
  const headers = { 'client_id': CLIENT_ID, 't': t, 'sign': sign, 'sign_method': 'HMAC-SHA256' };
  const r = await axios.get(url, { headers });
  accessToken = r.data.result.access_token;
  tokenExpireAt = Date.now() + r.data.result.expire_time * 1000 - 60000;
  console.log('Tuya token obtained. Expires in (s):', r.data.result.expire_time);
}

async function ensureToken() {
  if (!accessToken || Date.now() >= tokenExpireAt) await fetchAccessToken();
  return accessToken;
}

function readDeviceMap() {
  try { return JSON.parse(fs.readFileSync(DEVICE_DB_FILE,'utf8') || '{}'); }
  catch(e){ return {}; }
}
function writeDeviceMap(map) { fs.writeFileSync(DEVICE_DB_FILE, JSON.stringify(map, null, 2)); }

async function resolveDeviceId(userId, deviceNameOrId) {
  const map = readDeviceMap();
  if (map[userId] && map[userId][deviceNameOrId]) return map[userId][deviceNameOrId];
  if (/^[a-z0-9]{8,}$/i.test(deviceNameOrId)) return deviceNameOrId;
  return null;
}

async function sendTuyaCommand(deviceId, commands) {
  await ensureToken();
  const t = nowTs();
  const sign = makeSign(CLIENT_ID, CLIENT_SECRET, t, accessToken);
  const url = `${TUYA_BASE}/v1.0/iot-03/devices/${deviceId}/commands`;
  const headers = {
    'client_id': CLIENT_ID,
    'access_token': accessToken,
    't': t,
    'sign': sign,
    'sign_method': 'HMAC-SHA256',
    'Content-Type': 'application/json'
  };
  const r = await axios.post(url, { commands }, { headers });
  return r.data;
}

app.post('/mcps', async (req, res) => {
  const rpc = req.body;
  if (!rpc || !rpc.method) return res.status(400).json({ error: 'invalid json-rpc' });
  try {
    if (rpc.method === 'tools/list') {
      const tools = [{
        name: "smartlife.control",
        description: "Control SmartLife/Tuya devices",
        version: "1.0.0",
        input_schema: {
          type: "object",
          properties: {
            user_id: { type: "string" },
            device: { type: "string" },
            action: { type: "string", enum: ["turn_on","turn_off","set_brightness","set_color_temp"] },
            value: {}
          },
          required: ["user_id","device","action"]
        }
      }];
      return res.json({ jsonrpc: "2.0", id: rpc.id || null, result: tools });
    }

    if (rpc.method === 'tools/call') {
      const { name, input } = rpc.params || {};
      if (name !== 'smartlife.control') return res.json({ jsonrpc:"2.0", id: rpc.id || null, error:{ code:404, message:'Tool not found' }});
      const { user_id, device, action, value } = input || {};
      if (!user_id || !device || !action) return res.json({ jsonrpc:"2.0", id: rpc.id || null, error:{ code:400, message:'Missing params' }});

      const deviceId = await resolveDeviceId(user_id, device);
      if (!deviceId) return res.json({ jsonrpc:"2.0", id: rpc.id || null, error:{ code:404, message:'Device not found' }});

      let commands;
      if (action === 'turn_on') commands = [{ code: 'switch_1', value: true }];
      else if (action === 'turn_off') commands = [{ code: 'switch_1', value: false }];
      else if (action === 'set_brightness') commands = [{ code: 'brightness', value: Number(value) }];
      else if (action === 'set_color_temp') commands = [{ code: 'color_temp', value: Number(value) }];
      else return res.json({ jsonrpc:"2.0", id: rpc.id || null, error:{ code:400, message:'Unknown action' }});

      try {
        const tuyaRes = await sendTuyaCommand(deviceId, commands);
        const ok = !!tuyaRes.success;
        return res.json({ jsonrpc:"2.0", id: rpc.id || null, result: { success: ok, message: ok ? 'OK' : 'Tuya false', device_id: deviceId }});
      } catch (err) {
        console.error('Tuya error:', err.response?.data || err.message);
        return res.json({ jsonrpc:"2.0", id: rpc.id || null, error:{ code:500, message:'Tuya API error', details: err.response?.data || err.message }});
      }
    }

    return res.json({ jsonrpc:"2.0", id: rpc.id || null, error:{ code:-32601, message:'Method not found' }});
  } catch(e) {
    console.error('Server error', e);
    return res.json({ jsonrpc:"2.0", id: rpc.id || null, error:{ code:-32000, message:'Server error', data: e.message }});
  }
});

// Optional admin endpoints for mapping devices (for quick testing)
app.get('/admin/device-map', (req,res) => res.json(readDeviceMap()));
app.post('/admin/device-map', (req,res) => {
  const { user_id, device_name, device_id } = req.body;
  if (!user_id || !device_name || !device_id) return res.status(400).json({ error: 'user_id, device_name, device_id required' });
  const map = readDeviceMap();
  map[user_id] = map[user_id] || {};
  map[user_id][device_name] = device_id;
  writeDeviceMap(map);
  return res.json({ ok:true });
});

app.listen(PORT, () => console.log(`MCP SmartLife server running on port ${PORT} at /mcps`));

