
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>Storyboard Tool</title>

<script src="https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>

<style>
body { font-family: sans-serif; padding: 10px; }

/* 上部 */
#palette {
  position: sticky;
  top: 0;
  background: white;
  z-index: 10;
  padding: 10px;
  border-bottom: 1px solid #ccc;
}



.piece {
  display: inline-block;
  padding: 6px 10px;
  border: 2px dashed #aaa;
  cursor: grab;
  margin-right: 10px;
}

/* グリッド（左追加） */
#container {
  display: grid;
  grid-auto-flow: column;
  grid-template-rows: repeat(4, auto);
  grid-auto-columns: 360px;
  column-gap: 40px;
  row-gap: 10px;
  align-content: start;
}

.panel, .textblock {
  direction: ltr;
  width: 360px;
  border: 2px solid #000;
  padding: 5px;
  background: #fff;
  position: relative;
}

.checkbox {
  position: absolute;
  top: 5px;
  right: 5px;
}

.tools {
  display: flex;
  gap: 5px;
  margin-bottom: 5px;
  flex-wrap: wrap;
}

canvas { border: 1px solid #ccc; width: 100%; }
textarea { width: 100%; height: 50px; }

.textblock { padding-top: 30px; }
.textblock input {
  width: 100%;
  text-align: center;
  font-size: 18px;
}

.num {
  position: absolute;
  bottom: 5px;
  left: 5px;
  font-size: 12px;
}

.drop-indicator {
  width: 360px;
  height: 6px;
  background: blue;
}

.export-mode .tools,
.export-mode .checkbox {
  display: none;
}

#addBottom {
  width: 360px;
  height: 40px;
  font-size: 16px;
  justify-self: start;
}

.active {
  outline: 2px solid blue;
}

.page-break {
  grid-column: span 1;
  height: 10px;
  border-top: 3px dashed #999;
  margin: 10px 0;
}

@media print {
  body { margin: 0; }
  #palette { display: none; }
  .panel, .textblock {
    page-break-inside: avoid;
  }
}

/* UI improvements */
#palette { display:flex; flex-wrap:wrap; gap:8px; align-items:center; }
#palette .group { display:flex; gap:6px; align-items:center; margin-right:12px; }
button { padding:6px 10px; cursor:pointer; }

/* modal */
#modal {
  position:fixed; inset:0; background:rgba(0,0,0,0.4);
  display:none; align-items:center; justify-content:center; z-index:100;
}
#modalContent{
  background:#fff; padding:20px; width:320px; max-height:70vh; overflow:auto;
  border-radius:8px;
}
.fileItem{
  border:1px solid #ccc; padding:8px; margin-bottom:6px;
  display:flex; justify-content:space-between; align-items:center;
}
.fileItem span{ cursor:pointer; }
#modalContent h3 { display:flex; justify-content:space-between; align-items:center; }
.closeBtn { cursor:pointer; font-size:18px; }

#statusBar { margin-left:auto; gap:10px; }
#saveStatus { font-weight:bold; }
</style>
</head>

<body>

<div id="palette">
  <div class="group">
    <div class="piece" draggable="true" data-type="panel">＋コマ</div>
    <div class="piece" draggable="true" data-type="text">見出し</div>
    <button onclick="addPanel()">＋コマ追加</button>
  </div>

  <div class="group">
    <button onclick="toggleDirection()">←→ 切替</button>
    <span id="bulk" style="display:none;">
      <button onclick="selectAll()">全て選択</button>
      <button onclick="deleteSelected()">削除</button>
      <button onclick="duplicateSelected()">複製</button>
      <button onclick="clearSelection()">解除</button>
    </span>
  </div>

  <div class="group">
    <button onclick="exportPNG()">PNG</button>
    <button onclick="exportPDF()">PDF</button>
  </div>

  <div class="group">
    <button onclick="saveAsNew()">新規保存</button>
    <button onclick="openModal()">開く</button>
  </div>
</div>

