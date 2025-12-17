<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Pokémon Adventure – Catch Them All</title>
<style>
body { margin:0; overflow:hidden; background:#87ceeb; }
#ui { position:absolute; top:10px; left:10px; color:white; font-family:Arial,sans-serif; z-index:10; }
.bar { width:200px; height:12px; background:#333; margin-bottom:6px; }
.fill { height:100%; background:#22c55e; }
#inventory { position:absolute; top:70px; left:10px; color:white; font-family:Arial,sans-serif; z-index:10; }
.pokemonSlot { display:inline-block; width:50px; height:50px; margin-right:5px; border:1px solid #fff; text-align:center; line-height:50px; }
#cutscene { position:absolute; top:0; left:0; width:100%; height:100%; background:black; color:white; display:flex; justify-content:center; align-items:center; font-size:2em; z-index:20; text-align:center; padding:20px; }
#battleUI { position:absolute; bottom:10px; left:50%; transform:translateX(-50%); color:white; font-family:Arial,sans-serif; display:none; z-index:15; background:rgba(0,0,0,0.5); padding:10px; border-radius:8px; }
button { margin:5px; padding:8px 12px; font-size:1em; }
</style>
</head>
<body>
<div id="ui">
<h2>⚡ Pokémon Adventure – Catch Them All</h2>
<div class="bar"><div id="playerHealth" class="fill"></div></div>
<p id="mission">Mission: Explore and catch all Pokémon!</p>
<p>WASD move • SPACE jump • CLICK attack • C throw Pokéball • 1-6 select Pokémon</p>
</div>
<div id="inventory"></div>
<div id="cutscene"></div>
<div id="battleUI">
<p id="battleText">Battle Started!</p>
<button onclick="attack('move1')">Move 1</button>
<button onclick="attack('move2')">Move 2</button>
<button onclick="attack('move3')">Move 3</button>
<button onclick="attack('move4')">Move 4</button>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.155/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.155/examples/js/loaders/GLTFLoader.js"></script>
<script src="https://cdn.socket.io/4.7.5/socket.io.min.js"></script>

<script>
// --- Scene Setup ---
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);
const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias:true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
document.body.appendChild(renderer.domElement);

scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const sun = new THREE.DirectionalLight(0xffffff, 1);
sun.position.set(20,50,20); sun.castShadow=true; scene.add(sun);

// --- Ground ---
const loader = new THREE.TextureLoader();
const groundTex = loader.load('textures/pokemon_grass.jpg');
groundTex.wrapS = groundTex.wrapT = THREE.RepeatWrapping; groundTex.repeat.set(50,50);
const ground = new THREE.Mesh(new THREE.PlaneGeometry(500,500), new THREE.MeshStandardMaterial({ map:groundTex }));
ground.rotation.x=-Math.PI/2; ground.receiveShadow=true; scene.add(ground);

// --- Player (Ash) ---
const gltfLoader = new THREE.GLTFLoader();
let player, playerMixer;
gltfLoader.load('models/ash.glb', gltf=>{
  player = gltf.scene; player.scale.set(2,2,2); player.position.set(0,1,0); scene.add(player);
  playerMixer = new THREE.AnimationMixer(player); gltf.animations.forEach(clip=>playerMixer.clipAction(clip).play());
  player.userData={health:100, velocityY:0, onGround:true, activePokemon:null, team:[]};
});

// --- Pokémon Data ---
const allPokemon = ['Pikachu','Charmander','Squirtle','Bulbasaur','Eevee','Snorlax','Gengar','Jigglypuff','Meowth','Mewtwo'];
const wildPokemons=[];
function spawnPokemon(name,x,z){
  const url=`models/${name.toLowerCase()}.glb`;
  gltfLoader.load(url, gltf=>{
    const p=gltf.scene; p.scale.set(1.5,1.5,1.5); p.position.set(x,1,z); scene.add(p);
    wildPokemons.push({mesh:p, name:name, health:50, captured:false});
  });
}
allPokemon.forEach(p => {
  spawnPokemon(p, (Math.random()-0.5)*200, (Math.random()-0.5)*200);
});

// --- Inventory ---
function updateInventory(){
  const invDiv=document.getElementById('inventory');
  invDiv.innerHTML='';
  player.userData.team.forEach(p => {
    const slot=document.createElement('div');
    slot.className='pokemonSlot'; slot.innerText=p.name; invDiv.appendChild(slot);
  });
}

// --- Controls ---
const keys={}; window.addEventListener('keydown',e=>keys[e.code]=true); window.addEventListener('keyup',e=>keys[e.code]=false);
const gravity=-0.01;
function updatePlayer(){
  if(!player) return;
  if(keys['KeyA']) player.position.x-=0.3;
  if(keys['KeyD']) player.position.x+=0.3;
  if(keys['KeyW']) player.position.z-=0.3;
  if(keys['KeyS']) player.position.z+=0.3;
  if(keys['Space'] && player.userData.onGround){ player.userData.velocityY=0.3; player.userData.onGround=false; }
  player.userData.velocityY+=gravity; player.position.y+=player.userData.velocityY;
  if(player.position.y<=1){ player.position.y=1; player.userData.velocityY=0; player.userData.onGround=true; }
  camera.position.lerp(new THREE.Vector3(player.position.x,player.position.y+6,player.position.z+10),0.1);
  camera.lookAt(player.position);
  document.getElementById('playerHealth').style.width=player.userData.health+'%';
}

// --- Pokéball Capture ---
document.addEventListener('keydown', e => {
  if(e.code==='KeyC'){
    wildPokemons.forEach(wp => {
      if(!wp.captured && player.position.distanceTo(wp.mesh.position)<5){
        const success = Math.random() < 0.7; // 70% capture chance
        if(success){
          wp.captured=true; scene.remove(wp.mesh);
          player.userData.team.push({name:wp.name, health:wp.health});
          updateInventory();
          alert(`You caught ${wp.name}!`);
        } else { alert(`${wp.name} escaped!`); }
      }
    });
  }
});

// --- Animation Loop ---
const clock=new THREE.Clock();
function animate(){
  requestAnimationFrame(animate);
  const delta=clock.getDelta();
  if(playerMixer) playerMixer.update(delta);
  updatePlayer();
  renderer.render(scene,camera);
}
animate();

window.addEventListener('resize',()=>{camera.aspect=window.innerWidth/window.innerHeight; camera.updateProjectionMatrix(); renderer.setSize(window.innerWidth,window.innerHeight);});
</script>
</body>
</html>
