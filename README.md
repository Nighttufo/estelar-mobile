<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Stellar Mobile Pro</title>
    <style>
        :root {
            --space-black: #050508;
            --star-cyan: #00f2ff;
            --glass: rgba(12, 12, 20, 0.98);
            --accent: #00f2ff;
        }

        body.mode-black { --space-black: #000; --accent: #666; --star-cyan: #fff; }
        
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Segoe UI', sans-serif; display: flex; color: #e0e0e0; touch-action: none; }

        /* Sidebar adaptável */
        #sidebar { 
            position: fixed; left: 0; top: 0; width: 280px; height: 100vh; 
            background: var(--glass); padding: 20px; box-sizing: border-box; 
            display: flex; flex-direction: column; border-right: 1px solid rgba(255, 255, 255, 0.1); 
            z-index: 1000; transition: transform 0.3s ease;
        }

        /* Esconder sidebar no mobile por padrão */
        @media (max-width: 768px) {
            #sidebar { transform: translateX(-100%); width: 85%; }
            #sidebar.active { transform: translateX(0); }
        }

        /* Botão para abrir menu no mobile */
        #menu-toggle {
            position: fixed; bottom: 20px; right: 20px; width: 60px; height: 60px;
            background: var(--star-cyan); color: #000; border-radius: 50%;
            display: flex; align-items: center; justify-content: center;
            font-weight: bold; font-size: 24px; z-index: 999; cursor: pointer;
            box-shadow: 0 0 15px var(--star-cyan); border: none;
        }

        .map-explorer { flex-grow: 1; overflow-y: auto; margin: 20px 0; }
        .map-item { background: rgba(255, 255, 255, 0.05); padding: 15px; border-radius: 8px; margin-top: 10px; font-size: 0.9em; display: flex; justify-content: space-between; align-items: center; }
        .active-map { border: 1px solid var(--star-cyan); background: rgba(0, 242, 255, 0.1); }

        button { cursor: pointer; padding: 12px; border: 1px solid var(--accent); border-radius: 6px; background: transparent; color: var(--accent); font-weight: bold; width: 100%; margin-bottom: 10px; }

        #custom-modal { position: fixed; top: 20%; left: 50%; transform: translateX(-50%); background: #111; padding: 25px; border: 1px solid var(--star-cyan); z-index: 2000; display: none; width: 80%; max-width: 350px; border-radius: 12px; }
        #custom-modal.active { display: block; }
        input { width: 100%; padding: 15px; margin-bottom: 15px; background: #000; border: 1px solid #333; color: #fff; box-sizing: border-box; border-radius: 6px; }

        #overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.8); z-index: 998; display: none; }
        canvas { flex-grow: 1; background: var(--space-black); width: 100vw; height: 100vh; }
    </style>
</head>
<body>

<button id="menu-toggle" onclick="toggleSidebar()">☰</button>
<div id="overlay" onclick="closeAll()"></div>

<div id="sidebar">
    <h2 style="color:var(--star-cyan);">STELLAR PRO</h2>
    <div id="map-list" class="map-explorer"></div>
    <div class="controls">
        <button onclick="openModal('map')">+ NOVO SETOR</button>
        <button onclick="saveCurrentMap()">SALVAR</button>
        <button onclick="clearAllData()" style="color:#ff4757; border-color:#ff4757">LIMPAR TUDO</button>
    </div>
</div>

<div id="custom-modal">
    <h3 id="modal-title" style="color:var(--star-cyan)">NOMEAR</h3>
    <input type="text" id="modal-input" placeholder="Identificação...">
    <div style="display:flex; gap:10px;">
        <button id="modal-confirm-btn">SALVAR</button>
        <button onclick="closeModal()">VOLTAR</button>
    </div>
</div>

<canvas id="canvas"></canvas>

<script>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const sidebar = document.getElementById('sidebar');
const modal = document.getElementById('custom-modal');
const overlay = document.getElementById('overlay');
const modalInput = document.getElementById('modal-input');

let maps = JSON.parse(localStorage.getItem('stellar_mobile_v1')) || {}; 
let currentMapName = localStorage.getItem('last_mobile_map') || null;
let points = (currentMapName && maps[currentMapName]) ? [...maps[currentMapName]] : [];
let camera = { x: 0, y: 0, zoom: 1 };
let isDragging = false;
let lastTouchPos = { x: 0, y: 0 };
let touchStartTime = 0;

function init() {
    window.onresize = () => { canvas.width = window.innerWidth; canvas.height = window.innerHeight; };
    window.onresize();
    renderMapList();
    draw();
}

function toggleSidebar() { sidebar.classList.toggle('active'); overlay.style.display = sidebar.classList.contains('active') ? 'block' : 'none'; }
function closeAll() { sidebar.classList.remove('active'); closeModal(); overlay.style.display = 'none'; }

function openModal(type, data = null) {
    overlay.style.display = 'block';
    modal.classList.add('active');
    modalInput.value = "";
    modalInput.focus();
    document.getElementById('modal-confirm-btn').onclick = () => {
        const val = modalInput.value.trim();
        if (type === 'map' && val) { maps[val] = []; switchMap(val); }
        else if (type === 'point' && currentMapName) { points.push({x: data.x, y: data.y, label: val || `P-${points.length + 1}`}); saveCurrentMap(); }
        closeAll();
    };
}

function closeModal() { modal.classList.remove('active'); }

function renderMapList() {
    const list = document.getElementById('map-list');
    list.innerHTML = '';
    Object.keys(maps).forEach(name => {
        const div = document.createElement('div');
        div.className = `map-item ${name === currentMapName ? 'active-map' : ''}`;
        div.innerHTML = `<span>${name.toUpperCase()}</span> <span onclick="deleteMap('${name}', event)">✖</span>`;
        div.onclick = () => { switchMap(name); closeAll(); };
        list.appendChild(div);
    });
}

function switchMap(name) {
    currentMapName = name;
    points = [...maps[name]];
    localStorage.setItem('last_mobile_map', name);
    renderMapList();
}

function deleteMap(name, e) {
    e.stopPropagation();
    if(confirm("Excluir?")) { delete maps[name]; saveToLocal(); renderMapList(); }
}

function saveCurrentMap() { if(currentMapName) { maps[currentMapName] = points; saveToLocal(); } }
function saveToLocal() { localStorage.setItem('stellar_mobile_v1', JSON.stringify(maps)); }
function clearAllData() { if(confirm("Apagar tudo?")) { localStorage.clear(); location.reload(); } }

function draw() {
    ctx.fillStyle = getComputedStyle(document.body).getPropertyValue('--space-black');
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    if(!currentMapName) {
        ctx.fillStyle = "#444"; ctx.textAlign = "center";
        ctx.fillText("USE O MENU PARA CRIAR UM SETOR", canvas.width/2, canvas.height/2);
        requestAnimationFrame(draw); return;
    }

    ctx.save();
    ctx.translate(canvas.width/2 + camera.x * camera.zoom, canvas.height/2 + camera.y * camera.zoom);
    ctx.scale(camera.zoom, camera.zoom);

    // Linhas e Pontos
    ctx.beginPath();
    ctx.strokeStyle = "rgba(0, 242, 255, 0.3)";
    if(points.length > 0) ctx.moveTo(points[0].x, points[0].y);
    points.forEach(p => {
        ctx.lineTo(p.x, p.y);
        ctx.fillStyle = "#fff";
        ctx.font = "bold 11px Arial"; ctx.textAlign = "center";
        ctx.fillText(p.label.toUpperCase(), p.x, p.y - 12);
    });
    ctx.stroke();

    points.forEach(p => {
        ctx.beginPath(); ctx.arc(p.x, p.y, 4, 0, Math.PI*2); ctx.fill();
    });

    ctx.restore();
    requestAnimationFrame(draw);
}

// EVENTOS MOBILE (TOUCH)
canvas.addEventListener('touchstart', e => {
    touchStartTime = Date.now();
    const touch = e.touches[0];
    lastTouchPos = { x: touch.clientX, y: touch.clientY };
    isDragging = true;
});

canvas.addEventListener('touchmove', e => {
    if (!isDragging) return;
    const touch = e.touches[0];
    const dx = touch.clientX - lastTouchPos.x;
    const dy = touch.clientY - lastTouchPos.y;
    camera.x += dx / camera.zoom;
    camera.y += dy / camera.zoom;
    lastTouchPos = { x: touch.clientX, y: touch.clientY };
});

canvas.addEventListener('touchend', e => {
    isDragging = false;
    const touchDuration = Date.now() - touchStartTime;
    // Se for um toque rápido (menos de 200ms), cria um ponto
    if (touchDuration < 200) {
        const rect = canvas.getBoundingClientRect();
        const x = (lastTouchPos.x - canvas.width/2 - camera.x * camera.zoom) / camera.zoom;
        const y = (lastTouchPos.y - canvas.height/2 - camera.y * camera.zoom) / camera.zoom;
        openModal('point', {x, y});
    }
});

init();
</script>
</body>
</html>
