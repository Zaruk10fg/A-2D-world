<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Terraria Lite - Web</title>
<style>
  html,body{height:100%;margin:0;background:#7ec0ff;}
  #gameCanvas{display:block;background:#87ceeb;width:100%;height:100%;}
  #ui{position:fixed;left:10px;top:10px;color:#fff;font-family:sans-serif;text-shadow:0 1px 2px rgba(0,0,0,.6)}
  .slot{display:inline-block;padding:6px 10px;border-radius:6px;background:rgba(0,0,0,.25);margin-right:6px}
  .sel{outline:2px solid yellow}
  #help{position:fixed;right:10px;top:10px;color:#fff;font-family:sans-serif;text-shadow:0 1px 2px rgba(0,0,0,.6)}
</style>
</head>
<body>
<canvas id="gameCanvas"></canvas>
<div id="ui"><span class="slot" id="s1">1: Dirt</span><span class="slot" id="s2">2: Stone</span><span class="slot" id="s3">3: Grass</span></div>
<div id="help">Move: A/D or ←/→ • Jump: Space • Left Click: Break • Right Click: Place</div>
<script>
(() => {
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');
  function resize(){ canvas.width = innerWidth; canvas.height = innerHeight; }
  addEventListener('resize', resize); resize();

  const TILE = 32;
  const MAP_W = 200; // columns
  const MAP_H = 60;  // rows
  const GRAVITY = 1500; // px/s^2

  // map: 0 air, 1 dirt, 2 stone, 3 grass
  const map = new Uint8Array(MAP_W * MAP_H);
  function idx(x,y){ return y*MAP_W + x; }

  // simple terrain generation
  for(let x=0;x<MAP_W;x++){
    // make a smooth ground level using sine + randomness
    const h = Math.floor((MAP_H/3) + Math.sin(x*0.08)*6 + (Math.random()*6-3));
    for(let y=h;y<MAP_H;y++){
      let v = 1; // dirt
      if(y>h+6) v = 2; // stone deeper
      map[idx(x,y)] = v;
    }
    // topmost block as grass if exists
    if(h<MAP_H) map[idx(x,h)] = 3;
  }

  // player
  const player = {
    x: MAP_W*TILE/2, y: TILE*5,
    w: 20, h: 30,
    vx:0, vy:0,
    speed: 240,
    onGround: false
  };

  // camera
  const cam = {x:0,y:0};

  // input
  const keys = {};
  addEventListener('keydown', e=>{
    keys[e.key.toLowerCase()] = true;
    if(/^[1-3]$/.test(e.key)) {
      selected = Number(e.key);
      updateUI();
    }
  });
  addEventListener('keyup', e=>{ keys[e.key.toLowerCase()] = false; });

  // mouse
  let mouse = {x:0,y:0,down:false,right:false};
  canvas.addEventListener('mousemove', e=>{
    const r=canvas.getBoundingClientRect();
    mouse.x = (e.clientX - r.left);
    mouse.y = (e.clientY - r.top);
  });
  canvas.addEventListener('mousedown', e=>{
    if(e.button===0) mouse.down=true;
    if(e.button===2) mouse.right=true;
  });
  canvas.addEventListener('mouseup', e=>{
    if(e.button===0) mouse.down=false;
    if(e.button===2) mouse.right=false;
  });
  canvas.addEventListener('contextmenu', e=> e.preventDefault());

  // selection
  let selected = 1; updateUI();
  function updateUI(){
    for(let i=1;i<=3;i++) document.getElementById('s'+i).classList.toggle('sel', i===selected);
  }

  // utility: world to screen and back
  function worldToScreen(wx,wy){ return {x: wx - cam.x, y: wy - cam.y}; }
  function screenToWorld(sx,sy){ return {x: sx + cam.x, y: sy + cam.y}; }

  // physics helpers
  function tileAtWorld(wx,wy){
    const tx = Math.floor(wx / TILE);
    const ty = Math.floor(wy / TILE);
    if(tx<0||tx>=MAP_W||ty<0||ty>=MAP_H) return 0;
    return map[idx(tx,ty)];
  }

  function setTileAtWorld(wx,wy,val){
    const tx = Math.floor(wx / TILE);
    const ty = Math.floor(wy / TILE);
    if(tx<0||tx>=MAP_W||ty<0||ty>=MAP_H) return false;
    map[idx(tx,ty)] = val;
    return true;
  }

  // simple collision detection against map tiles
  function resolveCollisions(dt){
    player.onGround = false;
    // AABB vs tiles around player
    const left = Math.floor((player.x - player.w/2)/TILE);
    const right = Math.floor((player.x + player.w/2)/TILE);
    const top = Math.floor((player.y - player.h)/TILE);
    let bottom = Math.floor(player.y/TILE); // changed to let for reassignment

    // vertical movement
    player.vy += GRAVITY*dt;
    player.y += player.vy*dt;

    // check vertical collisions
    for(let tx=left; tx<=right; tx++){
      for(let ty=top; ty<=bottom; ty++){
        if(tx<0||tx>=MAP_W||ty<0||ty>=MAP_H) continue;
        if(map[idx(tx,ty)]!==0){
          const tileTop = ty*TILE;
          const tileBottom = tileTop + TILE;
          const pBottom = player.y;
          const pTop = player.y - player.h;
          // coming down
          if(player.vy>0 && pBottom > tileTop && pTop < tileTop){
            player.y = tileTop;
            player.vy = 0;
            player.onGround = true;
            bottom = Math.floor(player.y/TILE);
          }
          // coming up
          if(player.vy<0 && pTop < tileBottom && pBottom > tileBottom){
            player.y = tileBottom + player.h;
            player.vy = 0;
          }
        }
      }
    }

    // horizontal movement
    player.x += player.vx*dt;
    const l = Math.floor((player.x - player.w/2)/TILE);
    const r = Math.floor((player.x + player.w/2)/TILE);
    const t = Math.floor((player.y - player.h)/TILE);
    let b = Math.floor(player.y/TILE); // changed to let for reassignment if needed
    for(let tx=l; tx<=r; tx++){
      for(let ty=t; ty<=b; ty++){
        if(tx<0||tx>=MAP_W||ty<0||ty>=MAP_H) continue;
        if(map[idx(tx,ty)]!==0){
          const tileLeft = tx*TILE;
          const tileRight = tileLeft + TILE;
          const pLeft = player.x - player.w/2;
          const pRight = player.x + player.w/2;
          // moving right
          if(player.vx>0 && pRight > tileLeft && pLeft < tileLeft){
            player.x = tileLeft - player.w/2;
            player.vx = 0;
          }
          // moving left
          if(player.vx<0 && pLeft < tileRight && pRight > tileRight){
            player.x = tileRight + player.w/2;
            player.vx = 0;
          }
        }
      }
    }
  }

  // breaking/placing
  let breakTimer = 0;
  const BREAK_SPEED = 0.35; // seconds to break a block

  function handleMouse(dt){
    const w = screenToWorld(mouse.x, mouse.y);
    const tx = Math.floor(w.x / TILE);
    const ty = Math.floor(w.y / TILE);
    if(tx<0||tx>=MAP_W||ty<0||ty>=MAP_H) return;
    if(mouse.down){
      // break
      if(map[idx(tx,ty)]!==0){
        breakTimer += dt;
        if(breakTimer >= BREAK_SPEED){
          map[idx(tx,ty)] = 0;
          breakTimer = 0;
        }
      }
    } else breakTimer = 0;
    if(mouse.right){
      // place selected block adjacent if empty
      if(map[idx(tx,ty)]===0){
        map[idx(tx,ty)] = selected;
      }
      // consume the click so it doesn't spam placement every frame
      mouse.right = false;
    }
  }

  // main loop
  let last = performance.now();
  let dayTime = 0; // seconds

  function loop(now){
    const dt = Math.min((now-last)/1000, 0.033);
    last = now;
    // input
    player.vx = 0;
    if(keys['a'] || keys['arrowleft']) player.vx = -player.speed;
    if(keys['d'] || keys['arrowright']) player.vx = player.speed;
    if((keys[' '] || keys['space']) && player.onGround) { player.vy = -520; player.onGround=false; }

    resolveCollisions(dt);
    handleMouse(dt);

    // camera center on player
    cam.x = player.x - canvas.width/2;
    cam.y = player.y - canvas.height/2;
    // clamp camera to map
    cam.x = Math.max(0, Math.min(cam.x, MAP_W*TILE - canvas.width));
    cam.y = Math.max(0, Math.min(cam.y, MAP_H*TILE - canvas.height));

    // day/night
    dayTime += dt;

    render(now);
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);

  function render(now){
    // sky color shift for day/night
    const dayFraction = (Math.sin(dayTime*0.1)+1)/2; // 0..1
    // base sky
    ctx.fillStyle = '#87ceeb';
    ctx.fillRect(0,0,canvas.width,canvas.height);

    // draw tiles
    const startX = Math.floor(cam.x / TILE);
    const endX = Math.floor((cam.x + canvas.width)/TILE)+1;
    const startY = Math.floor(cam.y / TILE);
    const endY = Math.floor((cam.y + canvas.height)/TILE)+1;

    for(let x=startX; x<=endX; x++){
      for(let y=startY; y<=endY; y++){
        if(x<0||x>=MAP_W||y<0||y>=MAP_H) continue;
        const v = map[idx(x,y)];
        if(v===0) continue;
        const sx = x*TILE - cam.x;
        const sy = y*TILE - cam.y;
        // simple tile colors
        if(v===1){ // dirt
          ctx.fillStyle = '#9b6b3d';
          ctx.fillRect(sx,sy,TILE,TILE);
          // darken bottom edge
          ctx.fillStyle = 'rgba(0,0,0,0.08)'; ctx.fillRect(sx,sy+TILE-4,TILE,4);
        } else if(v===2){ // stone
          ctx.fillStyle = '#7d7d7d'; ctx.fillRect(sx,sy,TILE,TILE);
        } else if(v===3){ // grass
          ctx.fillStyle = '#9b6b3d'; ctx.fillRect(sx,sy+8,TILE,TILE-8);
          ctx.fillStyle = '#3aa33a'; ctx.fillRect(sx,sy,TILE,8);
        }
        // tile border
        ctx.strokeStyle = 'rgba(0,0,0,0.06)'; ctx.strokeRect(sx+0.5,sy+0.5,TILE-1,TILE-1);
      }
    }

    // highlight tile under mouse
    const worldPos = screenToWorld(mouse.x, mouse.y);
    const mtX = Math.floor(worldPos.x/TILE);
    const mtY = Math.floor(worldPos.y/TILE);
    if(mtX>=startX && mtX<=endX && mtY>=startY && mtY<=endY){
      ctx.strokeStyle = 'rgba(255,255,255,0.8)';
      ctx.lineWidth = 2;
      ctx.strokeRect(mtX*TILE-cam.x, mtY*TILE-cam.y, TILE, TILE);
      ctx.lineWidth = 1;
    }

    // player
    const playerScreen = worldToScreen(player.x - player.w/2, player.y - player.h);
    ctx.fillStyle = '#2b6aff';
    ctx.fillRect(playerScreen.x, playerScreen.y, player.w, player.h);
    // simple eyes
    ctx.fillStyle = '#fff'; ctx.fillRect(playerScreen.x+4, playerScreen.y+6,4,4); ctx.fillRect(playerScreen.x+player.w-8, playerScreen.y+6,4,4);

    // HUD: selected block indicator
    ctx.fillStyle = 'rgba(0,0,0,0.4)';
    ctx.fillRect(10, canvas.height-54, 140,44);
    ctx.fillStyle = '#fff';
    ctx.font = '16px sans-serif';
    ctx.fillText('Selected: ' + (selected===1?'Dirt':selected===2?'Stone':'Grass'), 18, canvas.height-32);
  }

  // save on unload
  addEventListener('beforeunload', ()=>{
    try{
      localStorage.setItem('terraria_lite_map', JSON.stringify(Array.from(map)));
      localStorage.setItem('terraria_lite_player', JSON.stringify(player));
    }catch(e){}
  });

  // load if exists
  (function tryLoad(){
    try{
      const m = localStorage.getItem('terraria_lite_map');
      if(m){ const arr = JSON.parse(m); for(let i=0;i<Math.min(arr.length,map.length);i++) map[i]=arr[i]; }
      const p = localStorage.getItem('terraria_lite_player');
      if(p){ const pr = JSON.parse(p); player.x=pr.x; player.y=pr.y; }
    }catch(e){}
  })();

})();
</script>
</body>
</html>
