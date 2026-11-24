# Stripon-app
My first application
-- STRIPON DATABASE FOUNDATION

create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  username text unique not null,
  bio text default '',
  avatar_url text,
  created_at timestamptz default now()
);

create table public.follows (
  follower_id uuid references auth.users(id) on delete cascade,
  following_id uuid references auth.users(id) on delete cascade,
  created_at timestamptz default now(),
  primary key (follower_id, following_id),
  check (follower_id <> following_id)
);

create table public.videos (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  title text not null,
  description text default '',
  video_url text,
  category text default 'all',
  published boolean default false,
  created_at timestamptz default now()
);

-- Enable security
alter table public.profiles enable row level security;
alter table public.follows enable row level security;
alter table public.videos enable row level security;

-- Profile permissions
grant select on public.profiles to anon, authenticated;
grant insert, update on public.profiles to authenticated;

create policy "Public can view profiles"
on public.profiles for select
to anon, authenticated
using (true);

create policy "Users create own profile"
on public.profiles for insert
to authenticated
with check ((select auth.uid()) = id);

create policy "Users update own profile"
on public.profiles for update
to authenticated
using ((select auth.uid()) = id)
with check ((select auth.uid()) = id);

-- Follow permissions
grant select on public.follows to anon, authenticated;
grant insert, delete on public.follows to authenticated;

create policy "Public can view follows"
on public.follows for select
to anon, authenticated
using (true);

create policy "Users follow as themselves"
on public.follows for insert
to authenticated
with check ((select auth.uid()) = follower_id);

create policy "Users unfollow themselves"
on public.follows for delete
to authenticated
using ((select auth.uid()) = follower_id);

-- Video permissions
grant select on public.videos to anon, authenticated;
grant insert, update, delete on public.videos to authenticated;

create policy "Public can view published videos"
on public.videos for select
to anon, authenticated
using (published = true or (select auth.uid()) = user_id);

create policy "Users upload their own video records"
on public.videos for insert
to authenticated
with check ((select auth.uid()) = user_id);

create policy "Users update their own videos"
on public.videos for update
to authenticated
using ((select auth.uid()) = user_id)
with check ((select auth.uid()) = user_id);

create policy "Users delete their own videos"
on public.videos for delete
to authenticated
using ((select auth.uid()) = user_id);
const striponGames = [
  {id:1,name:"Tic Tac Toe",icon:"❌",category:"puzzle",description:"Challenge your strategy."},
  {id:2,name:"Memory Match",icon:"🧠",category:"puzzle",description:"Test your memory."},
  {id:3,name:"Number Challenge",icon:"🔢",category:"arcade",description:"Guess the secret number."},
  {id:4,name:"Rock Paper Scissors",icon:"✊",category:"action",description:"Challenge the computer."},
  {id:5,name:"Sports Quiz",icon:"⚽",category:"sports",description:"Test your sports knowledge."},
  {id:6,name:"Reaction Test",icon:"⚡",category:"arcade",description:"Test your reaction speed."}
];

let gamePoints = Number(localStorage.getItem("striponPoints")||0);
let gamesPlayed = Number(localStorage.getItem("striponPlayed")||0);
let activeGame = "";
let dailyClaimed = localStorage.getItem("striponDaily")==="yes";

function filterGames(category) {
  const games=category==="all"?striponGames:
    striponGames.filter(g=>g.category===category);

  document.getElementById("gameGrid").innerHTML=games.map(g=>`
    <div class="game-tile">
      <div class="game-art">${g.icon}</div>
      <h3>${g.name}</h3>
      <p>${g.description}</p>
      <button onclick="launchGame('${g.id}')">Play now</button>
    </div>
  `).join("");
}

function updateGamingStats() {
  document.getElementById("gamesPlayed").textContent=gamesPlayed;
  document.getElementById("playerScore").textContent=gamePoints;
  document.getElementById("playerRank").textContent=
    gamePoints>=500?"Champion":
    gamePoints>=200?"Pro":
    gamePoints>=50?"Rising Star":"Rookie";
}

function rewardGame() {
  gamePoints+=10;
  gamesPlayed++;
  localStorage.setItem("striponPoints",gamePoints);
  localStorage.setItem("striponPlayed",gamesPlayed);
  updateGamingStats();
}