<div class="group" id="statusBar">
  <span id="currentName">(未選択)</span>
  <span id="saveStatus">保存済み</span>
  <button onclick="manualSave()">保存</button>
</div>

<div id="container"></div>

<script>
  function setStatus(text){
  const el = document.getElementById("saveStatus");
  if(el) el.textContent = text;
}

function updateCurrentName(){
  const el = document.getElementById("currentName");
  if(el){
    el.textContent = currentKey.replace("story_","");
  }
}

function manualSave(){
  saveAll();
  setStatus("保存済み");
}
const container = document.getElementById("container");

let selected = new Set();
let dragSrc = null;
let dragType = null;
let flowDirection = "right";

let currentKey = "story_default";

const indicator = document.createElement("div");
indicator.className="drop-indicator";

/* ===== 自動スクロール ===== */
document.addEventListener("dragover",(e)=>{
  if(e.clientY<80) window.scrollBy(0,-10);
  if(window.innerHeight-e.clientY<80) window.scrollBy(0,10);
  indicator.style.pointerEvents = "none";
});

/* ===== 描画 ===== */
function createCanvas(canvas){
  const ctx=canvas.getContext("2d");

  let drawing=false;
  let tool="pen";
  let color="black";
  let currentButtons = {};

  let history=[];
  let step=-1;

  function save(){
    step++;
    history = history.slice(0, step);
    history.push({img:canvas.toDataURL()});
    saveAll();
  }

  return {
    setTool:(t,btn)=>{
      tool=t;
      if(currentButtons.tool) currentButtons.tool.classList.remove("active");
      currentButtons.tool = btn;
      btn.classList.add("active");
    },
    setColor:(c,btn)=>{
      color=c;
      if(currentButtons.color) currentButtons.color.classList.remove("active");
      currentButtons.color = btn;
      btn.classList.add("active");
    },

    undo:()=>{
      if(step<=0)return;
      step--;
      const img=new Image();
      img.src=history[step].img;
      img.onload=()=>{
        ctx.clearRect(0,0,canvas.width,canvas.height);
        ctx.drawImage(img,0,0);
      };
    },

    clear:()=>{
      ctx.clearRect(0,0,canvas.width,canvas.height);
      save();
    },

    start:(x,y)=>{
      drawing=true;
      ctx.beginPath();
      ctx.moveTo(x,y);
    },

    draw:(x,y)=>{
      if(!drawing)return;

      ctx.strokeStyle=color;
      ctx.lineWidth=tool==="pen"?1:10;
      ctx.globalCompositeOperation=
        tool==="pen"?"source-over":"destination-out";

      ctx.lineTo(x,y);
      ctx.stroke();
    },

    end:()=>{
      if(!drawing)return;
      drawing=false;
      ctx.globalCompositeOperation="source-over";
      save();
    }
  };
}

/* ===== コマ ===== */
function createPanel(){
  const div=document.createElement("div");
  div.className="panel";

  const num=document.createElement("div");
  num.className="num";

  const cb=document.createElement("input");
  cb.type="checkbox";
  cb.className="checkbox";

  cb.onchange=()=>{
    cb.checked?selected.add(div):selected.delete(div);
    div.draggable=cb.checked;
    updateBulk();
  };

  const tools=document.createElement("div");
  tools.className="tools";

  const canvas=document.createElement("canvas");
  canvas.width=360; canvas.height=240;

  const draw=createCanvas(canvas);

  const buttons=[
    ["✏️","tool","pen"],
    ["🧽","tool","eraser"],
    ["↩","undo"],
    ["🧼","clear"],
    ["🔴","color","red"],
    ["🔵","color","blue"],
    ["⚫","color","black"]
  ];

  buttons.forEach(([label,type,value])=>{
    const b=document.createElement("button");
    b.textContent=label;

    if(type==="tool"){
      b.onclick=()=>draw.setTool(value,b);
      if(value==="pen") b.classList.add("active");
    }
    else if(type==="color"){
      b.onclick=()=>draw.setColor(value,b);
      if(value==="black") b.classList.add("active");
    }
    else if(label==="↩") b.onclick=draw.undo;
    else if(label==="🧼") b.onclick=draw.clear;

    tools.appendChild(b);
  });

  canvas.style.touchAction = "none"; // iPad必須

canvas.onpointerdown = (e) => {
  e.preventDefault();
  canvas.setPointerCapture(e.pointerId);
  draw.start(e.offsetX, e.offsetY);
};

canvas.onpointermove = (e) => {
  if (e.pressure > 0 || e.buttons === 1) {
    draw.draw(e.offsetX, e.offsetY);
  }
};

canvas.onpointerup = (e) => {
  draw.end();
};

canvas.onpointercancel = () => {
  draw.end();
};

  const text=document.createElement("textarea");
 
  text.oninput = ()=>{
  setStatus("未保存");
  saveAll();
};

  div.append(num,cb,tools,canvas,text);

  div.addEventListener("dragstart",()=>{
    dragSrc = selected.has(div)?[...selected]:div;
  });

  return div;
}

