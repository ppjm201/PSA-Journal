<!DOCTYPE html>

<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="theme-color" content="#1e3d2f">
<meta name="description" content="Private symptom tracker for Psoriatic Arthritis — log daily pain, triggers, and see patterns over time.">
<title>PsA Journal</title>
<link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>📋</text></svg>">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Lora:wght@400;600;700;800&family=DM+Sans:opsz,wght@9..40,300;9..40,400;9..40,500;9..40,600&display=swap">

<script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>

<script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>

<script src="https://unpkg.com/@babel/standalone@7.24.7/babel.min.js"></script>

<style>
  * { box-sizing:border-box; margin:0; }
  html, body { margin:0; padding:0; background:#FAF7F2; -webkit-font-smoothing:antialiased; }
  body { font-family:'DM Sans',system-ui,sans-serif; color:#1e3d2f; }
  input[type=range] { width:100%; height:6px; border-radius:3px; outline:none; cursor:pointer; }
  button { font-family:inherit; }
  #root { min-height:100vh; }
  .loader { display:flex; align-items:center; justify-content:center; height:100vh; font-family:'Lora',serif; color:#2C5F4A; font-size:18px; }
</style>

</head>
<body>
<div id="root"><div class="loader">Loading your journal…</div></div>

<script type="text/babel" data-presets="react">
const { useState, useEffect, useRef } = React;

// ─── Data ─────────────────────────────────────────────────────────────────────

const JOINTS = [
  { id:"fingers_l", label:"Left Fingers" }, { id:"fingers_r", label:"Right Fingers" },
  { id:"wrist_l",   label:"Left Wrist" },   { id:"wrist_r",   label:"Right Wrist" },
  { id:"elbow_l",   label:"Left Elbow" },   { id:"elbow_r",   label:"Right Elbow" },
  { id:"shoulder_l",label:"Left Shoulder"}, { id:"shoulder_r",label:"Right Shoulder"},
  { id:"hip_l",     label:"Left Hip" },     { id:"hip_r",     label:"Right Hip" },
  { id:"knee_l",    label:"Left Knee" },    { id:"knee_r",    label:"Right Knee" },
  { id:"ankle_l",   label:"Left Ankle" },   { id:"ankle_r",   label:"Right Ankle" },
  { id:"toes_l",    label:"Left Toes" },    { id:"toes_r",    label:"Right Toes" },
  { id:"lower_back",label:"Lower Back" },   { id:"neck",      label:"Neck" },
];

const TRIGGERS = [
  { id:"dairy",         label:"Dairy",          emoji:"🧀" },
  { id:"gluten",        label:"Gluten/Wheat",   emoji:"🍞" },
  { id:"refined-carbs", label:"Refined Carbs",  emoji:"🥖" },
  { id:"processed",     label:"Processed Food", emoji:"🌭" },
  { id:"fried",         label:"Fried Foods",    emoji:"🍳" },
  { id:"sugar",         label:"Sugar/Sweets",   emoji:"🍬" },
  { id:"nightshades",   label:"Nightshades",    emoji:"🍅" },
  { id:"red-meat",      label:"Red Meat",       emoji:"🥩" },
  { id:"eggs",          label:"Eggs",           emoji:"🥚" },
];

const POSITIVES = [
  { id:"oily-fish",  label:"Oily Fish",       emoji:"🐟" },
  { id:"vegetables", label:"Fruits & Veg",    emoji:"🥦" },
  { id:"berries",    label:"Berries",         emoji:"🫐" },
  { id:"olive-oil",  label:"Olive Oil",       emoji:"🫒" },
  { id:"fermented",  label:"Fermented Foods", emoji:"🫙" },
  { id:"nuts",       label:"Nuts & Seeds",    emoji:"🥜" },
  { id:"turmeric",   label:"Turmeric",        emoji:"🌿" },
];

const ALL_FOODS = [...TRIGGERS, ...POSITIVES];
const EXERCISE_OPTS = ["None","Walking","Yoga / Stretching","Swimming","Cycling","Strength Training","Running","Other"];
const WEATHER_OPTS  = ["☀️ Sunny","⛅ Cloudy","🌧️ Rainy","🌬️ Windy","❄️ Cold","💧 Humid","🌡️ Hot"];

const PAIN_BG   = ["#E8F5E9","#C8E6C9","#FFF9C4","#FFE0B2","#FFCCBC","#FFCDD2"];
const PAIN_FG   = ["#388E3C","#2E7D32","#F9A825","#EF6C00","#BF360C","#B71C1C"];
const PAIN_LBLS = ["None","Very Mild","Mild","Moderate","Severe","Very Severe"];
const pb = v => PAIN_BG[v]   ?? PAIN_BG[0];
const pf = v => PAIN_FG[v]   ?? PAIN_FG[0];
const pl = v => PAIN_LBLS[v] ?? "";

const todayStr = () => new Date().toISOString().split("T")[0];

const emptyLog = () => ({
  date:todayStr(), overallPain:0, joints:{}, swelling:{},
  foods:[], exercise:"None", exerciseMins:0,
  alcohol:0, stress:0, sleep:7, weather:[], notes:"",
});

// ─── Storage: localStorage with in-memory fallback ────────────────────────────

const SKEY = "psa-tracker-v1";
let _cache = [];

function readLogs() {
  try {
    const raw = localStorage.getItem(SKEY);
    if (raw) { _cache = JSON.parse(raw); return _cache; }
  } catch (_) {}
  return _cache;
}

function writeLogs(logs) {
  _cache = logs;
  try { localStorage.setItem(SKEY, JSON.stringify(logs)); } catch (_) {}
  return true;
}

// ─── Small components ─────────────────────────────────────────────────────────

function Card({ title, emoji, children, style }) {
  return (
    <div style={{ background:"#fff", borderRadius:16, padding:"18px 20px", marginBottom:14,
      border:"1px solid #ECE5DC", boxShadow:"0 2px 14px rgba(44,95,74,.06)", ...style }}>
      {title && (
        <h3 style={{ margin:"0 0 14px", fontFamily:"'Lora',Georgia,serif", fontSize:17,
          fontWeight:700, color:"#1e3d2f", display:"flex", alignItems:"center", gap:8 }}>
          <span>{emoji}</span>{title}
        </h3>
      )}
      {children}
    </div>
  );
}

function Chip({ label, active, onClick, color="#2C5F4A" }) {
  return (
    <button onClick={onClick} style={{
      padding:"6px 14px", borderRadius:20,
      border:`1.5px solid ${active ? color : "#E2D9CE"}`,
      background: active ? color : "#fff",
      color: active ? "#fff" : "#7a7069",
      cursor:"pointer", fontSize:13,
      fontWeight: active ? 500 : 400, transition:"all .15s", whiteSpace:"nowrap",
    }}>{label}</button>
  );
}

function Stepper({ value, onChange, min=0, step=1, suffix="" }) {
  return (
    <div style={{ display:"flex", alignItems:"center", gap:10,
      background:"#F7F2EC", borderRadius:10, padding:"5px 14px", width:"fit-content" }}>
      <button onClick={() => onChange(Math.max(min, value - step))}
        style={{ width:30, height:30, borderRadius:7, border:"1px solid #E2D9CE",
          background:"#fff", cursor:"pointer", fontSize:18, fontWeight:700, color:"#2C5F4A", lineHeight:1 }}>−</button>
      <span style={{ minWidth:40, textAlign:"center", fontFamily:"'Lora',serif",
        fontWeight:700, fontSize:16, color:"#1e3d2f" }}>{value}{suffix}</span>
      <button onClick={() => onChange(value + step)}
        style={{ width:30, height:30, borderRadius:7, border:"1px solid #E2D9CE",
          background:"#fff", cursor:"pointer", fontSize:18, fontWeight:700, color:"#2C5F4A", lineHeight:1 }}>+</button>
    </div>
  );
}

function ScaleDots({ value, onChange }) {
  return (
    <div style={{ display:"flex", gap:6 }}>
      {[0,1,2,3,4,5].map(v => (
        <button key={v} onClick={() => onChange(v)} style={{
          width:36, height:36, borderRadius:9,
          border:`2px solid ${value===v ? pf(v) : "#E2D9CE"}`,
          background: value===v ? pb(v) : "#faf7f3",
          cursor:"pointer", fontWeight:700, fontSize:14,
          color: value===v ? pf(v) : "#b0a99f", transition:"all .12s",
        }}>{v}</button>
      ))}
    </div>
  );
}

// Simple SVG line chart (no external dependencies) ────────────────────────────
function SimpleLineChart({ data }) {
  if (!data.length) return null;
  const w = 320, h = 160, padL = 24, padR = 6, padT = 8, padB = 28;
  const maxX = Math.max(data.length - 1, 1);
  const xScale = i => padL + (i / maxX) * (w - padL - padR);
  const yScale = v => h - padB - (v / 5) * (h - padT - padB);
  const points = data.map((d, i) => `${xScale(i)},${yScale(d.pain)}`).join(" ");
  const showEvery = Math.max(1, Math.ceil(data.length / 7));

  return (
    <svg viewBox={`0 0 ${w} ${h}`} style={{ width:"100%", height:160, overflow:"visible" }}>
      {[0,1,2,3,4,5].map(y => (
        <g key={y}>
          <line x1={padL} y1={yScale(y)} x2={w-padR} y2={yScale(y)}
            stroke="#F0E8E0" strokeDasharray="3 3" />
          <text x={padL-4} y={yScale(y)+3} fontSize="9" fill="#9a9189" textAnchor="end">{y}</text>
        </g>
      ))}
      {data.map((d,i) => i % showEvery === 0 ? (
        <text key={i} x={xScale(i)} y={h-10} fontSize="9" fill="#9a9189" textAnchor="middle">{d.date}</text>
      ) : null)}
      <polyline points={points} fill="none" stroke="#2C5F4A" strokeWidth="2.5"
        strokeLinecap="round" strokeLinejoin="round" />
      {data.map((d,i) => (
        <circle key={i} cx={xScale(i)} cy={yScale(d.pain)} r="3.5" fill="#2C5F4A" />
      ))}
    </svg>
  );
}

// ─── Log Tab ─────────────────────────────────────────────────────────────────

function LogTab({ log, setLog, onSave, saveMsg }) {
  const [openJoint, setOpenJoint] = useState(null);

  const toggle = (field, id) => setLog(p => {
    const arr = p[field] || [];
    return { ...p, [field]: arr.includes(id) ? arr.filter(x=>x!==id) : [...arr, id] };
  });

  return (
    <div>
      <Card title="Overall Pain Today" emoji="🌡️">
        <div style={{ textAlign:"center", marginBottom:18 }}>
          <div style={{ fontSize:52, fontWeight:800, fontFamily:"'Lora',serif",
            color:pf(log.overallPain), lineHeight:1 }}>{log.overallPain}</div>
          <div style={{ fontSize:14, color:"#9a9189", marginTop:4 }}>{pl(log.overallPain)}</div>
        </div>
        <div style={{ display:"flex", alignItems:"center", gap:10 }}>
          <span style={{ fontSize:11, color:"#9a9189" }}>None</span>
          <input type="range" min={0} max={5} value={log.overallPain}
            onChange={e => setLog(p=>({...p, overallPain:+e.target.value}))}
            style={{ flex:1, accentColor:"#2C5F4A" }} />
          <span style={{ fontSize:11, color:"#9a9189" }}>Severe</span>
        </div>
      </Card>

      <Card title="Joint Symptoms" emoji="🦴">
        <p style={{ fontSize:13, color:"#9a9189", margin:"0 0 12px" }}>Tap a joint to rate pain (0–5) or mark as swollen</p>
        <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:8 }}>
          {JOINTS.map(j => {
            const pain    = log.joints[j.id] || 0;
            const swollen = log.swelling?.[j.id];
            const open    = openJoint === j.id;
            return (
              <div key={j.id} style={{ borderRadius:10, overflow:"hidden",
                border:`1.5px solid ${pain>0 ? pf(pain)+"50" : "#E2D9CE"}`,
                background:pb(pain) }}>
                <button onClick={() => setOpenJoint(open ? null : j.id)} style={{
                  width:"100%", background:"none", border:"none", cursor:"pointer",
                  padding:"9px 12px", display:"flex", justifyContent:"space-between", alignItems:"center" }}>
                  <span style={{ fontSize:12, fontWeight:600, color:"#1e3d2f" }}>{j.label}</span>
                  <div style={{ display:"flex", alignItems:"center", gap:5 }}>
                    {swollen && <span style={{ fontSize:9, background:"#3B82F6", color:"#fff", borderRadius:4, padding:"1px 5px" }}>swollen</span>}
                    {pain>0  && <span style={{ fontSize:12, fontWeight:800, color:pf(pain) }}>{pain}</span>}
                    <span style={{ fontSize:11, color:"#9a9189" }}>{open?"▲":"▼"}</span>
                  </div>
                </button>
                {open && (
                  <div style={{ padding:"0 12px 12px", borderTop:"1px solid #E2D9CE" }}>
                    <div style={{ display:"flex", justifyContent:"space-between", paddingTop:10, marginBottom:8 }}>
                      {[0,1,2,3,4,5].map(v => (
                        <button key={v}
                          onClick={() => setLog(p=>({...p, joints:{...p.joints,[j.id]:v}}))}
                          style={{ width:34, height:34, borderRadius:8,
                            border:`2px solid ${pain===v?pf(v):"transparent"}`,
                            background:pb(v), cursor:"pointer", fontWeight:800, fontSize:13, color:pf(v) }}>{v}</button>
                      ))}
                    </div>
                    <button onClick={() => setLog(p=>({...p, swelling:{...p.swelling,[j.id]:!p.swelling?.[j.id]}}))}
                      style={{ width:"100%", padding:"6px 0", borderRadius:7,
                        border:"1.5px solid #3B82F6",
                        background:swollen?"#3B82F6":"#fff", color:swollen?"#fff":"#3B82F6",
                        cursor:"pointer", fontSize:12, fontWeight:500 }}>
                      💧 {swollen ? "Swollen — tap to remove" : "Mark as Swollen"}
                    </button>
                  </div>
                )}
              </div>
            );
          })}
        </div>
      </Card>

      <Card title="Food & Diet" emoji="🍽️">
        <p style={{ fontSize:13, color:"#9a9189", margin:"0 0 8px" }}>Potential triggers eaten today</p>
        <div style={{ display:"flex", flexWrap:"wrap", gap:8, marginBottom:16 }}>
          {TRIGGERS.map(f => <Chip key={f.id} label={`${f.emoji} ${f.label}`} active={log.foods.includes(f.id)} onClick={()=>toggle("foods",f.id)} color="#B85C38"/>)}
        </div>
        <p style={{ fontSize:13, color:"#9a9189", margin:"0 0 8px" }}>Anti-inflammatory foods</p>
        <div style={{ display:"flex", flexWrap:"wrap", gap:8 }}>
          {POSITIVES.map(f => <Chip key={f.id} label={`${f.emoji} ${f.label}`} active={log.foods.includes(f.id)} onClick={()=>toggle("foods",f.id)} color="#2C5F4A"/>)}
        </div>
      </Card>

      <Card title="Exercise" emoji="🏃">
        <div style={{ display:"flex", flexWrap:"wrap", gap:8, marginBottom:14 }}>
          {EXERCISE_OPTS.map(e => <Chip key={e} label={e} active={log.exercise===e} onClick={()=>setLog(p=>({...p,exercise:e}))} color="#2C5F4A"/>)}
        </div>
        {log.exercise!=="None" && (
          <div style={{ display:"flex", alignItems:"center", gap:12 }}>
            <span style={{ fontSize:13, color:"#7a7069" }}>Duration:</span>
            <Stepper value={log.exerciseMins} onChange={v=>setLog(p=>({...p,exerciseMins:v}))} min={0} step={15} suffix="m"/>
          </div>
        )}
      </Card>

      <Card title="Alcohol" emoji="🍷">
        <div style={{ display:"flex", alignItems:"center", gap:14, flexWrap:"wrap" }}>
          <span style={{ fontSize:13, color:"#7a7069" }}>Units today:</span>
          <Stepper value={log.alcohol} onChange={v=>setLog(p=>({...p,alcohol:v}))} min={0}/>
          {log.alcohol>0 && <span style={{ fontSize:12, color:"#9a9189" }}>≈ {log.alcohol*8}g ethanol</span>}
        </div>
      </Card>

      <Card title="Wellbeing" emoji="🧘">
        <div style={{ marginBottom:18 }}>
          <div style={{ display:"flex", justifyContent:"space-between", marginBottom:8 }}>
            <span style={{ fontSize:13, color:"#7a7069" }}>Stress level</span>
            <span style={{ fontSize:13, fontWeight:600, color:"#2C5F4A" }}>
              {["–","Very Low","Low","Moderate","High","Very High"][log.stress] ?? "–"}
            </span>
          </div>
          <ScaleDots value={log.stress} onChange={v=>setLog(p=>({...p,stress:v}))}/>
        </div>
        <div>
          <div style={{ display:"flex", justifyContent:"space-between", marginBottom:8 }}>
            <span style={{ fontSize:13, color:"#7a7069" }}>Hours sleep</span>
            <span style={{ fontSize:13, fontWeight:600, color:"#2C5F4A" }}>{log.sleep}h</span>
          </div>
          <input type="range" min={3} max={12} step={0.5} value={log.sleep}
            onChange={e=>setLog(p=>({...p,sleep:+e.target.value}))}
            style={{ width:"100%", accentColor:"#2C5F4A" }}/>
          <div style={{ display:"flex", justifyContent:"space-between" }}>
            <span style={{ fontSize:11, color:"#9a9189" }}>3h</span>
            <span style={{ fontSize:11, color:"#9a9189" }}>12h</span>
          </div>
        </div>
      </Card>

      <Card title="Weather Conditions" emoji="🌤️">
        <div style={{ display:"flex", flexWrap:"wrap", gap:8 }}>
          {WEATHER_OPTS.map(w => <Chip key={w} label={w} active={log.weather.includes(w)} onClick={()=>toggle("weather",w)} color="#4A90B8"/>)}
        </div>
      </Card>

      <Card title="Notes" emoji="📝">
        <textarea value={log.notes} onChange={e=>setLog(p=>({...p,notes:e.target.value}))}
          placeholder="Anything else today? New medications, stress, travel, how you felt generally..."
          style={{ width:"100%", minHeight:80, border:"1.5px solid #E2D9CE", borderRadius:10,
            padding:12, fontFamily:"inherit", fontSize:13, color:"#1e3d2f",
            background:"#FAF7F2", resize:"vertical", outline:"none", lineHeight:1.6 }}/>
      </Card>

      <div style={{ paddingBottom:36 }}>
        <button onClick={onSave} style={{
          width:"100%", padding:"15px 0",
          background: saveMsg==="Saved ✓" ? "#388E3C" : "#2C5F4A",
          color:"#fff", border:"none", borderRadius:13, fontSize:16,
          fontFamily:"'Lora',serif", fontWeight:700, cursor:"pointer",
          letterSpacing:.5, boxShadow:"0 4px 16px rgba(44,95,74,.35)", transition:"background .3s",
        }}>{saveMsg || "Save Today's Entry"}</button>
      </div>
    </div>
  );
}

// ─── History Tab ──────────────────────────────────────────────────────────────

function HistoryTab({ logs, onExport, onImport }) {
  const fileRef = useRef(null);

  if (!logs.length) return (
    <div style={{ padding:"60px 0", textAlign:"center" }}>
      <div style={{ fontSize:52, marginBottom:14 }}>📖</div>
      <div style={{ fontFamily:"'Lora',serif", fontSize:20, color:"#1e3d2f", marginBottom:8 }}>No entries yet</div>
      <div style={{ fontSize:14, color:"#9a9189", lineHeight:1.7 }}>Use the Today tab to log your first entry.</div>
    </div>
  );

  return (
    <div>
      <div style={{ display:"flex", gap:10, marginBottom:16 }}>
        <button onClick={onExport} style={{ flex:1, padding:"10px 0", borderRadius:10,
          border:"1.5px solid #2C5F4A", background:"#fff", color:"#2C5F4A",
          cursor:"pointer", fontSize:13, fontWeight:600 }}>⬇ Export backup</button>
        <button onClick={()=>fileRef.current?.click()} style={{ flex:1, padding:"10px 0", borderRadius:10,
          border:"1.5px solid #E2D9CE", background:"#fff", color:"#7a7069",
          cursor:"pointer", fontSize:13, fontWeight:600 }}>⬆ Restore backup</button>
        <input ref={fileRef} type="file" accept=".json" style={{ display:"none" }}
          onChange={e=>{ const f=e.target.files?.[0]; if(f) onImport(f); e.target.value=""; }}/>
      </div>

      {[...logs].reverse().map(entry => {
        const affected = Object.entries(entry.joints||{}).filter(([,v])=>v>0);
        const d = new Date(entry.date+"T12:00:00");
        return (
          <Card key={entry.date} style={{ marginBottom:12 }}>
            <div style={{ display:"flex", justifyContent:"space-between", alignItems:"flex-start", marginBottom:10 }}>
              <div>
                <div style={{ fontFamily:"'Lora',serif", fontWeight:700, fontSize:16, color:"#1e3d2f" }}>
                  {d.toLocaleDateString("en-GB",{weekday:"long",day:"numeric",month:"short"})}
                </div>
                {entry.notes && <div style={{ fontSize:12, color:"#9a9189", marginTop:3, fontStyle:"italic" }}>
                  "{entry.notes.slice(0,70)}{entry.notes.length>70?"…":""}"
                </div>}
              </div>
              <div style={{ textAlign:"center", background:pb(entry.overallPain), borderRadius:10,
                padding:"5px 12px", minWidth:52, border:`1.5px solid ${pf(entry.overallPain)}30` }}>
                <div style={{ fontSize:22, fontWeight:800, fontFamily:"'Lora',serif",
                  color:pf(entry.overallPain), lineHeight:1 }}>{entry.overallPain}</div>
                <div style={{ fontSize:10, color:"#9a9189", marginTop:2 }}>/ 5</div>
              </div>
            </div>
            <div style={{ display:"flex", flexWrap:"wrap", gap:6 }}>
              {affected.slice(0,5).map(([jid,pain])=>{
                const j=JOINTS.find(x=>x.id===jid);
                return <span key={jid} style={{ fontSize:11, background:pb(pain), color:pf(pain),
                  borderRadius:20, padding:"2px 9px", border:`1px solid ${pf(pain)}30` }}>{j?.label||jid}</span>;
              })}
              {affected.length>5 && <span style={{ fontSize:11, color:"#9a9189" }}>+{affected.length-5} more</span>}
              {(entry.foods||[]).map(fid=>{ const f=ALL_FOODS.find(x=>x.id===fid);
                return f?<span key={fid} style={{ fontSize:11, background:"#F5F0EB", color:"#7a7069", borderRadius:20, padding:"2px 9px" }}>{f.emoji} {f.label}</span>:null; })}
              {entry.exercise&&entry.exercise!=="None"&&<span style={{ fontSize:11, background:"#E8F5E9", color:"#2C5F4A", borderRadius:20, padding:"2px 9px" }}>🏃 {entry.exercise}{entry.exerciseMins?` ${entry.exerciseMins}m`:""}</span>}
              {(entry.alcohol||0)>0&&<span style={{ fontSize:11, background:"#FFF3E0", color:"#E65100", borderRadius:20, padding:"2px 9px" }}>🍷 {entry.alcohol} unit{entry.alcohol!==1?"s":""}</span>}
            </div>
          </Card>
        );
      })}
    </div>
  );
}

// ─── Insights Tab ─────────────────────────────────────────────────────────────

function InsightsTab({ logs }) {
  const n = logs.length;
  if (n < 5) return (
    <div style={{ padding:"40px 0" }}>
      <div style={{ textAlign:"center", marginBottom:24 }}>
        <div style={{ fontSize:52, marginBottom:14 }}>📊</div>
        <div style={{ fontFamily:"'Lora',serif", fontSize:20, color:"#1e3d2f", marginBottom:8 }}>Building your picture…</div>
        <div style={{ fontSize:14, color:"#9a9189", lineHeight:1.7, maxWidth:280, margin:"0 auto" }}>
          Insights unlock after 5 entries. You have {n} so far — keep logging!
        </div>
      </div>
      <Card style={{ background:"#F5F0EB", border:"none" }}>
        <div style={{ fontFamily:"'Lora',serif", fontWeight:700, marginBottom:8, color:"#1e3d2f" }}>What you'll see</div>
        <ul style={{ margin:0, padding:"0 0 0 18px", fontSize:13, color:"#7a7069", lineHeight:2 }}>
          <li>Pain trend over time</li>
          <li>Which foods correlate with worse days</li>
          <li>Impact of exercise, alcohol &amp; sleep</li>
          <li>Most consistently affected joints</li>
        </ul>
      </Card>
    </div>
  );

  const avge = arr => arr.length ? arr.reduce((s,x)=>s+x,0)/arr.length : null;
  const fmt  = v => v===null ? "–" : v.toFixed(1);

  const trend = logs.slice(-21).map(l=>({ date:l.date.slice(5), pain:l.overallPain }));

  const foodInsights = ALL_FOODS.map(food=>{
    const w  = logs.filter(l=>(l.foods||[]).includes(food.id)).map(l=>l.overallPain);
    const wo = logs.filter(l=>!(l.foods||[]).includes(food.id)).map(l=>l.overallPain);
    if(!w.length||!wo.length) return null;
    const aw=avge(w), ao=avge(wo);
    return {...food, aw:+aw.toFixed(1), ao:+ao.toFixed(1), diff:+(aw-ao).toFixed(1), n:w.length};
  }).filter(Boolean).sort((a,b)=>Math.abs(b.diff)-Math.abs(a.diff));

  const exDays   = logs.filter(l=>l.exercise&&l.exercise!=="None");
  const noExDays = logs.filter(l=>!l.exercise||l.exercise==="None");
  const exAvg    = avge(exDays.map(l=>l.overallPain));
  const noExAvg  = avge(noExDays.map(l=>l.overallPain));

  const alcDays   = logs.filter(l=>(l.alcohol||0)>0);
  const noAlcDays = logs.filter(l=>(l.alcohol||0)===0);
  const alcAvg    = avge(alcDays.map(l=>l.overallPain));
  const noAlcAvg  = avge(noAlcDays.map(l=>l.overallPain));

  const sleepBins = [
    {label:"< 6h", data:logs.filter(l=>l.sleep<6)},
    {label:"6–8h", data:logs.filter(l=>l.sleep>=6&&l.sleep<=8)},
    {label:"8h+",  data:logs.filter(l=>l.sleep>8)},
  ].map(b=>({label:b.label, avg:avge(b.data.map(l=>l.overallPain)), n:b.data.length})).filter(b=>b.n>0);

  const jointTotals = {};
  logs.forEach(l=>Object.entries(l.joints||{}).forEach(([id,pain])=>{
    if(!jointTotals[id]) jointTotals[id]=[];
    jointTotals[id].push(pain);
  }));
  const topJoints = JOINTS
    .map(j=>({...j, avg:avge(jointTotals[j.id]||[])}))
    .filter(j=>j.avg!==null&&j.avg>0)
    .sort((a,b)=>b.avg-a.avg).slice(0,6);

  const SB = ({label, value, sub, color="#2C5F4A", bg="#E8F5E9"}) => (
    <div style={{ flex:1, background:bg, borderRadius:12, padding:"14px 12px", textAlign:"center" }}>
      <div style={{ fontSize:26, fontWeight:800, fontFamily:"'Lora',serif", color, lineHeight:1 }}>{value}</div>
      <div style={{ fontSize:12, color, marginTop:4, fontWeight:600 }}>{label}</div>
      {sub && <div style={{ fontSize:11, color:"#9a9189", marginTop:2 }}>{sub}</div>}
    </div>
  );

  return (
    <div>
      <Card title="Pain Trend" emoji="📈">
        <div style={{ fontSize:12, color:"#9a9189", marginBottom:10 }}>Last {trend.length} days logged</div>
        <SimpleLineChart data={trend}/>
      </Card>

      {topJoints.length>0 && (
        <Card title="Most Affected Joints" emoji="🦴">
          {topJoints.map(j=>(
            <div key={j.id} style={{ display:"flex", alignItems:"center", gap:10, marginBottom:10 }}>
              <span style={{ fontSize:12, flex:1, color:"#1e3d2f", fontWeight:500 }}>{j.label}</span>
              <div style={{ flex:2, height:8, background:"#F0E8E0", borderRadius:4, overflow:"hidden" }}>
                <div style={{ width:`${(j.avg/5)*100}%`, height:"100%",
                  background:"linear-gradient(to right,#81C784,#EF5350)", borderRadius:4 }}/>
              </div>
              <span style={{ fontSize:13, fontWeight:800, color:pf(Math.round(j.avg)), width:26, textAlign:"right" }}>{j.avg.toFixed(1)}</span>
            </div>
          ))}
        </Card>
      )}

      {exAvg!==null&&noExAvg!==null&&(
        <Card title="Exercise Impact" emoji="🏃">
          <div style={{ display:"flex", gap:10, marginBottom:12 }}>
            <SB label="With exercise"    value={fmt(exAvg)}   sub={`${exDays.length} days`}   color="#2C5F4A" bg="#E8F5E9"/>
            <SB label="Without exercise" value={fmt(noExAvg)} sub={`${noExDays.length} days`} color="#E65100" bg="#FFF3E0"/>
          </div>
          <div style={{ padding:12, background:"#F5F0EB", borderRadius:9, fontSize:13, color:"#1e3d2f", lineHeight:1.6 }}>
            {exAvg<noExAvg ? `💡 Pain averages ${(noExAvg-exAvg).toFixed(1)} points lower on exercise days. Movement seems to help!`
              : exAvg>noExAvg ? `⚠️ Pain is slightly higher on exercise days — could be post-activity soreness. Worth mentioning to your rheumatologist.`
              : `➡️ No significant difference yet between exercise and rest days.`}
          </div>
        </Card>
      )}

      {alcAvg!==null&&noAlcAvg!==null&&alcDays.length>=2&&(
        <Card title="Alcohol Impact" emoji="🍷">
          <div style={{ display:"flex", gap:10 }}>
            <SB label="Alcohol days" value={fmt(alcAvg)}   sub={`${alcDays.length} days`}
              color={alcAvg>noAlcAvg?"#C62828":"#2C5F4A"} bg={alcAvg>noAlcAvg?"#FFEBEE":"#E8F5E9"}/>
            <SB label="No alcohol"   value={fmt(noAlcAvg)} sub={`${noAlcDays.length} days`} color="#2C5F4A" bg="#E8F5E9"/>
          </div>
        </Card>
      )}

      {sleepBins.length>=2&&(
        <Card title="Sleep & Pain" emoji="😴">
          <div style={{ display:"flex", gap:10 }}>
            {sleepBins.map(b=><SB key={b.label} label={b.label} value={fmt(b.avg)} sub={`${b.n} nights`} color="#2C5F4A" bg="#E8F5E9"/>)}
          </div>
        </Card>
      )}

      {foodInsights.length>0&&(
        <Card title="Food Correlations" emoji="🍽️">
          <p style={{ fontSize:12, color:"#9a9189", margin:"0 0 12px", lineHeight:1.6 }}>
            Change in average pain on days she ate each food. Positive = higher pain, negative = lower.
          </p>
          {foodInsights.slice(0,8).map(f=>(
            <div key={f.id} style={{ display:"flex", alignItems:"center", gap:10, marginBottom:10 }}>
              <span style={{ fontSize:13, width:22, textAlign:"center" }}>{f.emoji}</span>
              <span style={{ fontSize:12, flex:1, color:"#1e3d2f" }}>{f.label}</span>
              <div style={{ display:"flex", alignItems:"center", gap:8 }}>
                <div style={{ width:70, height:6, background:"#F0E8E0", borderRadius:3, overflow:"hidden" }}>
                  <div style={{ width:`${Math.min(100,Math.abs(f.diff)/2*100)}%`, height:"100%",
                    background:f.diff>0?"#EF5350":"#4CAF50", borderRadius:3 }}/>
                </div>
                <span style={{ fontSize:13, fontWeight:700, minWidth:36, textAlign:"right",
                  color:f.diff>0.2?"#C62828":f.diff<-0.2?"#2C5F4A":"#9a9189" }}>
                  {f.diff>0?"+":""}{f.diff}
                </span>
              </div>
            </div>
          ))}
          <div style={{ fontSize:12, color:"#9a9189", fontStyle:"italic", marginTop:8 }}>
            Based on {n} entries. Aim for 4+ weeks for reliable patterns.
          </div>
        </Card>
      )}

      <Card style={{ background:"#F5F0EB", border:"none" }}>
        <div style={{ fontFamily:"'Lora',serif", fontWeight:700, color:"#1e3d2f", marginBottom:6 }}>📋 Share with your doctor</div>
        <div style={{ fontSize:13, color:"#7a7069", lineHeight:1.7 }}>
          These are statistical patterns, not medical advice. Share them with your rheumatologist — they can help interpret what the data means for your specific situation.
        </div>
      </Card>
    </div>
  );
}

// ─── App Root ─────────────────────────────────────────────────────────────────

function PsATracker() {
  const [tab,     setTab]     = useState("log");
  const [log,     setLog]     = useState(emptyLog());
  const [logs,    setLogs]    = useState([]);
  const [saveMsg, setSaveMsg] = useState("");

  useEffect(() => {
    const stored = readLogs();
    if (stored.length) {
      setLogs(stored);
      const todayEntry = stored.find(l => l.date === todayStr());
      if (todayEntry) setLog(todayEntry);
    }
  }, []);

  const onSave = () => {
    const entry   = { ...log, date: todayStr() };
    const updated = [...logs.filter(l => l.date !== todayStr()), entry]
                      .sort((a,b) => a.date.localeCompare(b.date));
    writeLogs(updated);
    setLogs(updated);
    setSaveMsg("Saved ✓");
    setTimeout(() => setSaveMsg(""), 2500);
  };

  const onExport = () => {
    const blob = new Blob([JSON.stringify(logs, null, 2)], { type:"application/json" });
    const url  = URL.createObjectURL(blob);
    const a    = document.createElement("a");
    a.href = url; a.download = `psa-journal-${todayStr()}.json`; a.click();
    URL.revokeObjectURL(url);
  };

  const onImport = file => {
    const reader = new FileReader();
    reader.onload = e => {
      try {
        const imported = JSON.parse(e.target.result);
        if (!Array.isArray(imported)) throw new Error();
        const merged = Object.values(
          [...logs, ...imported].reduce((acc,l) => ({ ...acc, [l.date]:l }), {})
        ).sort((a,b) => a.date.localeCompare(b.date));
        writeLogs(merged);
        setLogs(merged);
        const todayEntry = merged.find(l => l.date === todayStr());
        if (todayEntry) setLog(todayEntry);
        alert(`Imported ${imported.length} entries.`);
      } catch { alert("Could not read file. Please use a valid PsA Journal backup."); }
    };
    reader.readAsText(file);
  };

  return (
    <div style={{ maxWidth:480, margin:"0 auto", minHeight:"100vh", background:"#FAF7F2" }}>
      <header style={{ background:"linear-gradient(135deg,#1e3d2f 0%,#2C5F4A 100%)",
        padding:"22px 20px 18px", display:"flex", justifyContent:"space-between", alignItems:"flex-end" }}>
        <div>
          <div style={{ fontFamily:"'Lora',serif", fontSize:26, fontWeight:800, color:"#fff", letterSpacing:.3, lineHeight:1 }}>PsA Journal</div>
          <div style={{ fontSize:12, color:"rgba(255,255,255,.65)", marginTop:4, letterSpacing:.4 }}>Psoriatic Arthritis Tracker</div>
        </div>
        <div style={{ textAlign:"right" }}>
          <div style={{ fontSize:13, color:"rgba(255,255,255,.85)" }}>
            {new Date().toLocaleDateString("en-GB",{weekday:"short",day:"numeric",month:"short",year:"numeric"})}
          </div>
          <div style={{ fontSize:11, color:"rgba(255,255,255,.5)", marginTop:2 }}>
            {logs.length} entr{logs.length===1?"y":"ies"} logged
          </div>
        </div>
      </header>

      <nav style={{ display:"flex", background:"#fff", borderBottom:"1px solid #ECE5DC",
        position:"sticky", top:0, zIndex:100, boxShadow:"0 2px 8px rgba(0,0,0,.06)" }}>
        {[["log","📋 Today"],["history","📅 History"],["insights","📊 Insights"]].map(([id,label])=>(
          <button key={id} onClick={()=>setTab(id)} style={{
            flex:1, padding:"12px 4px", border:"none",
            borderBottom:`2.5px solid ${tab===id?"#2C5F4A":"transparent"}`,
            background:"#fff", cursor:"pointer", fontSize:13,
            fontWeight:tab===id?600:400,
            color:tab===id?"#2C5F4A":"#9a9189", transition:"all .15s",
          }}>{label}</button>
        ))}
      </nav>

      <main style={{ padding:"16px 16px 0" }}>
        {tab==="log"      && <LogTab      log={log} setLog={setLog} onSave={onSave} saveMsg={saveMsg}/>}
        {tab==="history"  && <HistoryTab  logs={logs} onExport={onExport} onImport={onImport}/>}
        {tab==="insights" && <InsightsTab logs={logs}/>}
      </main>
    </div>
  );
}

ReactDOM.createRoot(document.getElementById('root')).render(<PsATracker />);
</script>

</body>
</html>