function launchGame(id) {
  activeGame=id;
  const game=striponGames.find(g=>g.id===Number(id));
  document.getElementById("activeGameTitle").textContent=game.name;
  document.getElementById("gameModal").style.display="block";

  if(id==="1") ticTacToe();
  if(id==="2") memoryGame();
  if(id==="3") numberGame();
  if(id==="4") rockPaperScissors();
  if(id==="5") sportsQuiz();
  if(id==="6") reactionGame();
}

function closeGame() {
  document.getElementById("gameModal").style.display="none";
}

function ticTacToe() {
  let board=["","","","","","","","",""];
  let turn="X";
  const area=document.getElementById("gameArea");

  function draw() {
    area.innerHTML=`<p>Turn: ${turn}</p>
    <div style="display:grid;grid-template-columns:repeat(3,1fr);max-width:300px;margin:auto">
    ${board.map((v,i)=>`<button class="game-button" onclick="ttMove(${i})">${v||" "}</button>`).join("")}
    </div><button onclick="launchGame('1')">Restart</button>`;
  }

  window.ttMove=function(i) {
    if(board[i]||winner()) return;
    board[i]=turn;
    if(winner()) {
      draw();
      area.innerHTML+="<h3>"+turn+" wins!</h3>";
      rewardGame();
      return;
    }
    if(board.every(Boolean)) {
      draw();
      area.innerHTML+="<h3>It's a draw!</h3>";
      return;
    }
    turn=turn==="X"?"O":"X";
    draw();
  };

  function winner() {
    return [[0,1,2],[3,4,5],[6,7,8],
      [0,3,6],[1,4,7],[2,5,8],
      [0,4,8],[2,4,6]].some(line=>
      line.every(i=>board[i]&&board[i]===board[line[0]])
    );
  }
  draw();
}

function numberGame() {
  const answer=Math.floor(Math.random()*10)+1;
  document.getElementById("gameArea").innerHTML=`
    <h3>Guess a number between 1 and 10</h3>
    <input type="number" id="guess" min="1" max="10">
    <button onclick="checkNumber(${answer})">Guess</button>
    <p id="guessResult"></p>`;
}

window.checkNumber=function(answer) {
  const guess=Number(document.getElementById("guess").value);
  const result=document.getElementById("guessResult");
  if(guess===answer) {
    result.textContent="Correct! You earned 10 points.";
    rewardGame();
  } else {
    result.textContent=guess<answer?"Try a higher number!":"Try a lower number!";
  }
};

function rockPaperScissors() {
  document.getElementById("gameArea").innerHTML=`
    <h3>Choose your move</h3>
    <button class="game-button" onclick="playRPS('Rock')">✊</button>
    <button class="game-button" onclick="playRPS('Paper')">✋</button>
    <button class="game-button" onclick="playRPS('Scissors')">✌️</button>
    <p id="rpsResult"></p>`;
}

window.playRPS=function(choice) {
  const options=["Rock","Paper","Scissors"];
  const computer=options[Math.floor(Math.random()*3)];
  let result=choice===computer?"Draw":
    ((choice==="Rock"&&computer==="Scissors")||
    (choice==="Paper"&&computer==="Rock")||
    (choice==="Scissors"&&computer==="Paper"))?"You win!":"Computer wins!";
  document.getElementById("rpsResult").textContent=
    `You: ${choice} | Computer: ${computer} | ${result}`;
  if(result==="You win!") rewardGame();
};

function sportsQuiz() {
  document.getElementById("gameArea").innerHTML=`
    <h3>Which sport uses a racket and shuttlecock?</h3>
    <button class="game-button" onclick="quizAnswer(false)">Tennis</button>
    <button class="game-button" onclick="quizAnswer(true)">Badminton</button>
    <button class="game-button" onclick="quizAnswer(false)">Football</button>
    <p id="quizResult"></p>`;
}

window.quizAnswer=function(correct) {
  document.getElementById("quizResult").textContent=
    correct?"Correct! Well done!":"Not quite. Try again.";
  if(correct) rewardGame();
};