/* ===== 見出し ===== */
function createText(){
  const div=document.createElement("div");
  div.className="textblock";

  const cb=document.createElement("input");
  cb.type="checkbox";
  cb.className="checkbox";

  const input=document.createElement("input");
  input.placeholder="見出し";

  input.oninput=saveAll;

  cb.onchange=()=>{
    cb.checked?selected.add(div):selected.delete(div);
    div.draggable=cb.checked;
    updateBulk();
  };

  div.append(cb,input);

  div.addEventListener("dragstart",()=>{
    dragSrc = selected.has(div)?[...selected]:div;
  });

  return div;
}

/* ===== ピース ===== */
document.querySelectorAll(".piece").forEach(p=>{
  p.addEventListener("dragstart",()=>{
    dragSrc="new";
    dragType=p.dataset.type;
  });
});

/* ===== UI ===== */
function updateBulk(){
  document.getElementById("bulk").style.display =
    selected.size?"inline":"none";
}

function clearSelection(){
  selected.forEach(e=>{
    e.querySelector(".checkbox").checked=false;
    e.draggable=false;
  });
  selected.clear();
  updateBulk();
}

function selectAll(){
  selected.clear();
  [...container.children].forEach(el=>{
    if(el.id !== "addBottom"){
      const cb = el.querySelector(".checkbox");
      if(cb){
        cb.checked = true;
        el.draggable = true;
        selected.add(el);
      }
    }
  });
  updateBulk();
}

/* ===== 番号 ===== */
function updateNumbers(){
  [...container.children].forEach((e,i)=>{
    if(e.classList.contains("panel")){
      e.querySelector(".num").textContent=i+1;
    }
  });
}

/* ===== 保存 ===== */
function saveAll(){
  const data=[...container.children].map(el=>{
    if(el.classList.contains("panel")){
      return {
        t:"p",
        i:el.querySelector("canvas").toDataURL(),
        txt:el.querySelector("textarea").value
      };
    }else if(el.classList.contains("textblock")){
      return {t:"t",txt:el.querySelector("input").value};
    }
  });
  localStorage.setItem(currentKey,JSON.stringify(data));
}

/* ===== 読み込み ===== */
function loadAll(){
  container.innerHTML = "";

  const d = JSON.parse(localStorage.getItem(currentKey)||"[]");

  d.forEach(x=>{
    if(x.t==="p"){
      const p=createPanel();
      const img=new Image();
      img.src=x.i;
      img.onload=()=>p.querySelector("canvas").getContext("2d").drawImage(img,0,0);
      p.querySelector("textarea").value=x.txt;
      container.appendChild(p);
    }else{
      const t=createText();
      t.querySelector("input").value=x.txt;
      container.appendChild(t);
    }
  });

  update();
  updateAddButton();
}

