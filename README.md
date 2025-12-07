# Tonk-game
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<title>Tonk – play offline</title>
<style>
  body{font-family:Arial,Helvetica,sans-serif;background:#0b4d0b;color:#fff;margin:0;display:flex;flex-direction:column;align-items:center}
  h1{margin:.3em 0}
  #table{width:720px;height:540px;background:#0f7b0f;border-radius:12px;display:flex;flex-direction:column;justify-content:space-between;padding:12px;box-shadow:0 0 18px #000}
  .hand{display:flex;justify-content:center;min-height:96px}
  #middle{display:flex;justify-content:center;gap:40px}
  .pile{width:80px;height:112px;border:2px dashed #fff;border-radius:8px;display:flex;align-items:center;justify-content:center;cursor:pointer;position:relative}
  .pile span{pointer-events:none}
  .card{width:64px;height:89px;margin:0 -20px;cursor:pointer;border-radius:6px;box-shadow:2px 2px 4px rgba(0,0,0,.5);transition:transform .15s}
  .card:hover{transform:translateY(-12px)}
  .selected{border:3px solid #ffeb3b !important}
  #controls{margin-top:12px}
  button{padding:6px 12px;margin-right:8px}
  #msg{margin-left:12px;font-weight:700}
</style>
</head>
<body>
<h1>Tonk – 2-Player Demo</h1>

<div id="table">
  <div id="oppHand" class="hand"></div>
  <div id="middle">
    <div id="stock" class="pile"><span>Stock</span></div>
    <div id="discard" class="pile"><span>Discard</span></div>
  </div>
  <div id="myHand" class="hand"></div>
</div>

<div id="controls">
  <button id="knockBtn">Knock</button>
  <button id="newBtn">New Game</button>
  <span id="msg"></span>
</div>

<script>
/* ---------- card renderer (SVG) ---------- */
function makeCard(code){
  if(code==="back"){
    return `<svg class="card" viewBox="0 0 64 89">
      <rect width="64" height="89" rx="4" fill="#036"/>
      <rect x="4" y="4" width="56" height="81" rx="3" fill="#069"/>
      <text x="32" y="50" text-anchor="middle" font-size="18" fill="#fff">🂠</text>
    </svg>`;
  }
  const rank=code.slice(0,-1), suit=code.slice(-1);
  const color=(suit==="h"||suit==="d")?"red":"black";
  const sym={h:"♥",d:"♦",c:"♣",s:"♠"}[suit];
  return `<svg class="card" viewBox="0 0 64 89">
    <rect width="64" height="89" rx="4" fill="white"/>
    <text x="6" y="14" font-size="12" fill="${color}">${rank}</text>
    <text x="6" y="26" font-size="12" fill="${color}">${sym}</text>
    <text x="32" y="55" font-size="24" text-anchor="middle" fill="${color}">${sym}</text>
  </svg>`;
}

/* ---------- helpers ---------- */
const RANKS=["A","2","3","4","5","6","7","8","9","10","J","Q","K"];
function newDeck(){
  const d=[];
  for(const s of ["s","h","d","c"])
    for(const r of RANKS) d.push(r+s);
  return d.sort(()=>Math.random()-.5);
}
function value(card){
  const r=card.slice(0,-1);
  return r==="A"?1: (r==="J"||r==="Q"||r==="K"?10: Number(r));
}
function sum(hand){ return hand.reduce((a,c)=>a+value(c),0); }
function canKnock(hand){ return sum(hand)<=15; }

/* ---------- dom ---------- */
const myHandEl=document.getElementById("myHand");
const oppHandEl=document.getElementById("oppHand");
const stockEl=document.getElementById("stock");
const discardEl=document.getElementById("discard");
const knockBtn=document.getElementById("knockBtn");
const msgBox=document.getElementById("msg");
function render(){
  myHandEl.innerHTML=state.me.map((c,i)=>makeCard(c)).join("");
  oppHandEl.innerHTML=state.opp.map(()=>makeCard("back")).join("");
  stockEl.innerHTML=state.stock.length?makeCard("back"):"";
  discardEl.innerHTML=state.discard.length?makeCard(state.discard[0]):"";
  knockBtn.disabled=!canKnock(state.me);
}
function say(s){ msgBox.textContent=s; }

/* ---------- game state ---------- */
let state=null;
function newGame(){
  const deck=newDeck();
  state={
    me: deck.splice(0,5),
    opp: deck.splice(0,5),
    stock: deck,
    discard:[],
    turn:0,
    selected:null
  };
  state.discard.push(state.stock.pop());
  render();
  say("Your turn – draw from stock or discard.");
}
newGame();

/* ---------- events ---------- */
stockEl.onclick=()=>draw("stock");
discardEl.onclick=()=>draw("discard");
knockBtn.onclick=()=>knock();

function draw(pile){
  if(state.turn!==0) return;
  const src=pile==="stock"?state.stock:state.discard;
  if(!src.length) return;
  const card=src.pop();
  if(pile==="discard") state.discard=[];
  state.me.push(card);
  state.turn=1;
  render();
  say("Select a card to discard.");
}
myHandEl.addEventListener("click",e=>{
  const cardNode=e.target.closest(".card");
  if(!cardNode||state.turn!==1) return;
  const idx=[...myHandEl.children].indexOf(cardNode);
  if(idx<0||idx>=state.me.length) return;
  const dropped=state.me.splice(idx,1)[0];
  state.discard.unshift(dropped);
  state.turn=0;
  render();
  if(canKnock(state.me)) say("You may knock or continue.");
  else say("Opponent turn…");
  setTimeout(opponentTurn,600);
});

function knock(){
  if(!canKnock(state.me)) return;
  endGame("You knocked!");
}
function endGame(reason){
  const mySum=sum(state.me), oppSum=sum(state.opp);
  let txt=reason+"  You: "+mySum+"  Opponent: "+oppSum+" – ";
  if(mySum<oppSum) txt+="You win!";
  else if(mySum>oppSum) txt+="Opponent wins.";
  else txt="Draw.";
  say(txt); state.turn=-1;
}

/* ---------- simple AI ---------- */
function opponentTurn(){
  if(state.turn!==1) return;
  const takeFrom=value(state.discard[0])>7?"stock":"discard";
  const card=(takeFrom==="stock"?state.stock:state.discard).pop();
  if(takeFrom==="discard") state.discard=[];
  state.opp.push(card);
  state.opp.sort((a,b)=>value(b)-value(a));
  const drop=state.opp.shift();
  state.discard.unshift(drop);
  state.turn=0;
  render();
  if(canKnock(state.opp)){ endGame("Opponent knocked."); return; }
  say("Your turn.");
}

document.getElementById("newBtn").onclick=newGame;
</script>
</body>
</html>