function memoryGame() {
  const symbols=["🍎","🚀","⭐","🎮","🍎","🚀","⭐","🎮"];
  symbols.sort(()=>Math.random()-.5);
  let opened=[];
  let matched=[];
  let busy=false;

  const area=document.getElementById("gameArea");
  function draw() {
    area.innerHTML=`<div style="display:grid;grid-template-columns:repeat(4,1fr);gap:8px">
    ${symbols.map((s,i)=>`
      <button class="game-button" onclick="memoryFlip(${i})">
        ${matched.includes(i)||opened.includes(i)?s:"❔"}
      </button>`).join("")}</div>`;
  }

  window.memoryFlip=function(i) {
    if(busy||matched.includes(i)||opened.includes(i))return;
    opened.push(i);
    draw();
    if(opened.length===2) {
      busy=true;
      if(symbols[opened[0]]===symbols[opened[1]]) {
        matched.push(...opened);
        opened=[];
        busy=false;
        if(matched.length===symbols.length) {
          area.innerHTML+="<h3>You won!</h3>";
          rewardGame();
        }
        draw();
      } else {
        setTimeout(()=>{
          opened=[];
          busy=false;
          draw();
        },800);
      }
    }
  };
  draw();
}

function reactionGame() {
  let started=false;
  let startTime=0;
  const area=document.getElementById("gameArea");
  area.innerHTML=`<h3>Reaction challenge</h3>
    <p>Tap the button when it turns green.</p>
    <button id="reactionButton" class="game-button">Start</button>`;
  const btn=document.getElementById("reactionButton");
  let timer;

  btn.onclick=()=>{
    if(!started) {
      started=true;
      btn.textContent="Wait...";
      btn.style.background="#c65c25";
      timer=setTimeout(()=>{
        startTime=Date.now();
        btn.textContent="TAP NOW!";
        btn.style.background="#159447";
      },1500+Math.random()*3000);
    } else if(startTime) {
      const elapsed=Date.now()-startTime;
      btn.textContent=`${elapsed} ms`;
      rewardGame();
      started=false;
      startTime=0;
    } else {
      clearTimeout(timer);
      btn.textContent="Too early! Try again.";
      started=false;
    }
  };
}

function createGameRoom() {
  const name=document.getElementById("roomName").value.trim();
  if(!name) {
    document.getElementById("roomMessage").textContent="Enter a room name.";
    return;
  }
  const code=Math.random().toString(36).substring(2,8).toUpperCase();
  document.getElementById("roomMessage").textContent=
    `Room ${name} created. Demo code: ${code}`;
}

function joinGameRoom() {
  const code=document.getElementById("joinCode").value.trim();
  document.getElementById("roomMessage").textContent=code?
    `Room code ${code} entered. Online room connection needs a server.`:
    "Enter a room code.";
}

function dailyChallenge() {
  if(dailyClaimed) {
    alert("You have already claimed today's demo challenge.");
    return;
  }
  gamePoints+=10;
  dailyClaimed=true;
  localStorage.setItem("striponDaily","yes");
  localStorage.setItem("striponPoints",gamePoints);
  updateGamingStats();
  alert("10 demo points added!");
}

filterGames("all");
updateGamingStats();
const striponGames = [
  {id:1,name:"Tic Tac Toe",icon:"❌",category:"puzzle",description:"Challenge your strategy."},
  {id:2,name:"Memory Match",icon:"🧠",category:"puzzle",description:"Test your memory."},
  {id:3,name:"Number Challenge",icon:"🔢",category:"arcade",description:"Guess the secret number."},
  {id:4,name:"Rock Paper Scissors",icon:"✊",category:"action",description:"Challenge the computer."},
  {id:5,name:"Sports Quiz",icon:"⚽",category:"sports",description:"Test your sports knowledge."},
  {id:6,name:"Reaction Test",icon:"⚡",category:"arcade",description:"Test your reaction speed."}
];

let gamePoints = Number(localStorage.getItem("striponPoints")||0);
let gamesPlayed = Number(localStorage.getItem("striponPlayed")||0);
let activeGame = "";
let dailyClaimed = localStorage.getItem("striponDaily")==="yes";

function filterGames(category) {
  const games=category==="all"?striponGames:
    striponGames.filter(g=>g.category===category);

  document.getElementById("gameGrid").innerHTML=games.map(g=>`
    <div class="game-tile">
      <div class="game-art">${g.icon}</div>
      <h3>${g.name}</h3>
      <p>${g.description}</p>
      <button onclick="launchGame('${g.id}')">Play now</button>
    </div>
  `).join("");
}