/* ===== 追加 ===== */
function addPanel(){
  const btn = document.getElementById("addBottom");
  const el = createPanel();
  if(btn){
    container.insertBefore(el, btn);
  }else{
    container.appendChild(el);
  }
  updateNumbers();
  saveAll();
  updateAddButton();
}

function addText(){
  const btn = document.getElementById("addBottom");
  const el = createText();
  if(btn) container.insertBefore(el, btn);
  else container.appendChild(el);
  update();
}

/* ===== ドラッグ ===== */
function updateAddButton(){
  let btn = document.getElementById("addBottom");

  if(!btn){
    btn = document.createElement("button");
    btn.id = "addBottom";
    btn.textContent = "＋コマ追加";
    btn.onclick = addPanel;
  }

  container.appendChild(btn);
}

function getDrop(e){
  const els = [...container.children].filter(el=>el.id!=="addBottom" && el!==indicator);

  let closest = null;
  let minDist = Infinity;

  for(let el of els){
    const r = el.getBoundingClientRect();

    // center point of each panel
    const cx = r.left + r.width / 2;
    const cy = r.top + r.height / 2;

    const dx = e.clientX - cx;
    const dy = e.clientY - cy;
    const dist = Math.sqrt(dx*dx + dy*dy);

    if(dist < minDist){
      minDist = dist;
      closest = el;
    }
  }

  return closest;
}

container.addEventListener("dragover",e=>{
  e.preventDefault();
  const pos = getDrop(e);
  const btn = document.getElementById("addBottom");

  if(pos){
    const r = pos.getBoundingClientRect();
    if(e.clientY < r.top + r.height/2){
      container.insertBefore(indicator, pos);
    }else{
      container.insertBefore(indicator, pos.nextSibling);
    }
  }else if(btn){
    container.insertBefore(indicator, btn);
  }else{
    container.appendChild(indicator);
  }
});

container.addEventListener("drop",e=>{
  e.preventDefault();

  if(dragSrc==="new"){
    const el = dragType==="panel"?createPanel():createText();
    container.insertBefore(el,indicator);
  }else if(Array.isArray(dragSrc)){
    dragSrc.forEach(el=>container.insertBefore(el,indicator));
  }else{
    container.insertBefore(dragSrc,indicator);
  }

  indicator.remove();
  dragSrc = null;
  updateAddButton();
});

/* ===== 複数 ===== */
function deleteSelected(){
  selected.forEach(e=>e.remove());
  selected.clear();
  update();
}

function duplicateSelected(){
  const items = [...selected];

  items.forEach(el=>{
    let clone;

    if(el.classList.contains("panel")){
      clone = createPanel();

      // canvasコピー
      const srcCanvas = el.querySelector("canvas");
      const dstCanvas = clone.querySelector("canvas");
      const ctx = dstCanvas.getContext("2d");
      ctx.drawImage(srcCanvas, 0, 0);

      // テキストコピー
      clone.querySelector("textarea").value = el.querySelector("textarea").value;
    }else{
      clone = createText();
      clone.querySelector("input").value = el.querySelector("input").value;
    }

    // 元の直後に挿入
    container.insertBefore(clone, el.nextSibling);
  });

  update();
}

/* ===== 新規保存・開く ===== */
function saveAsNew(){
  const name = prompt("保存名を入力");
  if(!name) return;

  const key = "story_" + name;
  currentKey = key;
  saveAll();

  let list = JSON.parse(localStorage.getItem("story_list")||"[]");
  if(!list.includes(key)){
    list.push(key);
    localStorage.setItem("story_list", JSON.stringify(list));
  }
}

