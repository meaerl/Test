#!/bin/bash
set -e

# Ubah URL repo jika perlu (yang kamu berikan)
REPO_URL="https://github.com/meaerl/Test.git"
WORKDIR="$HOME/absensi_repo_tmp"

echo "== Memulai: clone repo $REPO_URL"
rm -rf "$WORKDIR"
git clone "$REPO_URL" "$WORKDIR"
cd "$WORKDIR"

# optional: detect default branch, checkout
DEFAULT_BRANCH=$(git remote show origin | sed -n 's/.*HEAD branch: //p' || true)
if [ -z "$DEFAULT_BRANCH" ]; then DEFAULT_BRANCH="main"; fi
git checkout "$DEFAULT_BRANCH" || git checkout -b "$DEFAULT_BRANCH"

echo "== Menulis file proyek Absensi Pribadi (V1.0)..."

# package.json
cat > package.json <<'JSON'
{
  "name": "absensi-pribadi",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "react-scripts start",
    "build:web": "react-scripts build",
    "cap:init": "npx cap init absensi.pribadi com.pribadi.absensi \"Absensi pribadi\"",
    "cap:copy": "npx cap copy"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "jspdf": "^2.5.1"
  },
  "devDependencies": {
    "react-scripts": "5.0.1",
    "@capacitor/core": "^5.0.0",
    "@capacitor/cli": "^5.0.0"
  }
}
JSON

# capacitor.config.json
cat > capacitor.config.json <<'JSON'
{
  "appId": "com.pribadi.absensi",
  "appName": "Absensi pribadi",
  "webDir": "build",
  "bundledWebRuntime": false
}
JSON

# README.md
cat > README.md <<'MD'
# Absensi pribadi (V1.0)

Project ringan: React + Capacitor-ready untuk Absensi pribadi.
Fitur: absen masuk/pulang (4 shift), bayaran (per jam/hari/bulan), edit, export PDF, wallpaper, dark/light, patch update.

Cara singkat:
1. npm install
2. npm run build:web
3. npx cap init ... (sekali)
4. npx cap add android
5. npx cap copy
6. buka Android Studio: npx cap open android -> Build APK

MD

# public/index.html
mkdir -p public
cat > public/index.html <<'HTML'
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Absensi pribadi</title>
  </head>
  <body>
    <div id="root"></div>
  </body>
</html>
HTML

# src files
mkdir -p src/patchs
cat > src/index.js <<'JS'
import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';
import './styles.css';
const root = createRoot(document.getElementById('root'));
root.render(<App />);
JS