function updateGamingStats() {
  document.getElementById("gamesPlayed").textContent=gamesPlayed;
  document.getElementById("playerScore").textContent=gamePoints;
  document.getElementById("playerRank").textContent=
    gamePoints>=500?"Champion":
    gamePoints>=200?"Pro":
    gamePoints>=50?"Rising Star":"Rookie";
}

function rewardGame() {
  gamePoints+=10;
  gamesPlayed++;
  localStorage.setItem("striponPoints",gamePoints);
  localStorage.setItem("striponPlayed",gamesPlayed);
  updateGamingStats();
}

function launchGame(id) {
  activeGame=id;
  const game=striponGames.find(g=>g.id===Number(id));
  document.getElementById("activeGameTitle").textContent=game.name;
  document.getElementById("gameModal").style.display="block";

  if(id==="1") ticTacToe();
  if(id==="2") memoryGame();
  if(id==="3") numberGame();
  if(id==="4") rockPaperScissors();
  if(id==="5") sportsQuiz();
  if(id==="6") reactionGame();
}

function closeGame() {
  document.getElementById("gameModal").style.display="none";
}

function ticTacToe() {
  let board=["","","","","","","","",""];
  let turn="X";
  const area=document.getElementById("gameArea");

  function draw() {
    area.innerHTML=`<p>Turn: ${turn}</p>
    <div style="display:grid;grid-template-columns:repeat(3,1fr);max-width:300px;margin:auto">
    ${board.map((v,i)=>`<button class="game-button" onclick="ttMove(${i})">${v||" "}</button>`).join("")}
    </div><button onclick="launchGame('1')">Restart</button>`;
  }

  window.ttMove=function(i) {
    if(board[i]||winner()) return;
    board[i]=turn;
    if(winner()) {
      draw();
      area.innerHTML+="<h3>"+turn+" wins!</h3>";
      rewardGame();
      return;
    }
    if(board.every(Boolean)) {
      draw();
      area.innerHTML+="<h3>It's a draw!</h3>";
      return;
    }
    turn=turn==="X"?"O":"X";
    draw();
  };

  function winner() {
    return [[0,1,2],[3,4,5],[6,7,8],
      [0,3,6],[1,4,7],[2,5,8],
      [0,4,8],[2,4,6]].some(line=>
      line.every(i=>board[i]&&board[i]===board[line[0]])
    );
  }
  draw();
}

function numberGame() {
  const answer=Math.floor(Math.random()*10)+1;
  document.getElementById("gameArea").innerHTML=`
    <h3>Guess a number between 1 and 10</h3>
    <input type="number" id="guess" min="1" max="10">
    <button onclick="checkNumber(${answer})">Guess</button>
    <p id="guessResult"></p>`;
}

window.checkNumber=function(answer) {
  const guess=Number(document.getElementById("guess").value);
  const result=document.getElementById("guessResult");
  if(guess===answer) {
    result.textContent="Correct! You earned 10 points.";
    rewardGame();
  } else {
    result.textContent=guess<answer?"Try a higher number!":"Try a lower number!";
  }
};

function rockPaperScissors() {
  document.getElementById("gameArea").innerHTML=`
    <h3>Choose your move</h3>
    <button class="game-button" onclick="playRPS('Rock')">✊</button>
    <button class="game-button" onclick="playRPS('Paper')">✋</button>
    <button class="game-button" onclick="playRPS('Scissors')">✌️</button>
    <p id="rpsResult"></p>`;
}

window.playRPS=function(choice) {
  const options=["Rock","Paper","Scissors"];
  const computer=options[Math.floor(Math.random()*3)];
  let result=choice===computer?"Draw":
    ((choice==="Rock"&&computer==="Scissors")||
    (choice==="Paper"&&computer==="Rock")||
    (choice==="Scissors"&&computer==="Paper"))?"You win!":"Computer wins!";
  document.getElementById("rpsResult").textContent=
    `You: ${choice} | Computer: ${computer} | ${result}`;
  if(result==="You win!") rewardGame();
};