function openModal(){
  const modal = document.getElementById("modal");
  const content = document.getElementById("modalContent");
  content.innerHTML = "<h3>データ一覧 <span class='closeBtn' onclick='closeModal()'>✕</span></h3>";

  const list = JSON.parse(localStorage.getItem("story_list")||"[]");

  list.forEach(key=>{
    const div = document.createElement("div");
    div.className = "fileItem";

    const data = JSON.parse(localStorage.getItem(key)||"[]");

    const left = document.createElement("div");
    left.style.display = "flex";
    left.style.alignItems = "center";
    left.style.gap = "8px";

    // サムネイル
    const thumb = document.createElement("canvas");
    thumb.width = 60;
    thumb.height = 40;
    const tctx = thumb.getContext("2d");

    const firstPanel = data.find(d=>d.t==="p");
    if(firstPanel){
      const img = new Image();
      img.src = firstPanel.i;
      img.onload = ()=>{
        tctx.drawImage(img,0,0,60,40);
      };
    }

    const name = document.createElement("span");
    name.textContent = key.replace("story_","");
    name.onclick = ()=>{
      currentKey = key;
      loadAll();
      closeModal();
    };

    left.append(thumb,name);

    const actions = document.createElement("div");

    const rename = document.createElement("button");
    rename.textContent = "名前変更";
    rename.onclick = ()=>{
      const newName = prompt("新しい名前");
      if(!newName) return;

      const newKey = "story_" + newName;
      localStorage.setItem(newKey, localStorage.getItem(key));
      localStorage.removeItem(key);

      let list2 = JSON.parse(localStorage.getItem("story_list")||"[]");
      list2 = list2.map(k => k===key ? newKey : k);
      localStorage.setItem("story_list", JSON.stringify(list2));

      openModal();
    };

    const del = document.createElement("button");
    del.textContent = "削除";
    del.onclick = ()=>{
      localStorage.removeItem(key);
      let list2 = JSON.parse(localStorage.getItem("story_list")||"[]");
      list2 = list2.filter(k=>k!==key);
      localStorage.setItem("story_list",JSON.stringify(list2));
      openModal();
    };

    const dup = document.createElement("button");
    dup.textContent = "複製";
    dup.onclick = ()=>{
      const newKey = key + "_copy";
      localStorage.setItem(newKey, localStorage.getItem(key));
      let list2 = JSON.parse(localStorage.getItem("story_list")||"[]");
      list2.push(newKey);
      localStorage.setItem("story_list",JSON.stringify(list2));
      openModal();
    };

    actions.append(rename,del,dup);
    div.append(left,actions);
    content.appendChild(div);
  });

  modal.style.display = "flex";
}

function closeModal(){
  document.getElementById("modal").style.display = "none";
}

document.getElementById("modal").onclick = (e)=>{
  if(e.target.id === "modal") closeModal();
};

/* ===== 出力 ===== */
function exportPNG(){
  document.body.classList.add("export-mode");
  html2canvas(container).then(c=>{
    const a=document.createElement("a");
    a.href=c.toDataURL();
    a.download="story.png";
    a.click();
    document.body.classList.remove("export-mode");
  });
}

function exportPDF(){
  document.body.classList.add("export-mode");
  html2canvas(container).then(c=>{
    const pdf=new jspdf.jsPDF();
    pdf.addImage(c.toDataURL(),"PNG",0,0,210,c.height*210/c.width);
    pdf.save("story.pdf");
    document.body.classList.remove("export-mode");
  });
}

/* ===== 更新 ===== */
function update(){
  // remove old page breaks
  document.querySelectorAll(".page-break").forEach(e=>e.remove());

  const items = [...container.children].filter(el=>el.id!=="addBottom");

  items.forEach((el,i)=>{
    if((i+1)%20===0){
      const br = document.createElement("div");
      br.className = "page-break";
      br.textContent = "--- Page ---";
      br.style.textAlign = "center";
      container.insertBefore(br, el.nextSibling);
    }
  });

  updateNumbers();
  saveAll();
  updateAddButton();
}

/* ===== 方向切替 ===== */
function toggleDirection(){
  flowDirection = flowDirection === "right" ? "left" : "right";

  if(flowDirection === "left"){
    container.style.direction = "rtl";
  }else{
    container.style.direction = "ltr";
  }
}

/* ===== 初期 ===== */
loadAll();
</script>

<div id="modal">
  <div id="modalContent"></div>
</div>

</body>
</html>
