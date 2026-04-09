<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>勤怠管理アプリ</title>
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;background:#f5f5f5;min-height:100vh}
select,input,button{font-family:inherit}
input,select{width:100%;padding:10px 12px;border:0.5px solid #ccc;border-radius:8px;font-size:14px;background:#fff;outline:none}
input:focus,select:focus{border-color:#888}
button{padding:8px 16px;border:0.5px solid #ccc;border-radius:8px;background:#fff;cursor:pointer;font-size:13px}
button:hover{background:#f0f0f0}
.tab-active{background:#f0f0f0!important;border-color:#888!important;font-weight:500!important}
.clock-btn{width:100%;padding:22px;font-size:22px;font-weight:500;border:none;border-radius:12px;cursor:pointer;color:#fff;letter-spacing:.05em}
.clock-btn:disabled{opacity:.35;cursor:default}
.mono{font-family:'SF Mono',Monaco,monospace}
th{text-align:left;padding:8px 10px;font-weight:500;color:#666;font-size:12px}
td{padding:10px;font-size:13px;border-bottom:0.5px solid #eee}
tr:last-child td{border-bottom:none}
.metric{background:#f5f5f5;border-radius:8px;padding:14px 16px}
.metric-label{font-size:11px;color:#888;margin-bottom:4px}
.metric-value{font-size:22px;font-weight:500}
.card{background:#fff;border:0.5px solid #ddd;border-radius:12px;padding:16px}
.dot{display:inline-block;width:7px;height:7px;border-radius:50%;margin-right:5px;vertical-align:middle}
.del-btn{background:transparent!important;border:none!important;color:#ccc;cursor:pointer;font-size:16px;padding:2px 6px}
.del-btn:hover{color:#E24B4A!important;background:transparent!important}
#root{padding:0}
</style>
</head>
<body>
<div id="root"></div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js" crossorigin></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js" crossorigin></script>
<script>

const {useState,useEffect}=React;
const e=React.createElement;

const R15up=d=>{const ms=15*60*1e3;return new Date(Math.ceil(new Date(d).getTime()/ms)*ms)};
const R15down=d=>{const ms=15*60*1e3;return new Date(Math.floor(new Date(d).getTime()/ms)*ms)};
const FT=d=>{if(!d)return"--:--";return new Date(d).toLocaleTimeString("ja-JP",{hour:"2-digit",minute:"2-digit",hour12:false})};
const FD=d=>{if(!d)return"";return new Date(d).toLocaleDateString("ja-JP",{month:"2-digit",day:"2-digit"})};
const DH=(ci,co)=>{if(!ci||!co)return 0;return Math.round((new Date(co)-new Date(ci))/6e4/15)*0.25};

function gasAPI(url){
  return async function call(params){
    const q=new URLSearchParams(params).toString();
    const res=await fetch(`${url}?${q}`,{redirect:"follow"});
    if(!res.ok)throw new Error(`HTTP ${res.status}`);
    return res.json();
  };
}

function App(){
  const [view,setView]=useState("clock");
  const [gasUrl,setGasUrl]=useState(()=>localStorage.getItem("gasUrl")||"");
  const [gasInput,setGasInput]=useState(()=>localStorage.getItem("gasUrl")||"");
  const [connected,setConnected]=useState(false);
  const [syncing,setSyncing]=useState(false);
  const [syncErr,setSyncErr]=useState("");

  const [wps,setWps]=useState(()=>{try{return JSON.parse(localStorage.getItem("workplaces"))||[];}catch{return[];}});
  const [emps,setEmps]=useState([]);
  const [recs,setRecs]=useState([]);

  const [tab,setTab]=useState("today");
  const [pass,setPass]=useState("");
  const [unlocked,setUnlocked]=useState(false);
  const [now,setNow]=useState(new Date());
  const [selEmp,setSelEmp]=useState("");
  const [selWp,setSelWp]=useState("");
  const [msg,setMsg]=useState({t:"",c:""});
  const [newEmp,setNewEmp]=useState({name:"",rate:"",dept:""});
  const [newWp,setNewWp]=useState("");
  const [payMonth,setPayMonth]=useState(()=>{const d=new Date();return`${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,"0")}`});
  const [editEmp,setEditEmp]=useState(null);
  const [editWp,setEditWp]=useState(null);
  const [editWpVal,setEditWpVal]=useState("");

  useEffect(()=>{const t=setInterval(()=>setNow(new Date()),1e3);return()=>clearInterval(t)},[]);
  useEffect(()=>{localStorage.setItem("workplaces",JSON.stringify(wps));},[wps]);
  useEffect(()=>{if(wps.length>0&&!selWp)setSelWp(wps[0].id);},[wps]);
  const [lastSync,setLastSync]=useState("");
  const [autoCount,setAutoCount]=useState(30);
  useEffect(()=>{
    const t=setInterval(()=>{
      setAutoCount(p=>{
        if(p<=1){
          if(gasUrl){
            const doLoad=async()=>{
              const api2=gasAPI(gasUrl);
              try{
                const[er,rr]=await Promise.all([api2({action:"employees"}),api2({action:"records"})]);
                if(er?.ok&&er.data.length>0)setEmps(er.data.map(e=>({...e,rate:Number(e.rate)})));
                if(rr?.ok)setRecs(rr.data.map(r=>({id:r.id,empId:r.empId,wpId:r.wpId,ci:r.clockIn,co:r.clockOut||null})));
                setLastSync(new Date().toLocaleTimeString("ja-JP",{hour:"2-digit",minute:"2-digit",second:"2-digit"}));
                setConnected(true);
              }catch{}
            };
            doLoad();
          }
          return 30;
        }
        return p-1;
      });
    },1e3);
    return()=>clearInterval(t);
  },[gasUrl]);

  const api=gasUrl?gasAPI(gasUrl):null;
  const callAPI=async(params)=>{
    if(!api)return null;
    setSyncing(true);setSyncErr("");
    try{const r=await api(params);setSyncing(false);return r;}
    catch(err){setSyncErr(err.message);setSyncing(false);return null;}
  };

  const loadSheets=async()=>{
    if(!api)return;
    const[er,rr]=await Promise.all([callAPI({action:"employees"}),callAPI({action:"records"})]);
    if(er?.ok&&er.data.length>0)setEmps(er.data.map(e=>({...e,rate:Number(e.rate)})));
    if(rr?.ok)setRecs(rr.data.map(r=>({id:r.id,empId:r.empId,wpId:r.wpId,ci:r.clockIn,co:r.clockOut||null})));
    if(er?.ok||rr?.ok)setConnected(true);
  };

  const saveUrl=async()=>{
    localStorage.setItem("gasUrl",gasInput);
    setGasUrl(gasInput);
    setSyncErr("");setSyncing(true);
    try{
      const r=await gasAPI(gasInput)({action:"employees"});
      if(r.ok){setConnected(true);if(r.data.length>0)setEmps(r.data.map(e=>({...e,rate:Number(e.rate)})));}
      else setSyncErr(r.error||"エラー");
    }catch(err){setSyncErr("接続失敗: "+err.message);}
    setSyncing(false);
  };

  const todayStr=new Date().toDateString();
  const getActive=id=>recs.find(r=>r.empId===id&&new Date(r.ci).toDateString()===todayStr&&!r.co);
  const getToday=id=>recs.find(r=>r.empId===id&&new Date(r.ci).toDateString()===todayStr);
  const EN=id=>emps.find(e=>e.id===id)?.name??id;
  const WN=id=>wps.find(w=>w.id===id)?.name??id;
  const SM=(t,c)=>{setMsg({t,c:c||"#185FA5"});setTimeout(()=>setMsg({t:"",c:""}),4e3)};

  const doClock=async()=>{
    if(!selEmp){SM("社員を選択してください","#A32D2D");return}
    if(!selWp){SM("現場を選択してください","#A32D2D");return}
    const ts=new Date().toISOString();
    const active=getActive(selEmp);
    if(active){
      const co=R15down(now).toISOString();
      setRecs(p=>p.map(r=>r.id===active.id?{...r,co}:r));
      SM(`退勤しました：${FT(co)}`,"#0F6E56");
      await callAPI({action:"clock_out",empId:selEmp,timestamp:ts});
    } else {
      if(recs.find(r=>r.empId===selEmp&&new Date(r.ci).toDateString()===todayStr&&r.co)){SM("本日は退勤済みです","#A32D2D");return}
      const ci=R15up(now).toISOString();
      setRecs(p=>[...p,{id:`${selEmp}-${Date.now()}`,empId:selEmp,wpId:selWp,ci,co:null}]);
      SM(`出勤しました：${FT(ci)}`,"#185FA5");
      await callAPI({action:"clock_in",empId:selEmp,wpId:selWp,timestamp:ts});
    }
  };

  const exportCSV=()=>{
    const rows=recs.filter(r=>r.co).map(r=>{
      const emp=emps.find(em=>em.id===r.empId);const h2=DH(r.ci,r.co);
      return`${r.empId},${emp?.name??""},${FD(r.ci)},${FT(r.ci)},${FT(r.co)},${h2.toFixed(2)},${emp?.rate??0},${Math.round(h2*(emp?.rate??0))}`;
    });
    const csv=["社員ID,氏名,日付,出勤,退勤,時間(h),時給,給与",...rows].join("\n");
    const a=document.createElement("a");a.href="data:text/csv;charset=utf-8,\uFEFF"+encodeURIComponent(csv);a.download=`勤怠_${payMonth}.csv`;a.click();
  };

  const payData=emps.map(emp=>{
    const[yr,mo]=payMonth.split("-").map(Number);
    const mrs=recs.filter(r=>{const d=new Date(r.ci);return r.empId===emp.id&&d.getFullYear()===yr&&d.getMonth()+1===mo&&r.co});
    const hrs=mrs.reduce((s,r)=>s+DH(r.ci,r.co),0);
    return{...emp,days:mrs.length,hrs,pay:Math.round(hrs*emp.rate)};
  });

  const todayRecs=recs.filter(r=>new Date(r.ci).toDateString()===todayStr);
  const active=selEmp?getActive(selEmp):null;
  const isClockedIn=!!active;
  const todayRec=selEmp?getToday(selEmp):null;
  const isDone=!!(todayRec?.co);

  if(view==="admin"&&!unlocked) return e("div",{style:{maxWidth:340,margin:"60px auto",padding:"0 20px"}},
    e("p",{style:{fontSize:12,color:"#888",marginBottom:8}},"管理者ログイン"),
    e("input",{type:"password",placeholder:"パスワード（初期値: admin123）",value:pass,
      onChange:ev=>setPass(ev.target.value),onKeyDown:ev=>ev.key==="Enter"&&pass==="admin123"&&setUnlocked(true),style:{width:"100%",marginBottom:10}}),
    e("button",{onClick:()=>pass==="admin123"&&setUnlocked(true),style:{width:"100%"}},"ログイン"),
    e("p",{style:{marginTop:16,fontSize:13}},e("span",{onClick:()=>setView("clock"),style:{cursor:"pointer",color:"#185FA5"}},"← 打刻画面"))
  );

  if(view==="admin"&&unlocked){
    const TABS=[{id:"today",label:"今日"},{id:"records",label:"記録"},{id:"employees",label:"社員"},{id:"workplaces",label:"現場"},{id:"payroll",label:"給与"},{id:"qr",label:"QR"},{id:"settings",label:"設定"}];
    return e("div",{style:{maxWidth:700,margin:"0 auto",padding:"20px 16px",background:"#fff",minHeight:"100vh"}},
      e("div",{style:{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:20,flexWrap:"wrap",gap:8}},
        e("h1",{style:{fontSize:17,fontWeight:500}},"管理画面"),
        e("div",{style:{display:"flex",gap:10,alignItems:"center"}},
          gasUrl&&e("span",{style:{fontSize:11,display:"flex",alignItems:"center",gap:6,color:syncErr?"#A32D2D":connected?"#0F6E56":"#888"}},
            e("span",{className:"dot",style:{background:syncing?"#EF9F27":syncErr?"#E24B4A":connected?"#1D9E75":"#888"}}),
            syncing?"同期中...":syncErr?"エラー":connected?`自動更新 ${autoCount}秒後`:"未接続",
            lastSync&&!syncing&&e("span",{style:{color:"#aaa",fontSize:10}},lastSync+"更新")),
          e("span",{onClick:()=>{setView("clock");setUnlocked(false)},style:{fontSize:13,cursor:"pointer",color:"#185FA5"}},"← 打刻画面")
        )
      ),
      e("div",{style:{display:"flex",gap:4,marginBottom:20,borderBottom:"0.5px solid #eee",paddingBottom:8,flexWrap:"wrap"}},
        TABS.map(t=>e("button",{key:t.id,onClick:()=>setTab(t.id),className:tab===t.id?"tab-active":"",
          style:{padding:"6px 14px",fontSize:13,background:"transparent",border:"0.5px solid transparent",borderRadius:"8px",cursor:"pointer"}},t.label))
      ),

      tab==="today"&&e("div",null,
        e("div",{style:{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:12}},
          e("p",{style:{fontSize:12,color:"#888"}},FD(now)+"の出勤状況"),
          gasUrl&&e("button",{onClick:loadSheets,style:{fontSize:12}},syncing?"同期中...":"シートから読み込み ↗")
        ),
        e("div",{style:{display:"grid",gridTemplateColumns:"repeat(3,1fr)",gap:10,marginBottom:20}},
          [{l:"出勤中",v:todayRecs.filter(r=>!r.co).length,c:"#0F6E56"},{l:"退勤済",v:todayRecs.filter(r=>r.co).length,c:null},{l:"本日合計",v:todayRecs.length,c:null}]
          .map(({l,v,c})=>e("div",{key:l,className:"metric"},e("div",{className:"metric-label"},l),e("div",{className:"metric-value",style:c?{color:c}:{}},v)))
        ),
        e("table",{style:{width:"100%",borderCollapse:"collapse"}},
          e("thead",null,e("tr",{style:{borderBottom:"0.5px solid #eee"}},["社員名","現場","出勤","退勤","時間"].map(h=>e("th",{key:h},h)))),
          e("tbody",null,
            !todayRecs.length&&e("tr",null,e("td",{colSpan:5,style:{padding:"20px 10px",color:"#888"}},"本日の記録はありません")),
            todayRecs.map(r=>e("tr",{key:r.id},
              e("td",{style:{fontWeight:500}},EN(r.empId)),
              e("td",{style:{color:"#888",fontSize:12}},WN(r.wpId)),
              e("td",{className:"mono"},FT(r.ci)),
              e("td",null,r.co?e("span",{className:"mono"},FT(r.co)):e("span",{style:{fontSize:11,padding:"2px 8px",background:"#E1F5EE",color:"#0F6E56",borderRadius:4}},"出勤中")),
              e("td",{className:"mono"},r.co?DH(r.ci,r.co).toFixed(2)+"h":"-")
            ))
          )
        )
      ),

      tab==="records"&&e("div",{style:{overflowX:"auto"}},
        e("table",{style:{width:"100%",borderCollapse:"collapse"}},
          e("thead",null,e("tr",{style:{borderBottom:"0.5px solid #eee"}},["日付","社員名","出勤","退勤","時間","給与"].map(h=>e("th",{key:h},h)))),
          e("tbody",null,
            !recs.filter(r=>r.co).length&&e("tr",null,e("td",{colSpan:6,style:{padding:"20px 10px",color:"#888"}},"記録がありません")),
            [...recs].reverse().filter(r=>r.co).slice(0,30).map(r=>{
              const emp=emps.find(em=>em.id===r.empId);const hrs=DH(r.ci,r.co);
              return e("tr",{key:r.id},
                e("td",{className:"mono",style:{color:"#888"}},FD(r.ci)),
                e("td",{style:{fontWeight:500}},EN(r.empId)),
                e("td",{className:"mono"},FT(r.ci)),e("td",{className:"mono"},FT(r.co)),
                e("td",{className:"mono"},hrs.toFixed(2)+"h"),
                e("td",{className:"mono",style:{fontWeight:500}},"¥"+Math.round(hrs*(emp?.rate??0)).toLocaleString())
              );
            })
          )
        )
      ),

      tab==="employees"&&e("div",null,
        e("table",{style:{width:"100%",borderCollapse:"collapse",marginBottom:24}},
          e("thead",null,e("tr",{style:{borderBottom:"0.5px solid #eee"}},["ID","氏名","部署","時給","",""].map(h=>e("th",{key:h},h)))),
          e("tbody",null,
            !emps.length&&e("tr",null,e("td",{colSpan:6,style:{padding:"20px 10px",color:"#888"}},"社員が登録されていません")),
            emps.map(emp=>e("tr",{key:emp.id},
              e("td",{className:"mono",style:{color:"#888",fontSize:12}},emp.id),
              e("td",{style:{fontWeight:500}},editEmp===emp.id?e("input",{defaultValue:emp.name,id:`en-${emp.id}`,style:{width:120,fontSize:13}}):emp.name),
              e("td",{style:{color:"#888"}},emp.dept),
              e("td",{className:"mono"},editEmp===emp.id?e("input",{defaultValue:emp.rate,id:`er-${emp.id}`,type:"number",style:{width:80,fontSize:13}}):"¥"+emp.rate.toLocaleString()),
              e("td",null,editEmp===emp.id
                ?e("button",{style:{fontSize:12},onClick:async()=>{
                  const name=document.getElementById(`en-${emp.id}`)?.value||emp.name;
                  const rate=Number(document.getElementById(`er-${emp.id}`)?.value)||emp.rate;
                  setEmps(p=>p.map(e2=>e2.id===emp.id?{...e2,name,rate}:e2));setEditEmp(null);
                  await callAPI({action:"update_employee",id:emp.id,name,rate});
                }},"保存")
                :e("button",{style:{fontSize:12},onClick:()=>setEditEmp(emp.id)},"編集")
              ),
              e("td",null,e("button",{className:"del-btn",title:"削除",onClick:()=>{if(confirm(emp.name+"を削除しますか？"))setEmps(p=>p.filter(e2=>e2.id!==emp.id))}},"✕"))
            ))
          )
        ),
        e("div",{className:"card"},
          e("p",{style:{fontSize:13,fontWeight:500,marginBottom:12}},"社員を追加"),
          e("div",{style:{display:"grid",gridTemplateColumns:"1fr 1fr",gap:8,marginBottom:8}},
            e("input",{placeholder:"氏名",value:newEmp.name,onChange:ev=>setNewEmp(p=>({...p,name:ev.target.value}))}),
            e("input",{placeholder:"時給（例: 1200）",type:"number",value:newEmp.rate,onChange:ev=>setNewEmp(p=>({...p,rate:ev.target.value}))})
          ),
          e("input",{placeholder:"部署・現場名",value:newEmp.dept,onChange:ev=>setNewEmp(p=>({...p,dept:ev.target.value})),style:{width:"100%",marginBottom:8}}),
          e("button",{style:{width:"100%"},onClick:async()=>{
            if(!newEmp.name||!newEmp.rate)return;
            const id=`E${String(Date.now()).slice(-5)}`;
            const emp={id,name:newEmp.name,rate:Number(newEmp.rate),dept:newEmp.dept};
            setEmps(p=>[...p,emp]);setNewEmp({name:"",rate:"",dept:""});
            await callAPI({action:"add_employee",...emp});
          }},"追加する")
        )
      ),

      tab==="workplaces"&&e("div",null,
        e("table",{style:{width:"100%",borderCollapse:"collapse",marginBottom:24}},
          e("thead",null,e("tr",{style:{borderBottom:"0.5px solid #eee"}},["ID","現場名","",""].map(h=>e("th",{key:h},h)))),
          e("tbody",null,
            !wps.length&&e("tr",null,e("td",{colSpan:4,style:{padding:"20px 10px",color:"#888"}},"現場が登録されていません")),
            wps.map(wp=>e("tr",{key:wp.id},
              e("td",{className:"mono",style:{color:"#888",fontSize:12}},wp.id),
              e("td",{style:{fontWeight:500}},
                editWp===wp.id
                  ?e("input",{value:editWpVal,onChange:ev=>setEditWpVal(ev.target.value),style:{width:200,fontSize:13}})
                  :wp.name
              ),
              e("td",null,
                editWp===wp.id
                  ?e("button",{style:{fontSize:12},onClick:()=>{setWps(p=>p.map(w=>w.id===wp.id?{...w,name:editWpVal}:w));setEditWp(null);}},"保存")
                  :e("button",{style:{fontSize:12},onClick:()=>{setEditWp(wp.id);setEditWpVal(wp.name);}},"編集")
              ),
              e("td",null,e("button",{className:"del-btn",title:"削除",onClick:()=>{if(confirm(wp.name+"を削除しますか？"))setWps(p=>p.filter(w=>w.id!==wp.id));}},"✕"))
            ))
          )
        ),
        e("div",{className:"card"},
          e("p",{style:{fontSize:13,fontWeight:500,marginBottom:12}},"現場を追加"),
          e("input",{placeholder:"現場名（例: 渋谷現場）",value:newWp,onChange:ev=>setNewWp(ev.target.value),
            onKeyDown:ev=>{if(ev.key==="Enter"&&newWp.trim()){const id=`W${String(Date.now()).slice(-5)}`;setWps(p=>[...p,{id,name:newWp.trim()}]);setNewWp("");}},
            style:{width:"100%",marginBottom:8}}),
          e("button",{style:{width:"100%"},onClick:()=>{
            if(!newWp.trim())return;
            const id=`W${String(Date.now()).slice(-5)}`;
            setWps(p=>[...p,{id,name:newWp.trim()}]);setNewWp("");
          }},"追加する")
        )
      ),

      tab==="payroll"&&e("div",null,
        e("div",{style:{display:"flex",gap:12,alignItems:"center",marginBottom:20,flexWrap:"wrap"}},
          e("input",{type:"month",value:payMonth,onChange:ev=>setPayMonth(ev.target.value),style:{width:160}}),
          e("button",{onClick:exportCSV},"CSVエクスポート ↗")
        ),
        e("table",{style:{width:"100%",borderCollapse:"collapse"}},
          e("thead",null,e("tr",{style:{borderBottom:"0.5px solid #eee"}},["氏名","出勤日数","合計時間","時給","支給額"].map(h=>e("th",{key:h},h)))),
          e("tbody",null,
            !emps.length&&e("tr",null,e("td",{colSpan:5,style:{padding:"20px 10px",color:"#888"}},"社員が登録されていません")),
            payData.map(emp=>e("tr",{key:emp.id},
              e("td",{style:{fontWeight:500,padding:"12px 10px"}},emp.name),
              e("td",{style:{padding:"12px 10px"}},emp.days+"日"),
              e("td",{className:"mono",style:{padding:"12px 10px"}},emp.hrs.toFixed(2)+"h"),
              e("td",{className:"mono",style:{padding:"12px 10px"}},"¥"+emp.rate.toLocaleString()),
              e("td",{className:"mono",style:{padding:"12px 10px",fontWeight:500,fontSize:15}},"¥"+emp.pay.toLocaleString())
            ))
          ),
          e("tfoot",null,e("tr",{style:{borderTop:"1px solid #ddd"}},
            e("td",{colSpan:4,style:{padding:"12px 10px",fontWeight:500}},"合計支給額"),
            e("td",{className:"mono",style:{padding:"12px 10px",fontWeight:500,fontSize:16}},"¥"+payData.reduce((s,emp)=>s+emp.pay,0).toLocaleString())
          ))
        )
      ),

      tab==="qr"&&e("div",null,
        wps.length===0&&e("p",{style:{color:"#888",fontSize:13,padding:"20px 0"}},"先に「現場」タブで現場を追加してください。"),
        e("div",{style:{display:"grid",gridTemplateColumns:"repeat(auto-fit,minmax(220px,1fr))",gap:16}},
          wps.map(wp=>{
            const url=`https://sdsdsd3104.github.io/kintai/?wp=${wp.id}`;
            return e("div",{key:wp.id,className:"card",style:{textAlign:"center"}},
              e("p",{style:{fontWeight:500,marginBottom:12,fontSize:14}},wp.name),
              e("img",{src:`https://api.qrserver.com/v1/create-qr-code/?size=180x180&data=${encodeURIComponent(url)}`,alt:wp.name,style:{width:180,height:180,border:"0.5px solid #ddd",borderRadius:4}}),
              e("p",{style:{fontSize:10,color:"#888",marginTop:10,wordBreak:"break-all"}},url),
              e("button",{style:{marginTop:10,width:"100%",fontSize:12},onClick:()=>{const a=document.createElement("a");a.href=`https://api.qrserver.com/v1/create-qr-code/?size=400x400&data=${encodeURIComponent(url)}`;a.download=`QR_${wp.name}.png`;a.target="_blank";a.click();}},"印刷用 ↗")
            );
          })
        )
      ),

      tab==="settings"&&e("div",null,
        e("div",{className:"card",style:{marginBottom:16}},
          e("p",{style:{fontWeight:500,fontSize:14,marginBottom:4}},"Google スプレッドシート連携"),
          e("p",{style:{fontSize:12,color:"#888",lineHeight:1.7,marginBottom:14}},"デプロイで取得したURLをここに貼り付けて「保存して接続テスト」を押してください。"),
          e("input",{placeholder:"https://script.google.com/macros/s/.../exec",value:gasInput,
            onChange:ev=>setGasInput(ev.target.value),style:{width:"100%",marginBottom:8,fontSize:12}}),
          e("div",{style:{display:"flex",gap:8}},
            e("button",{onClick:saveUrl,style:{flex:1}},syncing?"テスト中...":"保存して接続テスト"),
            connected&&e("button",{onClick:loadSheets,style:{flex:1}},"シートから読み込み ↗")
          ),
          connected&&!syncErr&&e("p",{style:{marginTop:10,fontSize:12,color:"#0F6E56"}},"✓ 接続成功！打刻データがリアルタイムでシートに書き込まれます"),
          syncErr&&e("div",{style:{marginTop:10,padding:"10px 12px",background:"#FCEBEB",borderRadius:"8px"}},
            e("p",{style:{fontSize:12,color:"#A32D2D",fontWeight:500}},"接続エラー: "+syncErr)
          )
        )
      )
    );
  }

  const rounded=isClockedIn?R15down(now):R15up(now);
  return e("div",{style:{maxWidth:390,margin:"0 auto",padding:"32px 20px",display:"flex",flexDirection:"column",minHeight:"100vh",background:"#fff"}},
    e("div",{style:{textAlign:"center",marginBottom:36}},
      e("p",{style:{fontSize:11,color:"#888",letterSpacing:".12em",marginBottom:6}},"勤怠打刻"),
      e("div",{className:"mono",style:{fontSize:48,fontWeight:500,letterSpacing:".04em",lineHeight:1}},FT(now)),
      e("p",{style:{fontSize:13,color:"#888",marginTop:6}},new Date(now).toLocaleDateString("ja-JP",{year:"numeric",month:"long",day:"numeric",weekday:"short"})),
      e("p",{style:{fontSize:11,color:"#888",marginTop:2}},
        e("span",{className:"mono"},FT(isClockedIn?R15down(now):R15up(now))),isClockedIn?" に退勤打刻（切り捨て）":" に出勤打刻（切り上げ）"),
      gasUrl&&e("p",{style:{fontSize:10,display:"flex",alignItems:"center",justifyContent:"center",gap:4,marginTop:4,color:connected?"#0F6E56":"#888"}},
        e("span",{className:"dot",style:{background:syncing?"#EF9F27":connected?"#1D9E75":"#888"}}),
        syncing?"シートに同期中...":connected?"スプレッドシート連携中":"未接続")
    ),
    wps.length===0
      ?e("p",{style:{textAlign:"center",fontSize:13,color:"#888",padding:"20px 0"}},"管理者が現場を登録するまでお待ちください")
      :e("div",null,
        e("div",{style:{marginBottom:12}},
          e("label",{style:{fontSize:11,color:"#888",display:"block",marginBottom:5}},"現場"),
          e("select",{value:selWp,onChange:ev=>setSelWp(ev.target.value),style:{width:"100%",fontSize:14,padding:"10px 12px"}},
            wps.map(w=>e("option",{key:w.id,value:w.id},w.name)))
        ),
        e("div",{style:{marginBottom:20}},
          e("label",{style:{fontSize:11,color:"#888",display:"block",marginBottom:5}},"社員"),
          e("select",{value:selEmp,onChange:ev=>{setSelEmp(ev.target.value);setMsg({t:"",c:""})},style:{width:"100%",fontSize:14,padding:"10px 12px"}},
            e("option",{value:""},"-- 選択してください --"),
            emps.map(em=>e("option",{key:em.id,value:em.id},em.name+" （"+em.dept+"）"))
          )
        ),
        selEmp&&e("div",{style:{background:isClockedIn?"#E1F5EE":isDone?"#f5f5f5":"#E6F1FB",borderRadius:"8px",padding:"10px 14px",marginBottom:16,fontSize:13}},
          isClockedIn?e("span",{style:{color:"#0F6E56"}},"● 出勤中 — "+FT(active?.ci)+" から")
          :isDone?e("span",{style:{color:"#888"}},"✓ 退勤済 "+FT(todayRec?.ci)+" → "+FT(todayRec?.co)+" ("+DH(todayRec?.ci,todayRec?.co).toFixed(2)+"h)")
          :e("span",{style:{color:"#185FA5"}},"○ 未出勤")
        ),
        e("button",{className:"clock-btn",onClick:doClock,disabled:!selEmp||isDone,
          style:{background:isClockedIn?"#0F6E56":"#185FA5",marginBottom:12}},
          syncing?"同期中...":(isClockedIn?"退勤する":"出勤する")),
        msg.t&&e("p",{style:{textAlign:"center",fontSize:14,fontWeight:500,color:msg.c,padding:"4px 0"}},msg.t)
      ),
    e("div",{style:{flex:1}}),
    e("p",{style:{textAlign:"center",fontSize:12,marginTop:32}},
      e("span",{onClick:()=>setView("admin"),style:{cursor:"pointer",color:"#185FA5"}},"管理者ログイン →"))
  );
}
ReactDOM.render(e(App),document.getElementById("root"));
</script>
</body>
</html>