function sportsQuiz() {
  document.getElementById("gameArea").innerHTML=`
    <h3>Which sport uses a racket and shuttlecock?</h3>
    <button class="game-button" onclick="quizAnswer(false)">Tennis</button>
    <button class="game-button" onclick="quizAnswer(true)">Badminton</button>
    <button class="game-button" onclick="quizAnswer(false)">Football</button>
    <p id="quizResult"></p>`;
}

window.quizAnswer=function(correct) {
  document.getElementById("quizResult").textContent=
    correct?"Correct! Well done!":"Not quite. Try again.";
  if(correct) rewardGame();
};

function memoryGame() {
  const symbols=["🍎","🚀","⭐","🎮","🍎","🚀","⭐","🎮"];
  symbols.sort(()=>Math.random()-.5);
  let opened=[];
  let matched=[];
  let busy=false;

  const area=document.getElementById("gameArea");
  function draw() {
    area.innerHTML=`<div style="display:grid;grid-template-columns:repeat(4,1fr);gap:8px">
    ${symbols.map((s,i)=>`
      <button class="game-button" onclick="memoryFlip(${i})">
        ${matched.includes(i)||opened.includes(i)?s:"❔"}
      </button>`).join("")}</div>`;
  }

  window.memoryFlip=function(i) {
    if(busy||matched.includes(i)||opened.includes(i))return;
    opened.push(i);
    draw();
    if(opened.length===2) {
      busy=true;
      if(symbols[opened[0]]===symbols[opened[1]]) {
        matched.push(...opened);
        opened=[];
        busy=false;
        if(matched.length===symbols.length) {
          area.innerHTML+="<h3>You won!</h3>";
          rewardGame();
        }
        draw();
      } else {
        setTimeout(()=>{
          opened=[];
          busy=false;
          draw();
        },800);
      }
    }
  };
  draw();
}

function reactionGame() {
  let started=false;
  let startTime=0;
  const area=document.getElementById("gameArea");
  area.innerHTML=`<h3>Reaction challenge</h3>
    <p>Tap the button when it turns green.</p>
    <button id="reactionButton" class="game-button">Start</button>`;
  const btn=document.getElementById("reactionButton");
  let timer;

  btn.onclick=()=>{
    if(!started) {
      started=true;
      btn.textContent="Wait...";
      btn.style.background="#c65c25";
      timer=setTimeout(()=>{
        startTime=Date.now();
        btn.textContent="TAP NOW!";
        btn.style.background="#159447";
      },1500+Math.random()*3000);
    } else if(startTime) {
      const elapsed=Date.now()-startTime;
      btn.textContent=`${elapsed} ms`;
      rewardGame();
      started=false;
      startTime=0;
    } else {
      clearTimeout(timer);
      btn.textContent="Too early! Try again.";
      started=false;
    }
  };
}

function createGameRoom() {
  const name=document.getElementById("roomName").value.trim();
  if(!name) {
    document.getElementById("roomMessage").textContent="Enter a room name.";
    return;
  }
  const code=Math.random().toString(36).substring(2,8).toUpperCase();
  document.getElementById("roomMessage").textContent=
    `Room ${name} created. Demo code: ${code}`;
}

function joinGameRoom() {
  const code=document.getElementById("joinCode").value.trim();
  document.getElementById("roomMessage").textContent=code?
    `Room code ${code} entered. Online room connection needs a server.`:
    "Enter a room code.";
}

function dailyChallenge() {
  if(dailyClaimed) {
    alert("You have already claimed today's demo challenge.");
    return;
  }
  gamePoints+=10;
  dailyClaimed=true;
  localStorage.setItem("striponDaily","yes");
  localStorage.setItem("striponPoints",gamePoints);
  updateGamingStats();
  alert("10 demo points added!");
}

