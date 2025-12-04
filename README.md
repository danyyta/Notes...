<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Retro Bowl Clone Animated</title>
<style>
  body { margin:0; font-family: sans-serif; background: #333; color:#fff; }
  #gameContainer { position: relative; width: 100vw; height: 80vh; overflow: hidden; background: green; }
  canvas { display: block; margin: auto; background: green; }
  #ui { text-align: center; margin-top: 10px; }
  button { margin: 2px; padding: 8px 12px; font-size: 16px; }
  #log { width: 100vw; max-height: 20vh; overflow-y: scroll; background: #111; padding:5px; font-size:14px; }
</style>
</head>
<body>

<div id="gameContainer">
  <canvas id="gameCanvas" width="800" height="400"></canvas>
</div>

<div id="ui">
  <button id="runBtn">Run</button>
  <button id="passBtn">Pass</button>
  <button id="puntBtn">Punt</button>
  <button id="fgBtn">Field Goal</button>
  <span id="scoreDisplay">HOME 0 - 0 AWAY</span>
</div>

<div id="log"></div>

<script>
// -------------------- Constants --------------------
const FIELD_LENGTH = 100;
const FIELD_WIDTH = 50;
const PIXELS_PER_YARD = 7;
const PLAYER_SIZE = 12;
const QB_SIZE = 14;
const BALL_SIZE = 8;
const PLAYER_SPEED = 1.5;

const TEAMS = [
  {name:"Bears", abbrev:"CHI", color:"#003366"},
  {name:"Packers", abbrev:"GNB", color:"#203731"},
  {name:"Eagles", abbrev:"PHI", color:"#004C54"},
  {name:"Cowboys", abbrev:"DAL", color:"#003594"},
  {name:"49ers", abbrev:"SFO", color:"#AA0000"},
  {name:"Giants", abbrev:"NYG", color:"#0B2265"},
  {name:"Rams", abbrev:"LAR", color:"#003594"},
  {name:"Seahawks", abbrev:"SEA", color:"#002244"}
  // Add more if desired
];

// -------------------- Variables --------------------
let canvas = document.getElementById("gameCanvas");
let ctx = canvas.getContext("2d");
let logDiv = document.getElementById("log");
let scoreDisplay = document.getElementById("scoreDisplay");

let homeTeam = TEAMS[0];
let awayTeam = TEAMS[1];

let score = {home:0, away:0};
let lineOfScrimmage = 20;
let down = 1;
let yardsToGo = 10;
let ball = {x: lineOfScrimmage * PIXELS_PER_YARD, y: canvas.height/2};
let qb = {x: ball.x, y: ball.y, color: homeTeam.color};
let players = [];
let currentPlay = null;

// -------------------- Setup Players --------------------
function setupPlayers(){
  players = [];
  for(let i=0;i<11;i++){
    // Offense
    players.push({x: ball.x-20, y: 50+i*25, color: homeTeam.color, type:'offense', legPhase:0});
    // Defense
    players.push({x: ball.x+100, y: 50+i*25, color: awayTeam.color, type:'defense', legPhase:0});
  }
}
setupPlayers();

// -------------------- Draw Field --------------------
function drawField(){
  ctx.clearRect(0,0,canvas.width,canvas.height);
  ctx.fillStyle="green";
  ctx.fillRect(0,0,canvas.width,canvas.height);
  ctx.strokeStyle="white";
  ctx.lineWidth=1;
  for(let y=0;y<=FIELD_LENGTH;y+=5){
    let x = y*PIXELS_PER_YARD - ball.x + canvas.width/2;
    if(x>=0 && x<=canvas.width){
      ctx.beginPath();
      ctx.moveTo(x,0);
      ctx.lineTo(x,canvas.height);
      ctx.stroke();
      if(y%10===0){
        ctx.fillStyle="white";
        ctx.font="10px Arial";
        ctx.fillText(y,x-5,10);
      }
    }
  }
  // First down marker
  let fdX = (lineOfScrimmage + yardsToGo)*PIXELS_PER_YARD - ball.x + canvas.width/2;
  ctx.strokeStyle="yellow";
  ctx.beginPath();
  ctx.moveTo(fdX,0);
  ctx.lineTo(fdX,canvas.height);
  ctx.stroke();
}

// -------------------- Draw Players --------------------
function drawPlayers(){
  // QB
  drawHumanoid(qb.x, qb.y, qb.color, qb);
  // Players
  for(let p of players){
    drawHumanoid(p.x, p.y, p.color, p);
  }
  // Ball
  ctx.fillStyle="yellow";
  ctx.beginPath();
  ctx.ellipse(canvas.width/2, ball.y, BALL_SIZE, BALL_SIZE/2, 0,0,2*Math.PI);
  ctx.fill();
}

// -------------------- Draw Humanoid --------------------
function drawHumanoid(x,y,color,player){
  let drawX = x - ball.x + canvas.width/2;
  let drawY = y;
  ctx.fillStyle=color;
  // Body
  ctx.fillRect(drawX-PLAYER_SIZE/2, drawY-PLAYER_SIZE, PLAYER_SIZE, PLAYER_SIZE);
  // Legs animation
  ctx.strokeStyle=color;
  ctx.lineWidth=2;
  ctx.beginPath();
  let legOffset = (player.legPhase%10<5? -3:3);
  ctx.moveTo(drawX-PLAYER_SIZE/4, drawY);
  ctx.lineTo(drawX-PLAYER_SIZE/4, drawY+6+legOffset);
  ctx.moveTo(drawX+PLAYER_SIZE/4, drawY);
  ctx.lineTo(drawX+PLAYER_SIZE/4, drawY+6-legOffset);
  ctx.stroke();
  player.legPhase++;
}

// -------------------- Update Play --------------------
function updatePlay(){
  if(currentPlay==='run'){
    ball.x += PLAYER_SPEED;
    qb.x = ball.x;
    for(let p of players){
      if(p.type==='offense') p.x += PLAYER_SPEED;
      else p.x -= PLAYER_SPEED/1.2;
    }
    checkTackle();
  } else if(currentPlay==='pass'){
    ball.x += PLAYER_SPEED*1.2;
    checkPassCompletion();
  } else if(currentPlay==='punt'){
    ball.x += PLAYER_SPEED*2;
    if(ball.x>FIELD_LENGTH*PIXELS_PER_YARD) switchPossession();
  } else if(currentPlay==='fg'){
    if(ball.x >= lineOfScrimmage*PIXELS_PER_YARD + 30){
      log("Field Goal GOOD!");
      score.home +=3;
    } else log("Field Goal MISS!");
    switchPossession();
  }
}

// -------------------- Check Tackle & Completion --------------------
function checkTackle(){
  for(let p of players){
    if(p.type==='defense' && Math.abs(p.x-ball.x)<10 && Math.abs(p.y-ball.y)<10){
      let yardsGained = Math.round((ball.x/PIXELS_PER_YARD - lineOfScrimmage));
      log(`Tackled! Gained ${yardsGained} yards`);
      down++;
      yardsToGo -= yardsGained;
      lineOfScrimmage = Math.round(ball.x/PIXELS_PER_YARD);
      if(down>4){ down=1; yardsToGo=10; switchPossession(); }
      currentPlay=null;
    }
  }
}

function checkPassCompletion(){
  if(ball.x >= FIELD_LENGTH*PIXELS_PER_YARD){
    log("Touchdown!");
    score.home +=7;
    switchPossession();
    currentPlay=null;
  }
}

// -------------------- Switch Possession --------------------
function switchPossession(){
  [homeTeam,awayTeam] = [awayTeam,homeTeam];
  scoreDisplay.textContent=`${homeTeam.abbrev} ${score.home} - ${score.away} ${awayTeam.abbrev}`;
  ball.x = lineOfScrimmage*PIXELS_PER_YARD;
  qb.x = ball.x;
  setupPlayers();
  down=1;
  yardsToGo=10;
  currentPlay=null;
  log("Possession switched");
}

// -------------------- Logging --------------------
function log(msg){
  let p = document.createElement("div");
  p.textContent = msg;
  logDiv.appendChild(p);
  logDiv.scrollTop = logDiv.scrollHeight;
}

// -------------------- UI --------------------
document.getElementById("runBtn").addEventListener("click",()=>{currentPlay='run'; log("Run play")});
document.getElementById("passBtn").addEventListener("click",()=>{currentPlay='pass'; log("Pass play")});
document.getElementById("puntBtn").addEventListener("click",()=>{currentPlay='punt'; log("Punt play")});
document.getElementById("fgBtn").addEventListener("click",()=>{currentPlay='fg'; log("Field Goal attempt")});

// -------------------- Game Loop --------------------
function gameLoop(){
  drawField();
  drawPlayers();
  updatePlay();
  requestAnimationFrame(gameLoop);
}
gameLoop();
</script>

</body>
</html>