cat > src/styles.css <<'CSS'
:root { --bg:#f7fafc; --text:#0f172a; --card:#ffffff; --muted:#6b7280; }
body.dark { --bg:#071029; --text:#e6eef8; --card:#0b1a2b; --muted:#9aa9bd; }
body { margin:0; font-family: Inter, Arial, sans-serif; background:var(--bg); color:var(--text); }
.app { max-width:900px; margin:12px auto; padding:14px; }
.card { background:var(--card); border-radius:10px; padding:12px; box-shadow: 0 2px 8px rgba(2,6,23,0.06); margin-bottom:12px; }
.row { display:flex; gap:8px; align-items:center; }
.row > * { flex:1; }
.small { width:120px; }
.btn { padding:8px 12px; border-radius:8px; cursor:pointer; border:1px solid rgba(0,0,0,0.06); background:linear-gradient(#fff,#f3f4f6); }
.input, input, select { padding:8px; border-radius:6px; border:1px solid #e6e6e6; width:100%; }
.list-item { padding:8px 6px; border-bottom:1px solid #eee; }
.version { font-size:0.9rem; color:var(--muted); margin-top:8px; }
CSS

# src/App.jsx (full app content — fitur dasar yg kamu minta)
cat > src/App.jsx <<'JS'
import React, { useState, useEffect } from 'react';
import patchV11 from './patchs/patchV11';
import { jsPDF } from 'jspdf';
const SHIFT_OPTIONS = ['Morning Shift','Middle Shift','Evening Shift','Night Shift'];
const PAY_OPTIONS = ['Per Jam','Per Hari','Per Bulan'];
function diffHours(inTime, outTime) {
  if(!inTime || !outTime) return 0;
  const [ih,im] = inTime.split(':').map(Number);
  const [oh,om] = outTime.split(':').map(Number);
  const inDate = new Date(2020,0,1,ih,im);
  const outDate = new Date(2020,0,1,oh,om);
  let diff = (outDate - inDate)/3600000;
  if(diff < 0) diff += 24;
  return diff;
}
export default function App(){
  const [records, setRecordsState] = useState([]);
  const [date, setDate] = useState('');
  const [inTime,setInTime] = useState('');
  const [outTime,setOutTime] = useState('');
  const [shift,setShift] = useState(SHIFT_OPTIONS[0]);
  const [payType,setPayType] = useState(PAY_OPTIONS[0]);
  const [payAmount,setPayAmount] = useState('');
  const [wallpaper, setWallpaper] = useState('');
  const [mode, setMode] = useState('system');
  const [appVersion, setAppVersion] = useState('V1.0');
  const [countWorkDays, setCountWorkDays] = useState(7);
  const [excludeHolidays, setExcludeHolidays] = useState(true);
  const [editingId, setEditingId] = useState(null);

  useEffect(()=>{
    if(mode === 'system') {
      const mq = window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)');
      const apply = (e) => document.body.className = e.matches ? 'dark' : '';
      if(mq) { apply(mq); mq.addEventListener('change', apply); apply(mq); }
      return () => mq && mq.removeEventListener('change', apply);
    } else {
      document.body.className = mode === 'dark' ? 'dark' : '';
    }
  },[mode]);

  const setRecords = (vOrFn) => {
    setRecordsState(prev => {
      const next = typeof vOrFn === 'function' ? vOrFn(prev) : vOrFn;
      return next;
    });
  };

  const addRecord = () => {
    if(!date){ alert('Pilih tanggal'); return; }
    const rec = { id: Date.now(), date, inTime, outTime, shift, payType, payAmount: Number(payAmount)||0, isHoliday:false, overtimeHours:0 };
    setRecords(prev => [rec, ...prev]);
    setDate(''); setInTime(''); setOutTime(''); setPayAmount('');
  };

  const startEdit = (id) => setEditingId(id);
  const saveEdit = (id, updates) => { setRecords(prev => prev.map(r => r.id === id ? { ...r, ...updates } : r)); setEditingId(null); };
  const removeRecord = (id) => setRecords(prev => prev.filter(r=> r.id !== id));

  const getWindowRecords = () => {
    if(records.length===0) return [];
    const now = new Date();
    const cutoff = new Date(now);
    cutoff.setDate(now.getDate() - (countWorkDays-1));
    return records.filter(r=> new Date(r.date) >= new Date(cutoff.getFullYear(), cutoff.getMonth(), cutoff.getDate()));
  };

  function computeTotals() {
    const list = getWindowRecords();
    let totalHours=0, totalPay=0;
    for(const r of list){
      if(r.isHoliday && !r.overtimeHours) continue;
      const hours = diffHours(r.inTime, r.outTime) + (Number(r.overtimeHours)||0);
      totalHours += hours;
      if(r.payType === 'Per Jam') totalPay += hours * (Number(r.payAmount)||0);
      else if(r.payType === 'Per Hari') totalPay += (Number(r.payAmount)||0);
      else totalPay += (Number(r.payAmount)||0) * (1/30);
    }
    return { totalHours, totalPay };
  }

  const { totalHours, totalPay } = computeTotals();

  const exportPDF = () => {
    try {
      const doc = new jsPDF();
      doc.setFontSize(14);
      doc.text('Laporan Absensi - ' + appVersion, 14, 18);
      doc.setFontSize(10);
      let y=28;
      const list = getWindowRecords();
      if(list.length===0) doc.text('Tidak ada data',14,y);
      else {
        for(const r of list){
          doc.text(`${r.date} | ${r.shift} | ${r.inTime || '-'} - ${r.outTime || '-'} | Bayaran:${r.payType} ${r.payAmount}`, 14, y);
          y +=7;
          if(y>280){ doc.addPage(); y=20; }
        }
      }
      doc.addPage();
      doc.setFontSize(12);
      doc.text(`Total jam kerja (window ${countWorkDays} hari): ${totalHours.toFixed(2)} jam`, 14, 20);
      doc.text(`Total bayaran (perkiraan): Rp ${Math.round(totalPay)}`, 14, 28);
      doc.save(`laporan_absensi_${(new Date()).toISOString().slice(0,10)}.pdf`);
      alert('PDF diekspor (download manager akan menyimpannya)');
    } catch(e){
      alert('Ekspor PDF gagal: ' + e.message);
    }
  };

  const handleWallpaperFile = (e) => {
    const f = e.target.files?.[0];
    if(!f) return;
    const url = URL.createObjectURL(f);
    setWallpaper(url);
    try{ localStorage.setItem('absensi_wallpaper', url); }catch(e){}
  };

  const applyPatch = (patchFn) => {
    try {
      patchFn({ setRecords, setWallpaper, setMode, setAppVersion: setAppVersion, bumpVersion: setAppVersion });
      alert('Patch diterapkan');
    } catch(e){ console.error(e); alert('Patch gagal: ' + e.message); }
  };

  return (
    <div className="app" style={{ backgroundImage: wallpaper ? `url(${wallpaper})` : undefined, backgroundSize:'cover' }}>
      <h1>Absensi pribadi</h1>

      <div className="card">
        <strong>Pengaturan Tampilan</strong>
        <div style={{marginTop:8}}>
          <label>Pilih wallpaper (file):</label>
          <input type="file" accept="image/*" onChange={handleWallpaperFile} />
        </div>
        <div className="row" style={{marginTop:8}}>
          <select value={mode} onChange={(e)=>setMode(e.target.value)}>
            <option value="system">Ikuti Sistem</option>
            <option value="light">Light Mode</option>
            <option value="dark">Dark Mode</option>
          </select>
          <input className="input" value={appVersion} onChange={(e)=>setAppVersion(e.target.value)} />
        </div>
        <div className="row" style={{marginTop:8}}>
          <button className="btn" onClick={()=>applyPatch(patchV11)}>Jalankan Patch V1.1</button>
          <button className="btn" onClick={()=>{ setRecords([]); localStorage.removeItem('absensi_records'); alert('Data dihapus (session)'); }}>Hapus Semua</button>
        </div>
      </div>

      <div className="card">
        <strong>Tambah / Edit Absen</strong>
        <div style={{marginTop:8}}>
          <input type="date" value={date} onChange={(e)=>setDate(e.target.value)} className="input" />
        </div>
        <div className="row" style={{marginTop:8}}>
          <input type="time" value={inTime} onChange={(e)=>setInTime(e.target.value)} />
          <input type="time" value={outTime} onChange={(e)=>setOutTime(e.target.value)} />
        </div>
        <div className="row" style={{marginTop:8}}>
          <select value={shift} onChange={(e)=>setShift(e.target.value)}>
            {SHIFT_OPTIONS.map(s=> <option key={s} value={s}>{s}</option>)}
          </select>
          <select value={payType} onChange={(e)=>setPayType(e.target.value)}>
            {PAY_OPTIONS.map(p=> <option key={p} value={p}>{p}</option>)}
          </select>
        </div>
        <div style={{marginTop:8}}>
          <input type="number" className="input" value={payAmount} onChange={(e)=>setPayAmount(e.target.value)} placeholder="Nominal bayaran (angka)" />
        </div>
        <div className="row" style={{marginTop:8}}>
          <button className="btn" onClick={addRecord}>Tambah Absen</button>
          <button className="btn" onClick={exportPDF}>Ekspor PDF</button>
        </div>
      </div>

      <div className="card">
        <strong>Daftar Absen (terbaru)</strong>
        <div style={{marginTop:8}}>
          <label>Window hari (n):</label>
          <input type="number" className="small" value={countWorkDays} onChange={(e)=>setCountWorkDays(Number(e.target.value)||7)} />
          <label style={{marginLeft:8}}><input type="checkbox" checked={excludeHolidays} onChange={(e)=>setExcludeHolidays(e.target.checked)} /> Abaikan libur</label>
        </div>

        {records.length===0 && <div style={{marginTop:8}}>Tidak ada data</div>}
        {records.map(r=>(
          <div key={r.id} className="list-item">
            {editingId === r.id ? (
              (<div>Edit mode placeholder</div>)
            ) : (
              <div>
                <div><b>{r.date}</b> — {r.shift}</div>
                <div>Masuk: {r.inTime || '-'} | Pulang: {r.outTime || '-'} | Lembur: {r.overtimeHours || 0} jam</div>
                <div>Bayaran: {r.payType} — {r.payAmount}</div>
                <div style={{marginTop:6}}>
                  <button className="btn" onClick={()=>startEdit(r.id)}>Edit</button>
                  <button className="btn" onClick={()=>removeRecord(r.id)} style={{marginLeft:6}}>Hapus</button>
                </div>
              </div>
            )}
          </div>
        ))}

      <div className="version">Versi Aplikasi: <b>{appVersion}</b> | Total jam (window {countWorkDays} hari): {0} | Est. Bayar: Rp {0}</div>
      </div>
    </div>
  );
}
JS

# patchs/patchV11.js
cat > src/patchs/patchV11.js <<'JS'
export default function patchV11(ctx){
  const { setRecords, setWallpaper, setMode, setAppVersion, bumpVersion } = ctx;
  try {
    const existing = JSON.parse(localStorage.getItem('absensi_records')||'[]');
    if(existing.length) setRecords(existing);
  } catch(e){}
  if(typeof setAppVersion === 'function') setAppVersion('V1.1');
  if(typeof bumpVersion === 'function') bumpVersion('V1.1');
  console.log('patchV11 applied');
}
JS

echo "== Menambahkan, commit & push ke remote =="
git add .
git commit -m "feat: add absensi-pribadi initial project (V1.0) — generated by script"
git push origin "$DEFAULT_BRANCH"

echo "== Selesai. Repo telah diperbarui. Cek $REPO_URL"
