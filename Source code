<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Real-time 3D Modeling Studio</title>
<style>
* { margin:0; padding:0; box-sizing:border-box; }
body { font-family:Arial, sans-serif; background:#0a0a0a; color:#fff; overflow:hidden; }
#container { display:flex; height:100vh; }
#sidebar { width:300px; background:#1a1a1a; padding:20px; overflow-y:auto; border-right:1px solid #333; }
#viewport { flex:1; position:relative; background:#000; }
#toolbar { position:absolute; top:20px; left:20px; background:rgba(26,26,26,0.9); padding:15px; border-radius:8px; backdrop-filter:blur(10px); }
.tool-button { background:#333; border:none; color:white; padding:10px 15px; margin:5px; border-radius:4px; cursor:pointer; transition:all 0.3s ease; }
.tool-button:hover { background:#555; transform:translateY(-2px); }
.tool-button.active { background:#0066cc; }
#properties { margin-top:20px; }
.property-group { margin-bottom:20px; }
.property-group h3 { margin-bottom:10px; color:#0066cc; }
input[type="range"], input[type="color"], input[type="number"] { width:100%; margin:5px 0; }
#user-list { position:absolute; top:20px; right:20px; background:rgba(26,26,26,0.9); padding:15px; border-radius:8px; backdrop-filter:blur(10px); }
.user-indicator { display:inline-block; width:10px; height:10px; border-radius:50%; margin-right:5px; }
#connection-status { position:absolute; bottom:20px; left:20px; padding:10px; border-radius:4px; font-size:12px; }
.connected { background:rgba(0,255,0,0.2); border:1px solid #00ff00; }
.disconnected { background:rgba(255,0,0,0.2); border:1px solid #ff0000; }
</style>

<script src="https://cdn.socket.io/4.7.2/socket.io.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/TransformControls.js"></script>
</head>
<body>
<div id="container">
    <div id="sidebar">
        <h2>3D Modeling Studio</h2>
        <div id="session-controls">
            <button class="tool-button" onclick="createNewSession()">New Session</button>
            <button class="tool-button" onclick="joinSession()">Join Session</button>
            <input type="text" id="session-id" placeholder="Session ID" style="width:100%; margin:10px 0; padding:8px;">
        </div>
        <div id="properties">
            <div class="property-group">
                <h3>Primitives</h3>
                <button class="tool-button" onclick="addPrimitive('cube')">Cube</button>
                <button class="tool-button" onclick="addPrimitive('sphere')">Sphere</button>
                <button class="tool-button" onclick="addPrimitive('cylinder')">Cylinder</button>
                <button class="tool-button" onclick="addPrimitive('plane')">Plane</button>
            </div>
            <div class="property-group">
                <h3>Transform</h3>
                <label>Position X: <span id="pos-x">0</span></label>
                <input type="range" id="position-x" min="-10" max="10" step="0.1" value="0">
                <label>Position Y: <span id="pos-y">0</span></label>
                <input type="range" id="position-y" min="-10" max="10" step="0.1" value="0">
                <label>Position Z: <span id="pos-z">0</span></label>
                <input type="range" id="position-z" min="-10" max="10" step="0.1" value="0">
            </div>
            <div class="property-group">
                <h3>Material</h3>
                <label>Color:</label>
                <input type="color" id="material-color" value="#ff0000">
                <label>Opacity: <span id="opacity-value">1</span></label>
                <input type="range" id="material-opacity" min="0" max="1" step="0.1" value="1">
                <label>Metalness: <span id="metalness-value">0</span></label>
                <input type="range" id="material-metalness" min="0" max="1" step="0.1" value="0">
                <label>Roughness: <span id="roughness-value">0.5</span></label>
                <input type="range" id="material-roughness" min="0" max="1" step="0.1" value="0.5">
            </div>
            <div class="property-group">
                <h3>Actions</h3>
                <button class="tool-button" onclick="deleteSelectedObject()">Delete</button>
                <button class="tool-button" onclick="duplicateSelectedObject()">Duplicate</button>
                <button class="tool-button" onclick="resetScene()">Reset</button>
            </div>
        </div>
    </div>

    <div id="viewport">
        <div id="toolbar">
            <button class="tool-button active" onclick="setTool('select')" title="Select">🔍</button>
            <button class="tool-button" onclick="setTool('translate')" title="Move">↔️</button>
            <button class="tool-button" onclick="setTool('rotate')" title="Rotate">🔄</button>
            <button class="tool-button" onclick="setTool('scale')" title="Scale">📐</button>
        </div>
        <div id="user-list">
            <h4>Active Users</h4>
            <div id="users"></div>
        </div>
        <div id="connection-status" class="disconnected">Disconnected</div>
    </div>
</div>

<script>
let scene, camera, renderer, controls, transformControls;
let socket;
let currentSession = null;
let userId = null;
let selectedObject = null;
let raycaster, mouse;
let objects = new Map();
let currentTool = 'select';

// --- Scene ---
function initScene() {
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x0a0a0a);

    camera = new THREE.PerspectiveCamera(75, (window.innerWidth-300)/window.innerHeight, 0.1, 1000);
    camera.position.set(5,5,5);

    renderer = new THREE.WebGLRenderer({antialias:true});
    renderer.setSize(window.innerWidth-300, window.innerHeight);
    renderer.shadowMap.enabled = true;
    document.getElementById('viewport').appendChild(renderer.domElement);

    controls = new THREE.OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true; controls.dampingFactor=0.05;

    transformControls = new THREE.TransformControls(camera, renderer.domElement);
    transformControls.addEventListener('change', render);
    transformControls.addEventListener('dragging-changed', e=>{controls.enabled=!e.value});
    transformControls.addEventListener('objectChange', ()=>{ 
        if(selectedObject) emitObjectUpdate(selectedObject);
    });
    scene.add(transformControls);

    scene.add(new THREE.AmbientLight(0x404040,0.6));
    const dirLight = new THREE.DirectionalLight(0xffffff,0.8);
    dirLight.position.set(10,10,5); dirLight.castShadow=true;
    scene.add(dirLight);

    scene.add(new THREE.GridHelper(20,20,0x444444,0x444444));
    scene.add(new THREE.AxesHelper(5));

    raycaster = new THREE.Raycaster();
    mouse = new THREE.Vector2();

    renderer.domElement.addEventListener('click', onMouseClick);
    renderer.domElement.addEventListener('mousemove', onMouseMove);
    window.addEventListener('resize', onWindowResize);

    animate();
}

// --- Socket.IO ---
function connectToServer() {
    socket = io();

    socket.on('connect', ()=>{ 
        document.getElementById('connection-status').textContent='Connected';
        document.getElementById('connection-status').className='connected';
    });
    socket.on('disconnect', ()=>{ 
        document.getElementById('connection-status').textContent='Disconnected';
        document.getElementById('connection-status').className='disconnected';
    });
    socket.on('user_joined', data => { 
        updateUserList(data.model_state.users);
        if(data.model_state.objects) syncScene(data.model_state.objects);
    });
    socket.on('user_left', data => removeUserCursor(data.user_id));
    socket.on('object_added', data => createRemoteObject(data.object));
    socket.on('object_updated', data => updateRemoteObject(data.object_id,data.updates));
    socket.on('object_deleted', data => deleteRemoteObject(data.object_id));
}

// --- Session ---
async function createNewSession(){
    const res = await fetch('/api/create_session',{method:'POST',headers:{'Content-Type':'application/json'}});
    const data = await res.json();
    currentSession = data.session_id;
    userId = 'user_'+Date.now();
    document.getElementById('session-id').value = currentSession;
    socket.emit('join_session',{session_id:currentSession,user_id:userId});
}

function joinSession(){
    const sessionId = document.getElementById('session-id').value;
    if(sessionId){ currentSession=sessionId; userId='user_'+Date.now(); socket.emit('join_session',{session_id:currentSession,user_id:userId}); }
}

// --- Objects ---
function addPrimitive(type){
    if(!currentSession){ alert('Create or join session first'); return; }

    let geo;
    switch(type){
        case 'cube': geo=new THREE.BoxGeometry(1,1,1); break;
        case 'sphere': geo=new THREE.SphereGeometry(0.5,32,32); break;
        case 'cylinder': geo=new THREE.CylinderGeometry(0.5,0.5,1,32); break;
        case 'plane': geo=new THREE.PlaneGeometry(2,2); break;
    }

    const mat=new THREE.MeshStandardMaterial({
        color:document.getElementById('material-color').value,
        transparent:true,
        opacity:parseFloat(document.getElementById('material-opacity').value),
        metalness:parseFloat(document.getElementById('material-metalness').value),
        roughness:parseFloat(document.getElementById('material-roughness').value)
    });

    const mesh=new THREE.Mesh(geo,mat);
    mesh.position.set(parseFloat(document.getElementById('position-x').value),
                      parseFloat(document.getElementById('position-y').value),
                      parseFloat(document.getElementById('position-z').value));
    mesh.castShadow=true; mesh.receiveShadow=true;

    const objData={
        type:type,
        position:mesh.position.toArray(),
        rotation:mesh.rotation.toArray(),
        scale:mesh.scale.toArray(),
        material:{color:mat.color.getHex(),opacity:mat.opacity,metalness:mat.metalness,roughness:mat.roughness}
    };

    socket.emit('add_object',{session_id:currentSession,object:objData,user_id:userId});
}

// --- Sync & Transform ---
function createRemoteObject(obj){ 
    if(objects.has(obj.id)) return;
    let geo;
    switch(obj.type){ case'cube':geo=new THREE.BoxGeometry(1,1,1);break; case'sphere':geo=new THREE.SphereGeometry(0.5,32,32);break; case'cylinder':geo=new THREE.CylinderGeometry(0.5,0.5,1,32);break; case'plane':geo=new THREE.PlaneGeometry(2,2);break;}
    const mat=new THREE.MeshStandardMaterial(obj.material);
    const mesh=new THREE.Mesh(geo,mat);
    mesh.position.fromArray(obj.position); mesh.rotation.fromArray(obj.rotation); mesh.scale.fromArray(obj.scale); mesh.userData.id=obj.id;
    scene.add(mesh); objects.set(obj.id,mesh);
}

function updateRemoteObject(id,updates){ if(objects.has(id)){ const obj=objects.get(id); if(updates.position)obj.position.fromArray(updates.position); if(updates.rotation)obj.rotation.fromArray(updates.rotation); if(updates.scale)obj.scale.fromArray(updates.scale); } }

function deleteRemoteObject(id){ if(objects.has(id)){ scene.remove(objects.get(id)); objects.delete(id); } }

function emitObjectUpdate(obj){
    if(!currentSession||!obj.userData.id) return;
    socket.emit('update_object',{session_id:currentSession,object_id:obj.userData.id,updates:{position:obj.position.toArray(),rotation:obj.rotation.toArray(),scale:obj.scale.toArray()},user_id:userId});
}

// --- Selection ---
function onMouseClick(event){
    if(currentTool!=='select') return;
    mouse.x=((event.clientX-300)/(window.innerWidth-300))*2-1;
    mouse.y=-(event.clientY/window.innerHeight)*2+1;
    raycaster.setFromCamera(mouse,camera);
    const inter=raycaster.intersectObjects([...scene.children].filter(o=>o.type==='Mesh'));
    selectObject(inter.length>0?inter[0].object:null);
}

function selectObject(obj){
    if(selectedObject) transformControls.detach();
    selectedObject=obj;
    if(obj) transformControls.attach(obj);
    render();
}

// --- User List ---
function updateUserList(users){ const div=document.getElementById('users'); div.innerHTML=''; users.forEach(u=>{ const el=document.createElement('div'); el.textContent=u; div.appendChild(el); }); }

// --- Animation ---
function animate(){ requestAnimationFrame(animate); controls.update(); renderer.render(scene,camera); }
function render(){ renderer.render(scene,camera); }
function onWindowResize(){ camera.aspect=(window.innerWidth-300)/window.innerHeight; camera.updateProjectionMatrix(); renderer.setSize(window.innerWidth-300,window.innerHeight); render(); }

// --- Init ---
window.addEventListener('load',()=>{ initScene(); connectToServer(); });
</script>
</body>
</html>