filterGames("all");
updateGamingStats();
.gaming-hero {
  background:linear-gradient(120deg,#321078,#7419a8,#e12b85);
  border-radius:22px;
  padding:25px;
  display:flex;
  align-items:center;
  justify-content:space-between;
  color:white;
  min-height:160px;
}
.gaming-hero h1 {font-size:30px;margin:12px 0}
.gaming-tag {
  background:#ffffff25;
  padding:7px 12px;
  border-radius:20px;
  font-size:11px;
}
.game-controller {font-size:65px}
.gaming-stats {
  display:grid;
  grid-template-columns:repeat(3,1fr);
  gap:10px;
  margin:18px 0;
}
.gaming-stats div {
  background:var(--panel);
  padding:15px 8px;
  border-radius:13px;
  text-align:center;
}
.gaming-stats strong {display:block;font-size:20px;color:var(--accent)}
.gaming-stats span {font-size:11px;color:var(--muted)}
.game-categories {
  display:flex;
  gap:8px;
  overflow-x:auto;
  padding:10px 0;
}
.game-grid {
  display:grid;
  grid-template-columns:repeat(auto-fill,minmax(145px,1fr));
  gap:13px;
}
.game-tile {
  background:var(--panel);
  border-radius:17px;
  overflow:hidden;
}
.game-art {
  height:115px;
  display:grid;
  place-items:center;
  font-size:55px;
  background:linear-gradient(135deg,#25104b,#a52b8a);
}
.game-tile h3 {padding:0 10px;font-size:15px}
.game-tile p {padding:0 10px;color:var(--muted);font-size:12px}
.game-tile button {margin:0 10px 12px;width:calc(100% - 20px)}
.gaming-panel {
  background:var(--panel);
  padding:18px;
  border-radius:17px;
  margin:15px 0;
}
.game-modal {
  display:none;
  position:fixed;
  inset:0;
  z-index:100;
  background:#000e;
  padding:15px;
  overflow:auto;
}
.game-window {
  max-width:600px;
  margin:35px auto;
  background:var(--panel);
  border-radius:18px;
  padding:20px;
}
.close-game {float:right;background:#d12b57}
#gameArea {
  min-height:230px;
  padding:15px;
  text-align:center;
}
.game-button {
  padding:16px;
  margin:5px;
  font-size:18px;
}
@media(max-width:450px) {
  .gaming-hero h1 {font-size:24px}
  .game-grid {grid-template-columns:repeat(2,minmax(0,1fr))}
}
<section class="page" id="gaming">

  <div class="gaming-hero">
    <div>
      <span class="gaming-tag">STRIPON PLAY</span>
      <h1>Gaming Universe 🎮</h1>
      <p>Play. Compete. Connect.</p>
    </div>
    <div class="game-controller">🎮</div>
  </div>

  <div class="gaming-stats">
    <div><strong id="gamesPlayed">0</strong><span>Games played</span></div>
    <div><strong id="playerScore">0</strong><span>Player points</span></div>
    <div><strong id="playerRank">Rookie</strong><span>Your rank</span></div>
  </div>

  <h2>Explore games</h2>
  <div class="game-categories">
    <button onclick="filterGames('all')">All</button>
    <button onclick="filterGames('action')">Action</button>
    <button onclick="filterGames('puzzle')">Puzzle</button>
    <button onclick="filterGames('sports')">Sports</button>
    <button onclick="filterGames('arcade')">Arcade</button>
  </div>

  <div class="game-grid" id="gameGrid"></div>

  <h2>🌐 Multiplayer Arena</h2>
  <div class="gaming-panel">
    <h3>Play with friends</h3>
    <p>Create a room and invite friends using its room code.</p>
    <input id="roomName" placeholder="Enter room name">
    <button onclick="createGameRoom()">Create room</button>
    <input id="joinCode" placeholder="Enter room code">
    <button onclick="joinGameRoom()">Join room</button>
    <p id="roomMessage"></p>
  </div>

  <h2>🏆 Leaderboard</h2>
  <div class="gaming-panel">
    <div id="leaderboard">
      <p>1. STRIPON Player — 500 points</p>
      <p>2. Gaming Star — 350 points</p>
      <p>3. Pro Gamer — 200 points</p>
    </div>
  </div>

  <div class="gaming-panel">
    <h3>🎯 Daily challenge</h3>
    <p>Play a game today and earn 10 demo points.</p>
    <button onclick="dailyChallenge()">Claim challenge</button>
  </div>

</section>

<div id="gameModal" class="game-modal">
  <div class="game-window">
    <button onclick="closeGame()" class="close-game">✕</button>
    <h2 id="activeGameTitle">Game</h2>
    <div id="gameArea"></div>
  </div>
</div>

