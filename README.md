# agenda-ai
import { useState, useEffect, useCallback, useRef } from "react";

const QUADRANTS = [
  { id:"q1", label:"Urgente + Importante",     short:"Fai ora",  color:"#C53030", bg:"#FFF5F5", border:"#FC8181", icon:"🔥" },
  { id:"q2", label:"Non urgente + Importante", short:"Pianifica",color:"#2B6CB0", bg:"#EBF8FF", border:"#63B3ED", icon:"📅" },
  { id:"q3", label:"Urgente + Non importante", short:"Delega",   color:"#B7791F", bg:"#FFFFF0", border:"#F6E05E", icon:"👥" },
  { id:"q4", label:"Non urgente + Non imp.",   short:"Elimina",  color:"#4A5568", bg:"#F7FAFC", border:"#CBD5E0", icon:"🗑️" },
];

const TEAM = ["Io","Aurora","Eleonora","Sabina","Luigi","Lorenzo","Annalisa","Nicola C.","Eneida"];
const VIEWS = [["secretary","🤖 Segretaria"],["matrix","🧩 Matrice"],["list","📋 Lista"],["calendar","📅 Calendario"]];
const MONTHS_IT = ["Gennaio","Febbraio","Marzo","Aprile","Maggio","Giugno","Luglio","Agosto","Settembre","Ottobre","Novembre","Dicembre"];
const DOW_IT = ["Dom","Lun","Mar","Mer","Gio","Ven","Sab"];

function genId() { return Date.now().toString(36) + Math.random().toString(36).slice(2); }
function daysLeft(d) {
  if (!d) return null;
  const t = new Date(); t.setHours(0,0,0,0);
  return Math.ceil((new Date(d+"T00:00:00") - t) / 86400000);
}
function fmtDate(d) {
  if (!d) return "";
  return new Date(d+"T00:00:00").toLocaleDateString("it-IT",{day:"2-digit",month:"short",year:"numeric"});
}
function urgencyScore(t) {
  const d = daysLeft(t.dueDate);
  return ({q1:0,q2:1,q3:2,q4:3}[t.quadrant]??4)*100 + (d===null?10:d<0?-1:d);
}

function DaysChip({dateStr,big}) {
  const d = daysLeft(dateStr); if (d===null) return null;
  let bg,color,label;
  if (d<0)      {bg="#FED7D7";color="#9B2C2C";label=`${Math.abs(d)}g fa`;}
  else if(d===0){bg="#FEEBC8";color="#7B341E";label="oggi!";}
  else if(d<=2) {bg="#FED7D7";color="#9B2C2C";label=`${d}g`;}
  else if(d<=7) {bg="#FEEBC8";color="#7B341E";label=`${d}g`;}
  else          {bg="#C6F6D5";color="#22543D";label=`${d}g`;}
  return <span style={{fontSize:big?13:11,fontWeight:700,background:bg,color,borderRadius:6,padding:big?"3px 10px":"2px 7px"}}>{label}</span>;
}

// ── subtask progress bar ──
function SubProgress({subtasks=[]}) {
  if (!subtasks.length) return null;
  const done = subtasks.filter(s=>s.done).length;
  const pct = Math.round((done/subtasks.length)*100);
  return (
    <div style={{marginTop:5,display:"flex",alignItems:"center",gap:6}}>
      <div style={{flex:1,height:4,background:"#E2E8F0",borderRadius:4,overflow:"hidden"}}>
        <div style={{height:"100%",width:`${pct}%`,background:"#16A34A",borderRadius:4,transition:"width 0.3s"}}/>
      </div>
      <span style={{fontSize:10,fontWeight:700,color:"#64748B",whiteSpace:"nowrap"}}>{done}/{subtasks.length}</span>
    </div>
  );
}

const EMPTY = {title:"",quadrant:"q1",assignee:"Io",dueDate:"",notes:"",done:false,subtasks:[]};

// ── AI helpers ──────────────────────────────────────────────────────────────
async function callClaude(prompt, sys) {
  const r = await fetch("https://api.anthropic.com/v1/messages",{
    method:"POST", headers:{"Content-Type":"application/json"},
    body:JSON.stringify({model:"claude-sonnet-4-20250514",max_tokens:1000,system:sys,messages:[{role:"user",content:prompt}]})
  });
  return (await r.json()).content?.[0]?.text||"";
}

async function parseVoiceToTask(transcript) {
  const today = new Date().toISOString().split("T")[0];
  const sys = `Converti testo parlato in attività. Rispondi SOLO JSON valido senza backtick.
Schema: {"title":string,"quadrant":"q1"|"q2"|"q3"|"q4","assignee":string,"dueDate":"YYYY-MM-DD"|"","notes":string,"subtasks":[{"title":string,"done":false}]}
Quadranti: q1=urgente+importante,q2=pianifica,q3=delega,q4=elimina. Team: ${TEAM.join(",")}. Oggi: ${today}.`;
  try { return JSON.parse((await callClaude(transcript,sys)).replace(/```json|```/g,"")); } catch { return null; }
}

async function parseDocumentToTasks(text) {
  const today = new Date().toISOString().split("T")[0];
  const sys = `Estrai attività da documenti. Rispondi SOLO array JSON valido senza backtick.
Schema: [{"title":string,"quadrant":"q1"|"q2"|"q3"|"q4","assignee":string,"dueDate":"YYYY-MM-DD"|"","notes":string,"subtasks":[{"title":string,"done":false}]}]
Team: ${TEAM.join(",")}. Oggi: ${today}. Estrai TUTTE le attività, sotto-attività e scadenze.`;
  try {
    const arr = JSON.parse((await callClaude(text.slice(0,8000),sys)).replace(/```json|```/g,"").trim());
    return Array.isArray(arr)?arr:[];
  } catch { return []; }
}

async function generateBriefing(tasks) {
  const today = new Date().toISOString().split("T")[0];
  const urgent = tasks.filter(t=>!t.done&&(t.quadrant==="q1"||(t.dueDate&&daysLeft(t.dueDate)!==null&&daysLeft(t.dueDate)<=3)));
  return callClaude(
    `Data: ${today}\nUrgenti:\n${urgent.map(t=>`- [${t.assignee}] ${t.title} — scadenza: ${t.dueDate||"n.d."}\n  Sotto-attività: ${(t.subtasks||[]).map(s=>`${s.done?"✓":"○"} ${s.title}`).join(", ")||"nessuna"}`).join("\n")}\n\nTutte aperte:\n${tasks.filter(t=>!t.done).map(t=>`- [${t.assignee}] ${t.title}`).join("\n")}\n\nGenera briefing email professionale con Oggetto: e corpo.`,
    "Sei una segretaria professionale italiana. Genera briefing conciso ed elegante."
  );
}

