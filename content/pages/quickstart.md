<!DOCTYPE html>
<html lang="pl">
<head>
<meta charset="UTF-8">
<title>Mini GTA Demo 3D</title>
<style>
  body { margin:0; overflow:hidden; }
  canvas { display:block; }
  #info {
    position:absolute; top:10px; left:10px; color:white;
    font-family:sans-serif; font-size:16px;
  }
</style>
</head>
<body>
<div id="info">Wciśnij klik, aby zablokować mysz, WASD do ruchu, spacja do skoku, E do wsiadania do auta, LPM do strzału</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/examples/js/controls/PointerLockControls.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/examples/js/loaders/GLTFLoader.js"></script>

<script>
let scene, camera, renderer, controls;
let player, playerMixer, playerAction;
let car, npcList = [], bullets = [];
let clock = new THREE.Clock();
let keys = {};

// --- Inicjalizacja sceny ---
init();
animate();

function init() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x87CEEB);

    camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);

    renderer = new THREE.WebGLRenderer({antialias:true});
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    // --- Podłoga ---
    const floorGeometry = new THREE.PlaneGeometry(200,200);
    const floorMaterial = new THREE.MeshStandardMaterial({color:0x228B22});
    const floor = new THREE.Mesh(floorGeometry,floorMaterial);
    floor.rotation.x = -Math.PI/2;
    scene.add(floor);

    // --- Światło ---
    const light = new THREE.DirectionalLight(0xffffff,1);
    light.position.set(50,100,50);
    scene.add(light);
    const ambient = new THREE.AmbientLight(0x888888);
    scene.add(ambient);

    // --- Kontrola gracza ---
    controls = new THREE.PointerLockControls(camera, document.body);
    document.body.addEventListener('click', ()=> controls.lock());
    camera.position.y = 2;

    // --- Wczytanie modelu gracza ---
    const loader = new THREE.GLTFLoader();
    loader.load('player.glb', function(gltf){
        player = gltf.scene;
        player.scale.set(1,1,1);
        player.position.set(0,0,0);
        scene.add(player);

        if (gltf.animations.length > 0) {
            playerMixer = new THREE.AnimationMixer(player);
            playerAction = playerMixer.clipAction(gltf.animations[0]);
            playerAction.play();
        }
    });

    // --- Wczytanie samochodu ---
    loader.load('car.glb', function(gltf){
        car = gltf.scene;
        car.scale.set(1.5,1.5,1.5);
        car.position.set(5,0,5);
        scene.add(car);
    });

    // --- NPC (proste sześciany poruszające się w kółko) ---
    for(let i=0;i<3;i++){
        let npc = new THREE.Mesh(new THREE.BoxGeometry(1,2,1), new THREE.MeshStandardMaterial({color:0x0000ff}));
        npc.position.set(Math.random()*20-10,1,Math.random()*20-10);
        npc.direction = new THREE.Vector3(Math.random(),0,Math.random()).normalize();
        npcList.push(npc);
        scene.add(npc);
    }

    // --- Budynki ---
    for(let i=0;i<5;i++){
        let b = new THREE.Mesh(new THREE.BoxGeometry(4,Math.random()*10+5,4), new THREE.MeshStandardMaterial({color:0xaaaaaa}));
        b.position.set(Math.random()*50-25, (b.geometry.parameters.height/2), Math.random()*50-25);
        scene.add(b);
    }

    // --- Eventy klawiszy ---
    document.addEventListener('keydown', e => keys[e.code] = true);
    document.addEventListener('keyup', e => keys[e.code] = false);

    window.addEventListener('resize', ()=>{
        camera.aspect = window.innerWidth/window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
    });
}

// --- Strzały ---
function shoot(){
    const bullet = new THREE.Mesh(new THREE.SphereGeometry(0.1,8,8), new THREE.MeshBasicMaterial({color:0xff0000}));
    bullet.position.copy(player.position);
    bullet.direction = new THREE.Vector3(0,0,-1).applyQuaternion(player.quaternion);
    bullets.push(bullet);
    scene.add(bullet);
}

// --- Animacja ---
function animate(){
    requestAnimationFrame(animate);
    const delta = clock.getDelta();

    // --- Ruch gracza ---
    if(player){
        const speed = 5;
        let move = new THREE.Vector3();
        if(keys['KeyW']) move.z -= 1;
        if(keys['KeyS']) move.z += 1;
        if(keys['KeyA']) move.x -= 1;
        if(keys['KeyD']) move.x += 1;
        move.normalize().multiplyScalar(speed*delta);
        player.position.add(move);
        controls.getObject().position.copy(player.position);
    }

    // --- NPC poruszanie ---
    npcList.forEach(npc=>{
        npc.position.addScaledVector(npc.direction, delta*2);
        if(npc.position.x>50||npc.position.x<-50) npc.direction.x*=-1;
        if(npc.position.z>50||npc.position.z<-50) npc.direction.z*=-1;
    });

    // --- Aktualizacja strzałów ---
    bullets.forEach((b,i)=>{
        b.position.addScaledVector(b.direction, delta*20);
        if(Math.abs(b.position.x)>100||Math.abs(b.position.z)>100){
            scene.remove(b);
            bullets.splice(i,1);
        }
    });

    // --- Animacja modelu ---
    if(playerMixer) playerMixer.update(delta);

    renderer.render(scene,camera);
}

// --- Strzelanie myszką ---
document.addEventListener('mousedown', e=>{
    if(e.button===0) shoot();
});
</script>
</body>
</html>
