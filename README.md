<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <title>批量 IP 在线解析（本地版）</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 text-gray-800">
  <div class="max-w-6xl mx-auto p-6">
    <h1 class="text-2xl font-bold mb-4">批量 IP 在线解析</h1>
    <p class="text-sm text-gray-600 mb-4">支持 IPv4 / IPv6，单次最多 6000 条。数据来源：ip-api.com 免费接口。</p>

    <textarea id="ipInput" class="w-full h-40 p-3 border rounded-xl" placeholder="每行一个 IP，或用空格/逗号分隔"></textarea>
    <div class="mt-3 flex gap-2">
      <input type="file" id="fileInput" accept=".txt,.csv" class="text-sm" />
      <button id="parseBtn" class="px-4 py-2 bg-black text-white rounded-xl">开始解析</button>
      <button id="clearBtn" class="px-4 py-2 bg-white border rounded-xl">清空</button>
    </div>

    <div class="mt-4">
      <div class="w-full bg-gray-200 rounded-full h-3 overflow-hidden">
        <div id="bar" class="bg-gray-800 h-3 w-0"></div>
      </div>
      <div id="progressText" class="text-sm mt-1">等待开始…</div>
    </div>

    <div class="mt-6 overflow-auto bg-white rounded-xl border">
      <table class="min-w-full text-sm">
        <thead class="bg-gray-100">
          <tr>
            <th class="px-3 py-2 border-b">IP</th>
            <th class="px-3 py-2 border-b">状态</th>
            <th class="px-3 py-2 border-b">国家</th>
            <th class="px-3 py-2 border-b">省/州</th>
            <th class="px-3 py-2 border-b">城市</th>
            <th class="px-3 py-2 border-b">ISP</th>
            <th class="px-3 py-2 border-b">组织</th>
            <th class="px-3 py-2 border-b">纬度</th>
            <th class="px-3 py-2 border-b">经度</th>
            <th class="px-3 py-2 border-b">错误信息</th>
          </tr>
        </thead>
        <tbody id="tbody"></tbody>
      </table>
    </div>
  </div>

<script>
const MAX_IPS = 6000;
const BATCH_SIZE = 100;
const IPAPI_URL = 'http://ip-api.com/batch?fields=status,message,query,country,regionName,city,isp,org,lat,lon';
const ta = document.getElementById('ipInput');
const fileInput = document.getElementById('fileInput');
const parseBtn = document.getElementById('parseBtn');
const clearBtn = document.getElementById('clearBtn');
const bar = document.getElementById('bar');
const progressText = document.getElementById('progressText');
const tbody = document.getElementById('tbody');

function setProgress(done, total) {
  const pct = total ? Math.round(done * 100 / total) : 0;
  bar.style.width = pct + '%';
  progressText.textContent = total ? `已完成 ${done}/${total}（${pct}%）` : '等待开始…';
}

function isLikelyIP(s) {
  const t = s.trim().replace(/^\[/, '').replace(/\]$/, '');
  const isIPv4 = /^(?:\d{1,3}\.){3}\d{1,3}$/.test(t);
  const isIPv6 = /:/.test(t) && /^[0-9a-fA-F:]+$/.test(t);
  return isIPv4 || isIPv6;
}

function normalizeIPs(input) {
  const list = input.split(/\r?\n|,|\s+/).map(s => s.trim()).filter(Boolean);
  const uniq = Array.from(new Set(list.filter(isLikelyIP)));
  return uniq;
}

async function queryBatch(ips) {
  const resp = await fetch(IPAPI_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(ips)
  });
  return await resp.json();
}

async function lookupIPs(ips) {
  const results = [];
  for (let i = 0; i < ips.length; i += BATCH_SIZE) {
    const batch = ips.slice(i, i + BATCH_SIZE);
    try {
      const res = await queryBatch(batch);
      results.push(...res);
    } catch (e) {
      results.push(...batch.map(ip => ({ query: ip, status: 'fail', message: e.message })));
    }
    setProgress(Math.min(ips.length, i + batch.length), ips.length);
    await new Promise(r => setTimeout(r, 750));
  }
  return results;
}

function renderTable(rows) {
  tbody.innerHTML = '';
  rows.forEach(r => {
    const tr = document.createElement('tr');
    tr.innerHTML = `
      <td class="px-3 py-2 border-b">${r.query || ''}</td>
      <td class="px-3 py-2 border-b">${r.status || ''}</td>
      <td class="px-3 py-2 border-b">${r.country || ''}</td>
      <td class="px-3 py-2 border-b">${r.regionName || ''}</td>
      <td class="px-3 py-2 border-b">${r.city || ''}</td>
      <td class="px-3 py-2 border-b">${r.isp || ''}</td>
      <td class="px-3 py-2 border-b">${r.org || ''}</td>
      <td class="px-3 py-2 border-b">${r.lat ?? ''}</td>
      <td class="px-3 py-2 border-b">${r.lon ?? ''}</td>
      <td class="px-3 py-2 border-b text-red-600">${r.message || ''}</td>
    `;
    tbody.appendChild(tr);
  });
}

parseBtn.addEventListener('click', async () => {
  const ips = normalizeIPs(ta.value);
  if (!ips.length) return alert('请输入有效 IP');
  if (ips.length > MAX_IPS) return alert(`单次最多 ${MAX_IPS} 个 IP`);

  setProgress(0, 0);
  tbody.innerHTML = '';
  const results = await lookupIPs(ips);
  renderTable(results);
  progressText.textContent = `完成：共 ${ips.length} 条`;
});

clearBtn.addEventListener('click', () => {
  ta.value = '';
  tbody.innerHTML = '';
  setProgress(0, 0);
});

fileInput.addEventListener('change', async (e) => {
  const file = e.target.files[0];
  if (!file) return;
  const text = await file.text();
  ta.value += (ta.value ? '\n' : '') + text;
});
</script>
</body>
</html>