// ── MAIN ─────────────────────────────────────────────────────────────────────
export default function App() {
  const [tasks, setTasks]         = useState([]);
  const [view, setView]           = useState("secretary");
  const [showForm, setShowForm]   = useState(false);
  const [editId, setEditId]       = useState(null);
  const [form, setForm]           = useState(EMPTY);
  const [filter, setFilter]       = useState("all");
  const [calMonth, setCalMonth]   = useState(()=>{const n=new Date();return{y:n.getFullYear(),m:n.getMonth()};});
  const [loaded, setLoaded]       = useState(false);
  const [expandedTask, setExpanded] = useState(null); // id of task with open subtask panel

  // voice
  const [listening, setListening] = useState(false);
  const [voiceText, setVoiceText] = useState("");
  const [voiceStatus, setVoiceStatus] = useState("");
  const [aiLoading, setAiLoading] = useState(false);
  const recognRef = useRef(null);

  // doc
  const [docStatus, setDocStatus] = useState("");
  const [docLoading, setDocLoading] = useState(false);
  const fileRef = useRef(null);

  // briefing
  const [briefing, setBriefing]         = useState("");
  const [briefingLoading, setBriefingLoading] = useState(false);
  const [briefingSent, setBriefingSent] = useState(false);
  const [emailAddr, setEmailAddr]       = useState("s.randazzo@energiacapitale.com");

  // new subtask input per form
  const [newSubInput, setNewSubInput] = useState("");

  useEffect(()=>{
    (async()=>{
      try{const r=await window.storage.get("agenda-v3");if(r?.value)setTasks(JSON.parse(r.value));}catch{}
      try{const r=await window.storage.get("agenda-email");if(r?.value)setEmailAddr(r.value);}catch{}
      setLoaded(true);
    })();
  },[]);

  const saveTasks = useCallback(async(t)=>{
    setTasks(t);
    try{await window.storage.set("agenda-v3",JSON.stringify(t));}catch{}
  },[]);

  const saveEmail = async(e)=>{setEmailAddr(e);try{await window.storage.set("agenda-email",e);}catch{}};

  const addTask   = (t) => saveTasks([...tasks,{...t,id:genId(),done:false,subtasks:t.subtasks||[]}]);
  const toggleDone = (id) => saveTasks(tasks.map(t=>t.id===id?{...t,done:!t.done}:t));
  const deleteTask = (id) => saveTasks(tasks.filter(t=>t.id!==id));

  // subtask helpers
  const toggleSubtask = (taskId, subId) => saveTasks(tasks.map(t=>{
    if(t.id!==taskId) return t;
    return {...t, subtasks:(t.subtasks||[]).map(s=>s.id===subId?{...s,done:!s.done}:s)};
  }));
  const addSubtask = (taskId, title) => {
    if(!title.trim()) return;
    saveTasks(tasks.map(t=>{
      if(t.id!==taskId) return t;
      return {...t, subtasks:[...(t.subtasks||[]),{id:genId(),title:title.trim(),done:false}]};
    }));
  };
  const deleteSubtask = (taskId, subId) => saveTasks(tasks.map(t=>{
    if(t.id!==taskId) return t;
    return {...t, subtasks:(t.subtasks||[]).filter(s=>s.id!==subId)};
  }));

  const submitForm = () => {
    if(!form.title.trim()) return;
    const cleaned = {...form, subtasks:(form.subtasks||[]).filter(s=>s.title?.trim())};
    if(editId) saveTasks(tasks.map(t=>t.id===editId?{...cleaned,id:editId}:t));
    else saveTasks([...tasks,{...cleaned,id:genId(),done:false}]);
    setShowForm(false); setNewSubInput("");
  };

  const openEdit = (task) => { setForm({...task,subtasks:[...(task.subtasks||[])]}); setEditId(task.id); setShowForm(true); setNewSubInput(""); };
  const openNew  = (pre={}) => { setForm({...EMPTY,...pre,subtasks:[]}); setEditId(null); setShowForm(true); setNewSubInput(""); };

  const addFormSub = () => {
    if(!newSubInput.trim()) return;
    setForm(f=>({...f,subtasks:[...(f.subtasks||[]),{id:genId(),title:newSubInput.trim(),done:false}]}));
    setNewSubInput("");
  };

  const filtered = tasks.filter(t=>filter==="mine"?t.assignee==="Io":filter==="team"?t.assignee!=="Io":true);
  const sorted   = [...filtered].sort((a,b)=>{if(a.done!==b.done)return a.done?1:-1;return urgencyScore(a)-urgencyScore(b);});

  // voice
  const startVoice = () => {
    const SR = window.SpeechRecognition||window.webkitSpeechRecognition;
    if(!SR){setVoiceStatus("⚠️ Usa Chrome.");return;}
    const r=new SR(); r.lang="it-IT"; r.continuous=false; r.interimResults=false;
    r.onstart=()=>{setListening(true);setVoiceStatus("🎤 Sto ascoltando...");};
    r.onresult=async(e)=>{
      const t=e.results[0][0].transcript; setVoiceText(t); setListening(false);
      setVoiceStatus("🧠 Elaboro..."); setAiLoading(true);
      try{
        const task=await parseVoiceToTask(t);
        if(task?.title){addTask(task);setVoiceStatus(`✅ Aggiunto: "${task.title}"`);}
        else setVoiceStatus("⚠️ Non ho capito. Riprova.");
      }catch{setVoiceStatus("⚠️ Errore AI.");}
      setAiLoading(false); setTimeout(()=>setVoiceStatus(""),4000);
    };
    r.onerror=(e)=>{setListening(false);setVoiceStatus(`⚠️ ${e.error}`);};
    r.onend=()=>setListening(false);
    recognRef.current=r; r.start();
  };
  const stopVoice=()=>{recognRef.current?.stop();setListening(false);};

  // doc upload
  const handleDoc = async(e)=>{
    const file=e.target.files?.[0]; if(!file)return;
    setDocLoading(true); setDocStatus("📄 Lettura...");
    try{
      let text="";
      if(file.name.endsWith(".pdf")){
        const b64=await new Promise((res,rej)=>{const r=new FileReader();r.onload=()=>res(r.result.split(",")[1]);r.onerror=rej;r.readAsDataURL(file);});
        const resp=await fetch("https://api.anthropic.com/v1/messages",{method:"POST",headers:{"Content-Type":"application/json"},
          body:JSON.stringify({model:"claude-sonnet-4-20250514",max_tokens:1000,
            messages:[{role:"user",content:[{type:"document",source:{type:"base64",media_type:"application/pdf",data:b64}},{type:"text",text:"Estrai tutto il testo."}]}]})});
        text=(await resp.json()).content?.[0]?.text||"";
      } else { text=await file.text(); }
      setDocStatus("🧠 Estraggo attività...");
      const newTasks=await parseDocumentToTasks(text);
      if(newTasks.length>0){ saveTasks([...tasks,...newTasks.map(t=>({...t,id:genId(),done:false,subtasks:t.subtasks||[]}))]); setDocStatus(`✅ Estratte ${newTasks.length} attività!`); }
      else setDocStatus("⚠️ Nessuna attività trovata.");
    }catch{setDocStatus("⚠️ Errore lettura.");}
    setDocLoading(false); setTimeout(()=>setDocStatus(""),5000); e.target.value="";
  };

  // briefing
  const handleBriefing=async()=>{setBriefingLoading(true);setBriefing("");setBriefingSent(false);try{setBriefing(await generateBriefing(tasks));}catch{setBriefing("Errore.");}setBriefingLoading(false);};
  const sendBriefingEmail=async()=>{
    if(!briefing||!emailAddr)return;
    const lines=briefing.split("\n");
    let subject="Briefing Agenda - "+new Date().toLocaleDateString("it-IT"), body=briefing;
    const ol=lines.find(l=>l.toLowerCase().startsWith("oggetto:"));
    if(ol){subject=ol.replace(/^oggetto:\s*/i,"");body=lines.filter(l=>!l.toLowerCase().startsWith("oggetto:")).join("\n");}
    try{
      await fetch("https://api.anthropic.com/v1/messages",{method:"POST",headers:{"Content-Type":"application/json"},
        body:JSON.stringify({model:"claude-sonnet-4-20250514",max_tokens:1000,
          messages:[{role:"user",content:`Invia email a ${emailAddr}.\nOggetto: ${subject}\nCorpo:\n${body}`}],
          mcp_servers:[{type:"url",url:"https://microsoft365.mcp.claude.com/mcp",name:"microsoft365"}]})});
      setBriefingSent(true);
    }catch{setBriefingSent(false);}
  };

  // calendar
  const daysInMonth=(y,m)=>new Date(y,m+1,0).getDate();
  const firstDayOf=(y,m)=>new Date(y,m,1).getDay();
  const todayStr=new Date().toISOString().split("T")[0];

  // secretary buckets
  const overdue  = sorted.filter(t=>!t.done&&t.dueDate&&daysLeft(t.dueDate)<0);
  const dueToday = sorted.filter(t=>!t.done&&t.dueDate&&daysLeft(t.dueDate)===0);
  const dueSoon  = sorted.filter(t=>!t.done&&t.dueDate&&daysLeft(t.dueDate)>0&&daysLeft(t.dueDate)<=3);
  const fireQ1   = sorted.filter(t=>!t.done&&t.quadrant==="q1"&&!overdue.find(x=>x.id===t.id)&&!dueToday.find(x=>x.id===t.id));

  const C={dark:"#0F172A",mid:"#1E293B",accent:"#E53E3E",blue:"#2563EB",green:"#16A34A",gold:"#D97706",light:"#F8FAFC",muted:"#94A3B8",border:"#E2E8F0"};

  if(!loaded) return <div style={{fontFamily:"'Outfit',sans-serif",minHeight:"100vh",background:C.light,display:"flex",alignItems:"center",justifyContent:"center",color:C.muted,fontSize:14}}>Caricamento agenda...</div>;

  // ── TaskCard (inline subtask toggle) ────────────────────────────────────────
  const TaskCard = ({task,compact}) => {
    const q = QUADRANTS.find(x=>x.id===task.quadrant);
    const subs = task.subtasks||[];
    const isExpanded = expandedTask===task.id;
    const [inlineInput,setInlineInput] = useState("");
    return (
      <div style={{background:"#fff",borderRadius:10,marginBottom:8,boxShadow:"0 1px 3px rgba(0,0,0,0.07)",borderLeft:`3px solid ${q.border}`,overflow:"hidden"}}>
        <div style={{padding:compact?"9px 12px":"12px 14px",display:"flex",alignItems:"flex-start",gap:10,cursor:"pointer"}} onClick={()=>openEdit(task)}>
          <div onClick={e=>{e.stopPropagation();toggleDone(task.id);}} style={{width:17,height:17,borderRadius:4,border:`2px solid ${task.done?"#16A34A":C.border}`,background:task.done?"#16A34A":"transparent",display:"flex",alignItems:"center",justifyContent:"center",flexShrink:0,marginTop:1,cursor:"pointer"}}>
            {task.done&&<span style={{color:"#fff",fontSize:10,fontWeight:900}}>✓</span>}
          </div>
          <div style={{flex:1,minWidth:0}}>
            <div style={{fontSize:13,fontWeight:600,color:task.done?C.muted:C.dark,textDecoration:task.done?"line-through":"none",lineHeight:1.4}}>{task.title}</div>
            <div style={{display:"flex",gap:5,alignItems:"center",marginTop:4,flexWrap:"wrap"}}>
              <span style={{fontSize:11,fontWeight:600,background:q.bg,color:q.color,borderRadius:4,padding:"1px 6px"}}>{q.icon} {q.short}</span>
              <span style={{fontSize:11,fontWeight:600,background:"#EEF2FF",color:"#4338CA",borderRadius:4,padding:"1px 6px"}}>👤 {task.assignee}</span>
              {task.dueDate&&<DaysChip dateStr={task.dueDate}/>}
              {subs.length>0&&<span onClick={e=>{e.stopPropagation();setExpanded(isExpanded?null:task.id);}} style={{fontSize:11,fontWeight:600,background:"#F0FDF4",color:"#16A34A",borderRadius:4,padding:"1px 6px",cursor:"pointer"}}>☑ {subs.filter(s=>s.done).length}/{subs.length}</span>}
            </div>
            {subs.length>0&&<SubProgress subtasks={subs}/>}
          </div>
          {subs.length>0&&(
            <button onClick={e=>{e.stopPropagation();setExpanded(isExpanded?null:task.id);}} style={{background:"none",border:"none",cursor:"pointer",color:C.muted,fontSize:16,padding:"0 4px",flexShrink:0}}>
              {isExpanded?"▲":"▼"}
            </button>
          )}
        </div>

        {/* Inline subtasks panel */}
        {isExpanded&&(
          <div style={{background:"#F8FAFC",borderTop:`1px solid ${C.border}`,padding:"10px 14px 12px 40px"}}>
            {subs.map(s=>(
              <div key={s.id} style={{display:"flex",alignItems:"center",gap:8,marginBottom:6}}>
                <div onClick={()=>toggleSubtask(task.id,s.id)} style={{width:15,height:15,borderRadius:3,border:`2px solid ${s.done?"#16A34A":C.border}`,background:s.done?"#16A34A":"transparent",display:"flex",alignItems:"center",justifyContent:"center",flexShrink:0,cursor:"pointer"}}>
                  {s.done&&<span style={{color:"#fff",fontSize:9,fontWeight:900}}>✓</span>}
                </div>
                <span style={{fontSize:12,color:s.done?C.muted:C.dark,textDecoration:s.done?"line-through":"none",flex:1}}>{s.title}</span>
                <button onClick={()=>deleteSubtask(task.id,s.id)} style={{background:"none",border:"none",cursor:"pointer",color:"#FCA5A5",fontSize:13,padding:0}}>✕</button>
              </div>
            ))}
            <div style={{display:"flex",gap:6,marginTop:8}}>
              <input value={inlineInput} onChange={e=>setInlineInput(e.target.value)}
                onKeyDown={e=>{if(e.key==="Enter"&&inlineInput.trim()){addSubtask(task.id,inlineInput);setInlineInput("");}}}
                placeholder="+ Nuova sotto-attività..." style={{flex:1,border:`1px solid ${C.border}`,borderRadius:6,padding:"5px 8px",fontSize:12,outline:"none",fontFamily:"inherit"}}/>
              <button onClick={()=>{if(inlineInput.trim()){addSubtask(task.id,inlineInput);setInlineInput("");}}} style={{background:C.dark,color:"#fff",border:"none",borderRadius:6,padding:"5px 10px",fontSize:12,fontWeight:700,cursor:"pointer"}}>+</button>
            </div>
          </div>
        )}
      </div>
    );
  };

  return (
    <>
      <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@400;500;600;700;800&family=Lora:wght@600;700&display=swap" rel="stylesheet"/>
      <div style={{fontFamily:"'Outfit',sans-serif",minHeight:"100vh",background:C.light,color:C.dark}}>

        {/* HEADER */}
        <div style={{background:C.dark,padding:"14px 22px",display:"flex",alignItems:"center",justifyContent:"space-between",gap:12,flexWrap:"wrap"}}>
          <div>
            <div style={{fontFamily:"'Lora',serif",fontSize:19,color:"#F8FAFC",fontWeight:700,letterSpacing:-0.3}}>🤖 Agenda AI</div>
            <div style={{fontSize:11,color:C.muted,marginTop:1}}>La tua segretaria personale</div>
          </div>
          <div style={{display:"flex",gap:4,flexWrap:"wrap"}}>
            {VIEWS.map(([v,l])=>(
              <button key={v} onClick={()=>setView(v)} style={{padding:"6px 13px",borderRadius:18,border:"none",cursor:"pointer",fontSize:12,fontWeight:600,background:view===v?"#F8FAFC":"transparent",color:view===v?C.dark:C.muted}}>{l}</button>
            ))}
          </div>
          <div style={{display:"flex",gap:8,alignItems:"center"}}>
            <button onClick={listening?stopVoice:startVoice} disabled={aiLoading}
              style={{background:listening?"#E53E3E":aiLoading?"#4A5568":"#1E293B",color:"#fff",border:`2px solid ${listening?"#FC8181":"#334155"}`,borderRadius:10,padding:"7px 14px",fontSize:12,fontWeight:700,cursor:"pointer",display:"flex",alignItems:"center",gap:6,animation:listening?"pulse 1s infinite":"none"}}>
              {listening?"⏹ Stop":aiLoading?"⏳ AI...":"🎤 Voce"}
            </button>
            <button onClick={()=>fileRef.current?.click()} disabled={docLoading}
              style={{background:"#1E293B",color:"#fff",border:"2px solid #334155",borderRadius:10,padding:"7px 14px",fontSize:12,fontWeight:700,cursor:"pointer"}}>
              {docLoading?"⏳ Carico...":"📎 Documento"}
            </button>
            <input ref={fileRef} type="file" accept=".pdf,.txt,.docx,.xlsx,.csv" style={{display:"none"}} onChange={handleDoc}/>
            <button onClick={()=>openNew()} style={{background:C.accent,color:"#fff",border:"none",borderRadius:10,padding:"8px 16px",fontSize:12,fontWeight:700,cursor:"pointer"}}>+ Attività</button>
          </div>
        </div>

        {/* STATUS BAR */}
        {(voiceStatus||docStatus||voiceText)&&(
          <div style={{background:C.mid,padding:"8px 22px",display:"flex",gap:16,alignItems:"center",flexWrap:"wrap"}}>
            {voiceStatus&&<span style={{fontSize:12,color:"#F8FAFC",fontWeight:600}}>{voiceStatus}</span>}
            {voiceText&&!aiLoading&&<span style={{fontSize:11,color:C.muted}}>Detto: "{voiceText}"</span>}
            {docStatus&&<span style={{fontSize:12,color:"#F8FAFC",fontWeight:600}}>{docStatus}</span>}
          </div>
        )}

        <div style={{padding:"18px 22px",maxWidth:1100,margin:"0 auto"}}>

          {/* FILTER BAR */}
          <div style={{display:"flex",gap:6,marginBottom:16,alignItems:"center"}}>
            <span style={{fontSize:11,fontWeight:700,color:C.muted,marginRight:4}}>Mostra:</span>
            {[["all","Tutti"],["mine","Le mie"],["team","Del team"]].map(([f,l])=>(
              <button key={f} onClick={()=>setFilter(f)} style={{padding:"4px 12px",borderRadius:16,border:`2px solid ${filter===f?C.dark:C.border}`,background:filter===f?C.dark:"#fff",color:filter===f?"#fff":C.muted,fontSize:12,fontWeight:600,cursor:"pointer"}}>{l}</button>
            ))}
            <span style={{marginLeft:"auto",fontSize:12,color:C.muted}}>{tasks.filter(t=>!t.done).length} da fare · {tasks.filter(t=>t.done).length} completate</span>
          </div>

          {/* ── SECRETARY ── */}
          {view==="secretary"&&(
            <div style={{display:"grid",gridTemplateColumns:"1fr 340px",gap:18}}>
              <div>
                <div style={{display:"grid",gridTemplateColumns:"repeat(4,1fr)",gap:10,marginBottom:18}}>
                  {[[overdue.length,"Scadute","#E53E3E"],[dueToday.length,"Oggi","#D97706"],[dueSoon.length,"Prossimi 3g","#2563EB"],[fireQ1.length,"Q1 Urgenti","#C53030"]].map(([n,l,c])=>(
                    <div key={l} style={{background:"#fff",borderRadius:10,padding:"12px 14px",borderTop:`4px solid ${c}`,boxShadow:"0 1px 3px rgba(0,0,0,0.06)"}}>
                      <div style={{fontSize:22,fontWeight:800,color:C.dark}}>{n}</div>
                      <div style={{fontSize:11,color:C.muted,fontWeight:600}}>{l}</div>
                    </div>
                  ))}
                </div>
                {overdue.length>0&&<div style={{marginBottom:16}}><div style={{fontSize:12,fontWeight:800,color:"#E53E3E",textTransform:"uppercase",letterSpacing:0.8,marginBottom:8}}>🚨 Scadute</div>{overdue.map(t=><TaskCard key={t.id} task={t}/>)}</div>}
                {dueToday.length>0&&<div style={{marginBottom:16}}><div style={{fontSize:12,fontWeight:800,color:"#D97706",textTransform:"uppercase",letterSpacing:0.8,marginBottom:8}}>⚡ Scadono oggi</div>{dueToday.map(t=><TaskCard key={t.id} task={t}/>)}</div>}
                {dueSoon.length>0&&<div style={{marginBottom:16}}><div style={{fontSize:12,fontWeight:800,color:"#2563EB",textTransform:"uppercase",letterSpacing:0.8,marginBottom:8}}>📅 Prossimi 3 giorni</div>{dueSoon.map(t=><TaskCard key={t.id} task={t}/>)}</div>}
                {fireQ1.length>0&&<div style={{marginBottom:16}}><div style={{fontSize:12,fontWeight:800,color:"#C53030",textTransform:"uppercase",letterSpacing:0.8,marginBottom:8}}>🔥 Urgente + Importante</div>{fireQ1.map(t=><TaskCard key={t.id} task={t}/>)}</div>}
                {overdue.length===0&&dueToday.length===0&&dueSoon.length===0&&fireQ1.length===0&&(
                  <div style={{background:"#fff",borderRadius:12,padding:32,textAlign:"center",color:C.muted}}>
                    <div style={{fontSize:32,marginBottom:8}}>✅</div>
                    <div style={{fontWeight:600}}>Nessuna urgenza! Ottimo lavoro.</div>
                  </div>
                )}
              </div>

              {/* Right: briefing + team */}
              <div>
                <div style={{background:"#fff",borderRadius:12,padding:18,boxShadow:"0 1px 4px rgba(0,0,0,0.07)",marginBottom:14}}>
                  <div style={{fontFamily:"'Lora',serif",fontSize:15,fontWeight:700,marginBottom:12}}>📧 Briefing Email</div>
                  <div style={{marginBottom:10}}>
                    <label style={{fontSize:11,fontWeight:700,color:C.muted,textTransform:"uppercase",letterSpacing:0.6,display:"block",marginBottom:4}}>Invia a</label>
                    <input value={emailAddr} onChange={e=>saveEmail(e.target.value)} placeholder="email@azienda.com" style={{width:"100%",border:`2px solid ${C.border}`,borderRadius:8,padding:"7px 10px",fontSize:13,boxSizing:"border-box",fontFamily:"inherit",outline:"none"}}/>
                  </div>
                  <button onClick={handleBriefing} disabled={briefingLoading} style={{width:"100%",background:C.dark,color:"#fff",border:"none",borderRadius:8,padding:"9px",fontSize:13,fontWeight:700,cursor:"pointer",marginBottom:8}}>
                    {briefingLoading?"⏳ Generando...":"🧠 Genera briefing AI"}
                  </button>
                  {briefing&&(
                    <>
                      <div style={{background:"#F8FAFC",borderRadius:8,padding:12,fontSize:12,color:C.dark,lineHeight:1.6,maxHeight:220,overflowY:"auto",marginBottom:8,whiteSpace:"pre-wrap",border:`1px solid ${C.border}`}}>{briefing}</div>
                      <button onClick={sendBriefingEmail} disabled={!emailAddr} style={{width:"100%",background:briefingSent?"#16A34A":C.blue,color:"#fff",border:"none",borderRadius:8,padding:"9px",fontSize:13,fontWeight:700,cursor:"pointer"}}>
                        {briefingSent?"✅ Email inviata!":"📤 Invia via Outlook"}
                      </button>
                    </>
                  )}
                </div>
                <div style={{background:"#fff",borderRadius:12,padding:18,boxShadow:"0 1px 4px rgba(0,0,0,0.07)"}}>
                  <div style={{fontFamily:"'Lora',serif",fontSize:15,fontWeight:700,marginBottom:12}}>👥 Team — in scadenza</div>
                  {sorted.filter(t=>!t.done&&t.assignee!=="Io"&&t.dueDate).slice(0,6).map(t=>(
                    <div key={t.id} onClick={()=>openEdit(t)} style={{display:"flex",justifyContent:"space-between",alignItems:"center",padding:"8px 0",borderBottom:`1px solid ${C.border}`,cursor:"pointer"}}>
                      <div>
                        <div style={{fontSize:12,fontWeight:600,color:C.dark}}>{t.title}</div>
                        <div style={{fontSize:11,color:C.muted}}>👤 {t.assignee} {(t.subtasks||[]).length>0&&`· ☑ ${(t.subtasks||[]).filter(s=>s.done).length}/${(t.subtasks||[]).length}`}</div>
                      </div>
                      <DaysChip dateStr={t.dueDate}/>
                    </div>
                  ))}
                  {sorted.filter(t=>!t.done&&t.assignee!=="Io"&&t.dueDate).length===0&&<div style={{fontSize:12,color:C.muted}}>Nessuna scadenza del team.</div>}
                </div>
              </div>
            </div>
          )}

          {/* ── MATRIX ── */}
          {view==="matrix"&&(
            <>
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:4,marginBottom:6}}>
                <div style={{textAlign:"center",fontSize:10,fontWeight:800,color:"#E53E3E",textTransform:"uppercase",letterSpacing:1}}>🔴 URGENTE</div>
                <div style={{textAlign:"center",fontSize:10,fontWeight:800,color:C.muted,textTransform:"uppercase",letterSpacing:1}}>⚪ NON URGENTE</div>
              </div>
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:14}}>
                {QUADRANTS.map(q=>{
                  const qt=filtered.filter(t=>t.quadrant===q.id).sort((a,b)=>a.done?1:-1);
                  return(
                    <div key={q.id} style={{background:q.bg,border:`2px solid ${q.border}`,borderRadius:14,padding:16,minHeight:180}}>
                      <div style={{display:"flex",alignItems:"center",gap:8,marginBottom:12}}>
                        <span style={{fontSize:18}}>{q.icon}</span>
                        <div>
                          <div style={{fontSize:11,fontWeight:800,color:q.color,textTransform:"uppercase",letterSpacing:0.8}}>{q.short}</div>
                          <div style={{fontSize:11,color:C.muted}}>{q.label}</div>
                        </div>
                        <span style={{marginLeft:"auto",background:q.color,color:"#fff",borderRadius:20,fontSize:11,fontWeight:800,padding:"2px 8px"}}>{qt.length}</span>
                      </div>
                      {qt.length===0&&<div style={{textAlign:"center",color:C.muted,padding:"18px 0",fontSize:12}}>Nessuna attività</div>}
                      {qt.map(task=><TaskCard key={task.id} task={task} compact/>)}
                      <button onClick={()=>openNew({quadrant:q.id})} style={{width:"100%",background:"transparent",border:`1px dashed ${q.border}`,borderRadius:7,padding:"6px",fontSize:12,color:q.color,cursor:"pointer",fontWeight:600,marginTop:4}}>+ Aggiungi</button>
                    </div>
                  );
                })}
              </div>
            </>
          )}

          {/* ── LIST ── */}
          {view==="list"&&(
            <div style={{background:"#fff",borderRadius:14,overflow:"hidden",boxShadow:"0 2px 8px rgba(0,0,0,0.06)"}}>
              <div style={{display:"grid",gridTemplateColumns:"30px 1fr 110px 120px 110px 70px 80px",padding:"11px 16px",background:C.dark,alignItems:"center",gap:8}}>
                {["","Attività","Priorità","Assegnato a","Scadenza","Sotto-att.","Stato"].map((h,i)=><div key={i} style={{fontSize:10,color:C.muted,fontWeight:700,textTransform:"uppercase",letterSpacing:0.8}}>{h}</div>)}
              </div>
              {sorted.length===0&&<div style={{padding:32,textAlign:"center",color:C.muted}}>Nessuna attività.</div>}
              {sorted.map((task,i)=>{
                const q=QUADRANTS.find(x=>x.id===task.quadrant);
                const subs=task.subtasks||[];
                return(
                  <div key={task.id}>
                    <div onClick={()=>openEdit(task)} style={{display:"grid",gridTemplateColumns:"30px 1fr 110px 120px 110px 70px 80px",padding:"11px 16px",alignItems:"center",gap:8,background:i%2===0?"#fff":"#FAFAF9",borderBottom:`1px solid ${C.border}`,cursor:"pointer"}}>
                      <div onClick={e=>{e.stopPropagation();toggleDone(task.id);}} style={{width:16,height:16,borderRadius:4,border:`2px solid ${task.done?"#16A34A":C.border}`,background:task.done?"#16A34A":"transparent",display:"flex",alignItems:"center",justifyContent:"center",cursor:"pointer"}}>
                        {task.done&&<span style={{color:"#fff",fontSize:10,fontWeight:900}}>✓</span>}
                      </div>
                      <div>
                        <div style={{fontSize:13,fontWeight:600,color:task.done?C.muted:C.dark,textDecoration:task.done?"line-through":"none"}}>{task.title}</div>
                        {subs.length>0&&<SubProgress subtasks={subs}/>}
                      </div>
                      <span style={{fontSize:11,fontWeight:700,background:q.bg,color:q.color,border:`1px solid ${q.border}`,borderRadius:5,padding:"2px 7px",whiteSpace:"nowrap"}}>{q.icon} {q.short}</span>
                      <div style={{fontSize:12,color:"#4A5568",fontWeight:600}}>👤 {task.assignee}</div>
                      <div style={{display:"flex",gap:5,alignItems:"center"}}>
                        {task.dueDate?<><span style={{fontSize:11,color:C.muted}}>{fmtDate(task.dueDate)}</span><DaysChip dateStr={task.dueDate}/></>:<span style={{fontSize:11,color:C.border}}>—</span>}
                      </div>
                      <div onClick={e=>{e.stopPropagation();if(subs.length)setExpanded(expandedTask===task.id?null:task.id);}} style={{fontSize:11,fontWeight:700,color:subs.length?"#16A34A":C.muted,cursor:subs.length?"pointer":"default"}}>
                        {subs.length?`☑ ${subs.filter(s=>s.done).length}/${subs.length}`:"—"}
                      </div>
                      <span style={{fontSize:11,fontWeight:700,background:task.done?"#DCFCE7":"#F1F5F9",color:task.done?"#16A34A":C.muted,borderRadius:5,padding:"2px 7px"}}>{task.done?"✓ Fatto":"In corso"}</span>
                    </div>
                    {/* Inline subtask expand in list */}
                    {expandedTask===task.id&&subs.length>0&&(
                      <div style={{background:"#F8FAFC",borderBottom:`1px solid ${C.border}`,padding:"8px 16px 10px 56px"}}>
                        {subs.map(s=>(
                          <div key={s.id} style={{display:"flex",alignItems:"center",gap:8,marginBottom:5}}>
                            <div onClick={()=>toggleSubtask(task.id,s.id)} style={{width:14,height:14,borderRadius:3,border:`2px solid ${s.done?"#16A34A":C.border}`,background:s.done?"#16A34A":"transparent",display:"flex",alignItems:"center",justifyContent:"center",flexShrink:0,cursor:"pointer"}}>
                              {s.done&&<span style={{color:"#fff",fontSize:9,fontWeight:900}}>✓</span>}
                            </div>
                            <span style={{fontSize:12,color:s.done?C.muted:C.dark,textDecoration:s.done?"line-through":"none"}}>{s.title}</span>
                          </div>
                        ))}
                      </div>
                    )}
                  </div>
                );
              })}
            </div>
          )}

          {/* ── CALENDAR ── */}
          {view==="calendar"&&(
            <div style={{background:"#fff",borderRadius:14,overflow:"hidden",boxShadow:"0 2px 8px rgba(0,0,0,0.06)"}}>
              <div style={{display:"flex",alignItems:"center",justifyContent:"space-between",padding:"14px 20px",background:C.dark,color:"#F8FAFC"}}>
                <button onClick={()=>setCalMonth(p=>{const d=new Date(p.y,p.m-1);return{y:d.getFullYear(),m:d.getMonth()};})} style={{background:"none",border:"none",color:"#F8FAFC",fontSize:22,cursor:"pointer"}}>‹</button>
                <span style={{fontFamily:"'Lora',serif",fontSize:16,fontWeight:700}}>{MONTHS_IT[calMonth.m]} {calMonth.y}</span>
                <button onClick={()=>setCalMonth(p=>{const d=new Date(p.y,p.m+1);return{y:d.getFullYear(),m:d.getMonth()};})} style={{background:"none",border:"none",color:"#F8FAFC",fontSize:22,cursor:"pointer"}}>›</button>
              </div>
              <div style={{display:"grid",gridTemplateColumns:"repeat(7,1fr)",background:"#F1F5F9"}}>
                {DOW_IT.map(d=><div key={d} style={{padding:"7px 4px",textAlign:"center",fontSize:10,fontWeight:700,color:C.muted,textTransform:"uppercase"}}>{d}</div>)}
              </div>
              <div style={{display:"grid",gridTemplateColumns:"repeat(7,1fr)",gap:1,background:"#F1F5F9"}}>
                {Array.from({length:firstDayOf(calMonth.y,calMonth.m)}).map((_,i)=><div key={"e"+i} style={{background:"#FAFAF9",minHeight:80}}/>)}
                {Array.from({length:daysInMonth(calMonth.y,calMonth.m)}).map((_,i)=>{
                  const day=i+1;
                  const ds=`${calMonth.y}-${String(calMonth.m+1).padStart(2,"0")}-${String(day).padStart(2,"0")}`;
                  const isToday=ds===todayStr;
                  const dt=filtered.filter(t=>t.dueDate===ds);
                  return(
                    <div key={day} style={{background:isToday?"#EFF6FF":"#fff",minHeight:80,padding:"5px 7px",border:isToday?"2px solid #2563EB":"none",boxSizing:"border-box"}}>
                      <div style={{fontSize:11,fontWeight:isToday?800:600,color:isToday?"#2563EB":C.muted,marginBottom:3}}>{day}</div>
                      {dt.map(t=>{
                        const q=QUADRANTS.find(x=>x.id===t.quadrant);
                        const subs=t.subtasks||[];
                        return(
                          <div key={t.id} onClick={()=>openEdit(t)} style={{fontSize:10,fontWeight:600,background:q.bg,color:q.color,borderLeft:`3px solid ${q.border}`,borderRadius:"0 3px 3px 0",padding:"1px 4px",marginBottom:2,overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap",cursor:"pointer"}} title={t.title}>
                            {t.title}{subs.length?` ☑${subs.filter(s=>s.done).length}/${subs.length}`:""}
                          </div>
                        );
                      })}
                    </div>
                  );
                })}
              </div>
            </div>
          )}
        </div>

        {/* ── MODAL ── */}
        {showForm&&(
          <div onClick={e=>{if(e.target===e.currentTarget)setShowForm(false);}} style={{position:"fixed",inset:0,background:"rgba(15,23,42,0.65)",zIndex:100,display:"flex",alignItems:"center",justifyContent:"center",padding:16,backdropFilter:"blur(4px)"}}>
            <div style={{background:"#fff",borderRadius:18,padding:26,width:"100%",maxWidth:500,boxShadow:"0 20px 60px rgba(0,0,0,0.25)",maxHeight:"90vh",overflowY:"auto"}}>
              <div style={{fontFamily:"'Lora',serif",fontSize:17,fontWeight:700,marginBottom:18}}>{editId?"Modifica attività":"✨ Nuova attività"}</div>

              {/* Title */}
              <div style={{marginBottom:13}}>
                <label style={{fontSize:11,fontWeight:700,color:C.muted,textTransform:"uppercase",letterSpacing:0.7,display:"block",marginBottom:4}}>Titolo *</label>
                <input style={{width:"100%",border:`2px solid ${C.border}`,borderRadius:9,padding:"8px 11px",fontSize:14,color:C.dark,outline:"none",boxSizing:"border-box",fontFamily:"inherit"}} value={form.title} onChange={e=>setForm(f=>({...f,title:e.target.value}))} placeholder="Descrivi l'attività..." autoFocus/>
              </div>

              {/* Quadrant + Assignee */}
              <div style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:12,marginBottom:13}}>
                <div>
                  <label style={{fontSize:11,fontWeight:700,color:C.muted,textTransform:"uppercase",letterSpacing:0.7,display:"block",marginBottom:4}}>Priorità</label>
                  <select style={{width:"100%",border:`2px solid ${C.border}`,borderRadius:9,padding:"8px 11px",fontSize:13,color:C.dark,outline:"none",background:"#fff",boxSizing:"border-box",fontFamily:"inherit"}} value={form.quadrant} onChange={e=>setForm(f=>({...f,quadrant:e.target.value}))}>
                    {QUADRANTS.map(q=><option key={q.id} value={q.id}>{q.icon} {q.short}</option>)}
                  </select>
                </div>
                <div>
                  <label style={{fontSize:11,fontWeight:700,color:C.muted,textTransform:"uppercase",letterSpacing:0.7,display:"block",marginBottom:4}}>Assegnato a</label>
                  <select style={{width:"100%",border:`2px solid ${C.border}`,borderRadius:9,padding:"8px 11px",fontSize:13,color:C.dark,outline:"none",background:"#fff",boxSizing:"border-box",fontFamily:"inherit"}} value={form.assignee} onChange={e=>setForm(f=>({...f,assignee:e.target.value}))}>
                    {TEAM.map(t=><option key={t} value={t}>{t}</option>)}
                  </select>
                </div>
              </div>

              {/* Due date */}
              <div style={{marginBottom:13}}>
                <label style={{fontSize:11,fontWeight:700,color:C.muted,textTransform:"uppercase",letterSpacing:0.7,display:"block",marginBottom:4}}>Scadenza</label>
                <input type="date" style={{width:"100%",border:`2px solid ${C.border}`,borderRadius:9,padding:"8px 11px",fontSize:14,color:C.dark,outline:"none",boxSizing:"border-box",fontFamily:"inherit"}} value={form.dueDate} onChange={e=>setForm(f=>({...f,dueDate:e.target.value}))}/>
              </div>

              {/* Notes */}
              <div style={{marginBottom:16}}>
                <label style={{fontSize:11,fontWeight:700,color:C.muted,textTransform:"uppercase",letterSpacing:0.7,display:"block",marginBottom:4}}>Note</label>
                <textarea style={{width:"100%",border:`2px solid ${C.border}`,borderRadius:9,padding:"8px 11px",fontSize:14,color:C.dark,outline:"none",boxSizing:"border-box",fontFamily:"inherit",resize:"vertical",minHeight:56}} value={form.notes} onChange={e=>setForm(f=>({...f,notes:e.target.value}))} placeholder="Dettagli aggiuntivi..."/>
              </div>

              {/* ── SUBTASKS ── */}
              <div style={{marginBottom:16}}>
                <label style={{fontSize:11,fontWeight:700,color:C.muted,textTransform:"uppercase",letterSpacing:0.7,display:"block",marginBottom:8}}>☑ Sotto-attività / Checklist</label>
                <div style={{background:"#F8FAFC",borderRadius:10,border:`1px solid ${C.border}`,padding:"10px 12px",marginBottom:8}}>
                  {(form.subtasks||[]).length===0&&<div style={{fontSize:12,color:C.muted,textAlign:"center",padding:"6px 0"}}>Nessuna sotto-attività ancora.</div>}
                  {(form.subtasks||[]).map((s,si)=>(
                    <div key={s.id} style={{display:"flex",alignItems:"center",gap:8,marginBottom:6}}>
                      <div onClick={()=>setForm(f=>({...f,subtasks:f.subtasks.map((x,xi)=>xi===si?{...x,done:!x.done}:x)}))} style={{width:15,height:15,borderRadius:3,border:`2px solid ${s.done?"#16A34A":C.border}`,background:s.done?"#16A34A":"transparent",display:"flex",alignItems:"center",justifyContent:"center",flexShrink:0,cursor:"pointer"}}>
                        {s.done&&<span style={{color:"#fff",fontSize:9,fontWeight:900}}>✓</span>}
                      </div>
                      <input value={s.title} onChange={e=>setForm(f=>({...f,subtasks:f.subtasks.map((x,xi)=>xi===si?{...x,title:e.target.value}:x)}))}
                        style={{flex:1,border:`1px solid ${C.border}`,borderRadius:6,padding:"4px 8px",fontSize:12,outline:"none",fontFamily:"inherit",textDecoration:s.done?"line-through":"none",color:s.done?C.muted:C.dark}}/>
                      <button onClick={()=>setForm(f=>({...f,subtasks:f.subtasks.filter((_,xi)=>xi!==si)}))} style={{background:"none",border:"none",cursor:"pointer",color:"#FCA5A5",fontSize:14,padding:0}}>✕</button>
                    </div>
                  ))}
                </div>
                <div style={{display:"flex",gap:6}}>
                  <input value={newSubInput} onChange={e=>setNewSubInput(e.target.value)}
                    onKeyDown={e=>{if(e.key==="Enter"){e.preventDefault();addFormSub();}}}
                    placeholder="Aggiungi sotto-attività..." style={{flex:1,border:`2px solid ${C.border}`,borderRadius:9,padding:"7px 10px",fontSize:13,outline:"none",fontFamily:"inherit"}}/>
                  <button onClick={addFormSub} style={{background:C.dark,color:"#fff",border:"none",borderRadius:9,padding:"7px 14px",fontSize:13,fontWeight:700,cursor:"pointer"}}>+</button>
                </div>
              </div>

              <div style={{display:"flex",gap:8,justifyContent:"flex-end"}}>
                {editId&&<button onClick={()=>{deleteTask(editId);setShowForm(false);}} style={{padding:"8px 14px",background:"#FFF5F5",color:"#E53E3E",border:"2px solid #FEB2B2",borderRadius:9,cursor:"pointer",fontSize:13,fontWeight:700}}>🗑 Elimina</button>}
                <button onClick={()=>setShowForm(false)} style={{padding:"8px 16px",border:`2px solid ${C.border}`,background:"transparent",borderRadius:9,cursor:"pointer",fontSize:13,fontWeight:600,color:C.muted}}>Annulla</button>
                <button onClick={submitForm} style={{padding:"8px 20px",background:C.dark,color:"#fff",border:"none",borderRadius:9,cursor:"pointer",fontSize:13,fontWeight:700}}>{editId?"Salva":"Crea attività"}</button>
              </div>
            </div>
          </div>
        )}

        <style>{`@keyframes pulse{0%,100%{opacity:1}50%{opacity:0.6}}`}</style>
      </div>
    </>
  );
}
