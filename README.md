# pau23B.github.io
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<title>ScanLine — Prototipo</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script>
  // OpenCV.js se carga de forma asíncrona. Module.onRuntimeInitialized se dispara
  // cuando el motor WASM está listo para usarse.
  window.cvReady = false;
  var Module = {
    onRuntimeInitialized(){
      window.cvReady = true;
      document.dispatchEvent(new Event('cv-ready'));
    }
  };
</script>
<script async src="https://docs.opencv.org/4.x/opencv.js" onerror="document.dispatchEvent(new Event('cv-error'))"></script>
<style>
  :root{
    --bg:#0e0f11;
    --panel:#17181b;
    --panel-2:#1f2124;
    --line:#2a2c30;
    --paper:#f6f3ec;
    --paper-shadow:#00000055;
    --cyan:#35e0b0;
    --cyan-dim:#1c8f6c;
    --amber:#ffb038;
    --danger:#e8604a;
    --text:#eceae3;
    --text-dim:#8b8d92;
    --mono: ui-monospace, "SF Mono", "Cascadia Mono", "Courier New", monospace;
    --sans: -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  }
  *{box-sizing:border-box; -webkit-tap-highlight-color:transparent;}
  html,body{height:100%;}
  body{
    margin:0; background:var(--bg); color:var(--text);
    font-family:var(--sans);
    overflow:hidden;
    -webkit-user-select:none; user-select:none;
  }
  #app{
    max-width:480px; height:100dvh; margin:0 auto;
    display:flex; flex-direction:column;
    background:var(--panel);
    position:relative;
    border-left:1px solid var(--line); border-right:1px solid var(--line);
  }

  /* ---------- HEADER ---------- */
  header{
    display:flex; align-items:center; justify-content:space-between;
    padding:14px 16px 10px;
    flex-shrink:0; gap:8px;
  }
  .brand{ display:flex; align-items:center; gap:8px; }
  .brand .dot{
    width:8px; height:8px; border-radius:50%; background:var(--cyan);
    box-shadow:0 0 8px var(--cyan);
    animation:pulse 2s ease-in-out infinite;
  }
  @keyframes pulse{ 0%,100%{opacity:1;} 50%{opacity:.35;} }
  .brand span{
    font-family:var(--mono); letter-spacing:.14em; font-size:13px;
    text-transform:uppercase; color:var(--text-dim);
  }
  .brand strong{ color:var(--text); font-weight:700; }
  .badges{ display:flex; align-items:center; gap:6px; flex-shrink:0; }
  .page-count, .cv-badge{
    font-family:var(--mono); font-size:11px; color:var(--cyan);
    background:rgba(53,224,176,.08); border:1px solid rgba(53,224,176,.3);
    padding:4px 9px; border-radius:20px; white-space:nowrap;
  }
  .cv-badge.loading{ color:var(--text-dim); background:rgba(255,255,255,.04); border-color:var(--line); }
  .cv-badge.error{ color:var(--danger); background:rgba(232,96,74,.08); border-color:rgba(232,96,74,.3); }

  /* ---------- VIEWS ---------- */
  .view{ flex:1; display:none; min-height:0; flex-direction:column; }
  .view.active{ display:flex; }

  /* ---------- CAMERA VIEW ---------- */
  #camera-view{ position:relative; background:#000; margin:0 12px; border-radius:14px; overflow:hidden; }
  #video{
    width:100%; height:100%; object-fit:cover;
    display:block; background:#000;
  }
  .viewfinder{
    position:absolute; inset:22px; pointer-events:none;
  }
  .corner{ position:absolute; width:26px; height:26px; }
  .corner::before, .corner::after{ content:""; position:absolute; background:var(--cyan); }
  .corner.tl{ top:0; left:0; } .corner.tr{ top:0; right:0; }
  .corner.bl{ bottom:0; left:0; } .corner.br{ bottom:0; right:0; }
  .corner::before{ width:100%; height:2px; }
  .corner::after{ width:2px; height:100%; }
  .corner.tl::before,.corner.tl::after,.corner.tr::before,.corner.tr::after{ top:0; }
  .corner.bl::before,.corner.bl::after,.corner.br::before,.corner.br::after{ bottom:0; }
  .corner.tl::after,.corner.bl::after{ left:0; }
  .corner.tr::after,.corner.br::after{ right:0; }

  .scan-sweep{
    position:absolute; left:0; right:0; height:2px;
    background:linear-gradient(90deg, transparent, var(--cyan) 20%, var(--cyan) 80%, transparent);
    box-shadow:0 0 12px 2px var(--cyan);
    top:0; opacity:.85;
    animation:sweep 3.2s linear infinite;
  }
  @keyframes sweep{
    0%{ top:0%; } 50%{ top:100%; } 50.01%{ top:100%; } 100%{ top:0%; }
  }

  .hud-text{
    position:absolute; bottom:14px; left:14px; right:14px;
    font-family:var(--mono); font-size:10.5px; letter-spacing:.08em;
    color:rgba(255,255,255,.55); text-transform:uppercase;
    display:flex; justify-content:space-between; pointer-events:none;
  }

  .camera-msg{
    position:absolute; inset:0; display:flex; align-items:center; justify-content:center;
    flex-direction:column; gap:14px; padding:30px; text-align:center;
    background:#000; color:var(--text-dim); font-size:13.5px; line-height:1.5;
  }
  .camera-msg button{ margin-top:4px; }

  .flash{
    position:absolute; inset:0; background:#fff; opacity:0; pointer-events:none;
  }
  .flash.go{ animation:flashpop .35s ease-out; }
  @keyframes flashpop{ 0%{opacity:.9;} 100%{opacity:0;} }

  /* ---------- CAMERA CONTROLS ---------- */
  .cam-controls{
    display:flex; align-items:center; justify-content:center; gap:34px;
    padding:20px 16px 10px; flex-shrink:0;
  }
  .icon-btn{
    width:44px; height:44px; border-radius:50%;
    background:var(--panel-2); border:1px solid var(--line);
    color:var(--text-dim); display:flex; align-items:center; justify-content:center;
    cursor:pointer;
  }
  .shutter{
    width:70px; height:70px; border-radius:50%;
    background:transparent; border:3px solid var(--cyan);
    display:flex; align-items:center; justify-content:center;
    cursor:pointer; padding:5px;
  }
  .shutter .inner{
    width:100%; height:100%; border-radius:50%; background:var(--cyan);
    transition:transform .12s ease;
  }
  .shutter:active .inner{ transform:scale(.82); }
  .spacer44{ width:44px; }

  /* ---------- PAGES VIEW ---------- */
  #pages-view{ padding:14px 16px; overflow-y:auto; }
  .empty-state{
    display:flex; flex-direction:column; align-items:center; justify-content:center;
    text-align:center; color:var(--text-dim); gap:10px; padding:60px 20px;
    font-size:13.5px; line-height:1.6;
  }
  .grid{
    display:grid; grid-template-columns:repeat(2,1fr); gap:12px;
  }
  .page-card{
    position:relative; background:var(--paper); border-radius:8px; overflow:hidden;
    box-shadow:0 6px 16px var(--paper-shadow);
    aspect-ratio:3/4;
  }
  .page-card img{ width:100%; height:100%; object-fit:cover; display:block; }
  .page-card .num{
    position:absolute; top:6px; left:6px; background:rgba(0,0,0,.65);
    color:#fff; font-family:var(--mono); font-size:10px; padding:2px 7px; border-radius:10px;
  }
  .page-card .tag{
    position:absolute; bottom:6px; left:6px; background:rgba(53,224,176,.85);
    color:#06231a; font-family:var(--mono); font-size:9px; padding:2px 6px; border-radius:8px;
    text-transform:uppercase; letter-spacing:.04em;
  }
  .page-card .del{
    position:absolute; top:6px; right:6px; width:22px; height:22px; border-radius:50%;
    background:rgba(0,0,0,.65); color:#fff; border:none; font-size:13px; line-height:1;
    cursor:pointer; display:flex; align-items:center; justify-content:center;
  }
  .add-card{
    aspect-ratio:3/4; border-radius:8px; border:1.5px dashed var(--line);
    display:flex; align-items:center; justify-content:center; color:var(--text-dim);
    font-size:26px; cursor:pointer; background:var(--panel-2);
  }

  /* ---------- PDF VIEW ---------- */
  #pdf-view{ padding:14px 16px; overflow-y:auto; }
  #pdf-frame-wrap{
    background:#000; border-radius:10px; overflow:hidden; height:60vh; margin-bottom:14px;
    border:1px solid var(--line);
  }
  #pdf-frame{ width:100%; height:100%; border:none; }
  .pdf-meta{
    font-family:var(--mono); font-size:11.5px; color:var(--text-dim);
    display:flex; justify-content:space-between; margin-bottom:14px;
  }

  /* ---------- BUTTONS ---------- */
  .btn{
    font-family:var(--sans); font-weight:600; font-size:14px;
    border:none; border-radius:10px; padding:13px 18px;
    cursor:pointer; width:100%;
  }
  .btn-primary{ background:var(--cyan); color:#06231a; }
  .btn-secondary{ background:var(--panel-2); color:var(--text); border:1px solid var(--line); }
  .btn-ghost{ background:none; color:var(--text-dim); border:1px solid var(--line); }
  .btn-row{ display:flex; gap:10px; }
  .btn[disabled]{ opacity:.4; cursor:not-allowed; }

  /* ---------- BOTTOM NAV ---------- */
  nav{
    display:flex; border-top:1px solid var(--line); flex-shrink:0;
    padding:6px 10px calc(10px + env(safe-area-inset-bottom));
  }
  nav button{
    flex:1; background:none; border:none; color:var(--text-dim);
    font-family:var(--mono); font-size:10.5px; letter-spacing:.06em; text-transform:uppercase;
    padding:8px 4px; cursor:pointer; display:flex; flex-direction:column; align-items:center; gap:5px;
  }
  nav button .bar{
    width:22px; height:2px; background:transparent; border-radius:2px;
  }
  nav button.active{ color:var(--cyan); }
  nav button.active .bar{ background:var(--cyan); }

  #canvas-hidden{ display:none; }

  /* ---------- REVIEW / AUTO-CROP OVERLAY ---------- */
  .review-overlay{
    position:absolute; inset:0; z-index:50; background:var(--bg);
    display:none; flex-direction:column;
  }
  .review-overlay.active{ display:flex; }
  .review-header{
    padding:12px 16px; display:flex; align-items:center; justify-content:space-between;
    flex-shrink:0;
  }
  .review-header .title{ font-family:var(--mono); font-size:11.5px; letter-spacing:.1em; text-transform:uppercase; color:var(--text-dim); }
  .review-header button{
    background:none; border:none; color:var(--text-dim); font-size:20px; line-height:1; cursor:pointer;
    padding:4px 8px;
  }
  .review-stage{
    flex:1; min-height:0; display:flex; align-items:center; justify-content:center;
    background:#000; padding:14px; overflow:hidden;
  }
  #review-media{ position:relative; display:inline-block; line-height:0; max-width:100%; max-height:100%; }
  #review-canvas{ display:block; max-width:100%; max-height:calc(100dvh - 260px); border-radius:4px; }
  #review-svg{ position:absolute; inset:0; width:100%; height:100%; touch-action:none; }
  .handle{ fill:var(--cyan); stroke:#06231a; stroke-width:3; cursor:grab; }
  .handle:active{ cursor:grabbing; }
  .quad{ fill:rgba(53,224,176,.14); stroke:var(--cyan); stroke-width:3; }

  .review-status{
    font-family:var(--mono); font-size:11.5px; text-align:center; padding:10px 16px 2px;
    color:var(--cyan); flex-shrink:0;
  }
  .review-status.warn{ color:var(--amber); }
  .review-status.muted{ color:var(--text-dim); }
  .review-redo{
    display:block; margin:2px auto 0; background:none; border:none; color:var(--text-dim);
    font-family:var(--mono); font-size:10.5px; text-decoration:underline; cursor:pointer;
  }
  .review-actions{ padding:12px 16px calc(12px + env(safe-area-inset-bottom)); flex-shrink:0; display:flex; gap:8px; }
  .review-actions .btn{ font-size:12.5px; padding:12px 6px; }
</style>
</head>
<body>

<div id="app">

  <header>
    <div class="brand">
      <div class="dot"></div>
      <span><strong>Scan</strong>Line</span>
    </div>
    <div class="badges">
      <div class="cv-badge loading" id="cv-badge">CV: cargando…</div>
      <div class="page-count" id="page-count-badge">0 páginas</div>
    </div>
  </header>

  <!-- CAMERA VIEW -->
  <section class="view active" id="view-camera" style="flex:1; min-height:0; display:flex; flex-direction:column;">
    <div id="camera-view" style="flex:1; margin:0 12px 4px;">
      <video id="video" autoplay playsinline muted></video>
      <div class="viewfinder">
        <div class="corner tl"></div><div class="corner tr"></div>
        <div class="corner bl"></div><div class="corner br"></div>
        <div class="scan-sweep" id="sweep"></div>
      </div>
      <div class="hud-text">
        <span id="hud-status">BUSCANDO DOCUMENTO…</span>
        <span id="hud-clock">--:--:--</span>
      </div>
      <div class="flash" id="flash"></div>
      <div class="camera-msg" id="camera-msg" style="display:none;"></div>
    </div>

    <div class="cam-controls">
      <button class="icon-btn" id="btn-flip" title="Cambiar cámara">⟲</button>
      <button class="shutter" id="btn-shutter" title="Capturar"><div class="inner"></div></button>
      <div class="spacer44"></div>
    </div>
  </section>

  <!-- PAGES VIEW -->
  <section class="view" id="view-pages">
    <div id="pages-container"></div>
  </section>

  <!-- PDF VIEW -->
  <section class="view" id="view-pdf">
    <div id="pdf-empty" class="empty-state">
      Aún no generas ningún PDF.<br>Escanea páginas y pulsa "Generar PDF".
    </div>
    <div id="pdf-content" style="display:none;">
      <div id="pdf-frame-wrap"><iframe id="pdf-frame"></iframe></div>
      <div class="pdf-meta">
        <span id="pdf-name">documento.pdf</span>
        <span id="pdf-size">—</span>
      </div>
      <div class="btn-row">
        <button class="btn btn-secondary" id="btn-download">Descargar</button>
        <button class="btn btn-primary" id="btn-newscan">Nuevo escaneo</button>
      </div>
    </div>
  </section>

  <nav>
    <button class="active" data-tab="camera"><div class="bar"></div>Cámara</button>
    <button data-tab="pages"><div class="bar"></div>Páginas</button>
    <button data-tab="pdf"><div class="bar"></div>PDF</button>
  </nav>

  <!-- REVIEW / AUTO-CROP OVERLAY -->
  <div class="review-overlay" id="review-overlay">
    <div class="review-header">
      <span class="title">Ajustar recorte</span>
      <button id="btn-review-close" title="Cerrar">✕</button>
    </div>
    <div class="review-stage">
      <div id="review-media">
        <canvas id="review-canvas"></canvas>
        <svg id="review-svg"></svg>
      </div>
    </div>
    <div class="review-status muted" id="review-status">Detectando documento…</div>
    <button class="review-redo" id="btn-redo-detect">↺ Detectar de nuevo</button>
    <div class="review-actions">
      <button class="btn btn-ghost" id="btn-discard">Descartar</button>
      <button class="btn btn-secondary" id="btn-fullframe">Foto completa</button>
      <button class="btn btn-primary" id="btn-crop-confirm">Recortar y guardar</button>
    </div>
  </div>

</div>

<canvas id="canvas-hidden"></canvas>

<script>
(function(){
  const state = { pages: [], facingMode: "environment", stream: null, pdfBlobUrl: null };
  const review = { w:0, h:0, corners:null, draggingKey:null, autoDetected:false };

  const video = document.getElementById('video');
  const canvas = document.getElementById('canvas-hidden');
  const cameraMsg = document.getElementById('camera-msg');
  const hudStatus = document.getElementById('hud-status');
  const hudClock = document.getElementById('hud-clock');
  const pageCountBadge = document.getElementById('page-count-badge');
  const cvBadge = document.getElementById('cv-badge');
  const pagesContainer = document.getElementById('pages-container');

  // ---------- OpenCV status badge ----------
  document.addEventListener('cv-ready', ()=>{
    cvBadge.textContent = 'CV: listo';
    cvBadge.classList.remove('loading');
    if(document.getElementById('review-overlay').classList.contains('active') && !review.autoDetected){
      runDetection();
    }
  });
  document.addEventListener('cv-error', ()=>{
    cvBadge.textContent = 'CV: no disponible';
    cvBadge.classList.remove('loading');
    cvBadge.classList.add('error');
  });
  setTimeout(()=>{
    if(!window.cvReady){
      cvBadge.textContent = 'CV: sin conexión';
      cvBadge.classList.remove('loading');
      cvBadge.classList.add('error');
    }
  }, 12000);

  // ---------- Clock in HUD ----------
  setInterval(()=>{
    hudClock.textContent = new Date().toLocaleTimeString('es-MX', {hour12:false});
  }, 1000);

  // ---------- Tabs ----------
  const views = { camera:'view-camera', pages:'view-pages', pdf:'view-pdf' };
  document.querySelectorAll('nav button').forEach(btn=>{
    btn.addEventListener('click', ()=> switchTab(btn.dataset.tab));
  });
  function switchTab(tab){
    document.querySelectorAll('nav button').forEach(b=>b.classList.toggle('active', b.dataset.tab===tab));
    Object.entries(views).forEach(([key,id])=>{
      document.getElementById(id).classList.toggle('active', key===tab);
    });
    if(tab === 'pages') renderPages();
  }

  // ---------- Camera ----------
  async function startCamera(){
    stopCamera();
    hudStatus.textContent = 'INICIANDO CÁMARA…';
    cameraMsg.style.display = 'none';
    video.style.display = 'block';
    try{
      const stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: state.facingMode, width:{ideal:1280}, height:{ideal:1600} },
        audio:false
      });
      state.stream = stream;
      video.srcObject = stream;
      hudStatus.textContent = 'DOCUMENTO EN ENCUADRE';
    }catch(err){
      video.style.display = 'none';
      cameraMsg.style.display = 'flex';
      cameraMsg.innerHTML = `
        <div style="font-size:26px;">⚠</div>
        <div>No se pudo acceder a la cámara.<br><span style="color:var(--text-dim);font-family:var(--mono);font-size:11px;">${escapeHtml(err.message || err.name || 'permiso denegado')}</span></div>
        <button class="btn btn-primary" id="retry-cam" style="width:auto;padding:10px 20px;">Reintentar</button>
      `;
      document.getElementById('retry-cam').addEventListener('click', startCamera);
    }
  }
  function stopCamera(){
    if(state.stream){ state.stream.getTracks().forEach(t=>t.stop()); state.stream = null; }
  }
  function escapeHtml(s){ return String(s).replace(/[&<>"']/g, c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c])); }

  document.getElementById('btn-flip').addEventListener('click', ()=>{
    state.facingMode = state.facingMode === 'environment' ? 'user' : 'environment';
    startCamera();
  });

  // ---------- Capture ----------
  document.getElementById('btn-shutter').addEventListener('click', capture);

  function capture(){
    if(!video.videoWidth) return;
    const w = video.videoWidth, h = video.videoHeight;
    canvas.width = w; canvas.height = h;
    const ctx = canvas.getContext('2d');

    // Realce simple tipo "documento escaneado": contraste + ligera desaturación
    ctx.filter = 'contrast(1.18) saturate(0.85) brightness(1.04)';
    ctx.drawImage(video, 0, 0, w, h);

    const flash = document.getElementById('flash');
    flash.classList.remove('go'); void flash.offsetWidth; flash.classList.add('go');

    openReview(w, h);
  }

  function updateBadge(){
    pageCountBadge.textContent = state.pages.length + (state.pages.length===1 ? ' página' : ' páginas');
  }

  // =================================================================
  // REVIEW / AUTO-CROP (OpenCV.js)
  // =================================================================
  const reviewOverlay = document.getElementById('review-overlay');
  const reviewCanvas = document.getElementById('review-canvas');
  const reviewSvg = document.getElementById('review-svg');
  const reviewStatus = document.getElementById('review-status');

  function openReview(w, h){
    review.w = w; review.h = h; review.autoDetected = false; review.corners = null;

    reviewCanvas.width = w; reviewCanvas.height = h;
    reviewCanvas.getContext('2d').drawImage(canvas, 0, 0);
    reviewSvg.setAttribute('viewBox', `0 0 ${w} ${h}`);

    reviewOverlay.classList.add('active');
    setStatus('Detectando documento…', 'muted');

    if(window.cvReady){ runDetection(); }
    else { review.corners = defaultCorners(w,h); drawQuad(); }
  }

  function closeReview(){
    reviewOverlay.classList.remove('active');
  }
  document.getElementById('btn-review-close').addEventListener('click', closeReview);
  document.getElementById('btn-discard').addEventListener('click', closeReview);
  document.getElementById('btn-redo-detect').addEventListener('click', ()=>{
    if(window.cvReady) runDetection();
  });

  function defaultCorners(w,h){
    const ix = w*0.04, iy = h*0.04;
    return {
      tl:{x:ix, y:iy}, tr:{x:w-ix, y:iy},
      br:{x:w-ix, y:h-iy}, bl:{x:ix, y:h-iy}
    };
  }

  function setStatus(msg, kind){
    reviewStatus.textContent = msg;
    reviewStatus.className = 'review-status' + (kind ? ' '+kind : '');
  }

  function runDetection(){
    setStatus('Detectando documento…', 'muted');
    try{
      const found = detectDocumentCorners(reviewCanvas);
      if(found){
        review.corners = found;
        review.autoDetected = true;
        setStatus('Documento detectado — ajusta las esquinas si hace falta', '');
      } else {
        review.corners = defaultCorners(review.w, review.h);
        review.autoDetected = false;
        setStatus('No se detectó un documento claro — ajusta manualmente', 'warn');
      }
    }catch(e){
      review.corners = defaultCorners(review.w, review.h);
      setStatus('Detección no disponible — ajusta manualmente', 'warn');
    }
    drawQuad();
  }

  // ---- OpenCV: encontrar el contorno de 4 lados más grande ----
  function detectDocumentCorners(canvasEl){
    const src = cv.imread(canvasEl);
    const gray = new cv.Mat();
    const blurred = new cv.Mat();
    const edges = new cv.Mat();
    const kernel = cv.Mat.ones(3,3, cv.CV_8U);
    const contours = new cv.MatVector();
    const hierarchy = new cv.Mat();
    let bestApprox = null, bestArea = 0;

    try{
      cv.cvtColor(src, gray, cv.COLOR_RGBA2GRAY);
      cv.GaussianBlur(gray, blurred, new cv.Size(5,5), 0);
      cv.Canny(blurred, edges, 60, 160);
      cv.dilate(edges, edges, kernel);
      cv.findContours(edges, contours, hierarchy, cv.RETR_LIST, cv.CHAIN_APPROX_SIMPLE);

      const imgArea = src.rows * src.cols;
      for(let i=0; i<contours.size(); i++){
        const cnt = contours.get(i);
        const peri = cv.arcLength(cnt, true);
        const approx = new cv.Mat();
        cv.approxPolyDP(cnt, approx, 0.02*peri, true);
        if(approx.rows === 4 && cv.isContourConvex(approx)){
          const area = Math.abs(cv.contourArea(approx));
          if(area > bestArea && area > imgArea*0.15){
            bestArea = area;
            if(bestApprox) bestApprox.delete();
            bestApprox = approx;
          } else { approx.delete(); }
        } else { approx.delete(); }
        cnt.delete();
      }

      if(!bestApprox) return null;
      const pts = [];
      for(let i=0;i<4;i++){
        pts.push({ x: bestApprox.data32S[i*2], y: bestApprox.data32S[i*2+1] });
      }
      bestApprox.delete();
      return orderCorners(pts);
    } finally {
      src.delete(); gray.delete(); blurred.delete(); edges.delete();
      kernel.delete(); contours.delete(); hierarchy.delete();
    }
  }

  function orderCorners(pts){
    // tl = suma mínima, br = suma máxima, tr = diferencia mínima, bl = diferencia máxima
    const sums = pts.map(p=>p.x+p.y);
    const diffs = pts.map(p=>p.x-p.y);
    const tl = pts[sums.indexOf(Math.min(...sums))];
    const br = pts[sums.indexOf(Math.max(...sums))];
    const tr = pts[diffs.indexOf(Math.max(...diffs))];
    const bl = pts[diffs.indexOf(Math.min(...diffs))];
    return { tl, tr, br, bl };
  }

  // ---- Dibujar el cuadrilátero + manijas arrastrables (SVG) ----
  function drawQuad(){
    const c = review.corners;
    const order = ['tl','tr','br','bl'];
    const pointsAttr = order.map(k=>`${c[k].x},${c[k].y}`).join(' ');
    const r = Math.max(review.w, review.h) * 0.017;

    let svg = `<polygon class="quad" points="${pointsAttr}"></polygon>`;
    order.forEach(k=>{
      svg += `<circle class="handle" data-key="${k}" cx="${c[k].x}" cy="${c[k].y}" r="${r}"></circle>`;
    });
    reviewSvg.innerHTML = svg;

    reviewSvg.querySelectorAll('.handle').forEach(h=>{
      h.addEventListener('pointerdown', onHandleDown);
    });
  }

  function svgPointFromEvent(evt){
    const pt = reviewSvg.createSVGPoint();
    pt.x = evt.clientX; pt.y = evt.clientY;
    const inv = reviewSvg.getScreenCTM().inverse();
    return pt.matrixTransform(inv);
  }

  function onHandleDown(evt){
    evt.preventDefault();
    review.draggingKey = evt.target.getAttribute('data-key');
    evt.target.setPointerCapture(evt.pointerId);
    reviewSvg.addEventListener('pointermove', onHandleMove);
    reviewSvg.addEventListener('pointerup', onHandleUp);
    reviewSvg.addEventListener('pointercancel', onHandleUp);
  }
  function onHandleMove(evt){
    if(!review.draggingKey) return;
    const p = svgPointFromEvent(evt);
    const x = Math.min(Math.max(p.x, 0), review.w);
    const y = Math.min(Math.max(p.y, 0), review.h);
    review.corners[review.draggingKey] = { x, y };
    drawQuad();
  }
  function onHandleUp(){
    review.draggingKey = null;
    reviewSvg.removeEventListener('pointermove', onHandleMove);
    reviewSvg.removeEventListener('pointerup', onHandleUp);
    reviewSvg.removeEventListener('pointercancel', onHandleUp);
  }

  // ---- Confirmar recorte ----
  document.getElementById('btn-fullframe').addEventListener('click', ()=>{
    const dataUrl = reviewCanvas.toDataURL('image/jpeg', 0.92);
    pushPage(dataUrl, review.w, review.h, false);
    closeReview();
  });

  document.getElementById('btn-crop-confirm').addEventListener('click', ()=>{
    let dataUrl, w, h;
    if(window.cvReady){
      const res = warpPerspectiveCrop(reviewCanvas, review.corners);
      dataUrl = res.dataUrl; w = res.w; h = res.h;
    } else {
      const res = rectFallbackCrop(reviewCanvas, review.corners);
      dataUrl = res.dataUrl; w = res.w; h = res.h;
    }
    pushPage(dataUrl, w, h, true);
    closeReview();
  });

  function pushPage(dataUrl, w, h, cropped){
    state.pages.push({ id: Date.now()+Math.random(), dataUrl, w, h, cropped });
    updateBadge();
    hudStatus.textContent = `PÁGINA ${state.pages.length} GUARDADA`;
    setTimeout(()=>{ hudStatus.textContent = 'DOCUMENTO EN ENCUADRE'; }, 1200);
  }

  // ---- Perspectiva real con OpenCV (getPerspectiveTransform + warpPerspective) ----
  function warpPerspectiveCrop(canvasEl, corners){
    const dist = (a,b)=>Math.hypot(a.x-b.x, a.y-b.y);
    const widthTop = dist(corners.tl, corners.tr);
    const widthBottom = dist(corners.bl, corners.br);
    const heightLeft = dist(corners.tl, corners.bl);
    const heightRight = dist(corners.tr, corners.br);
    const outW = Math.max(1, Math.round(Math.max(widthTop, widthBottom)));
    const outH = Math.max(1, Math.round(Math.max(heightLeft, heightRight)));

    const src = cv.imread(canvasEl);
    const dst = new cv.Mat();
    const srcTri = cv.matFromArray(4, 1, cv.CV_32FC2, [
      corners.tl.x, corners.tl.y,
      corners.tr.x, corners.tr.y,
      corners.br.x, corners.br.y,
      corners.bl.x, corners.bl.y
    ]);
    const dstTri = cv.matFromArray(4, 1, cv.CV_32FC2, [
      0,0, outW-1,0, outW-1,outH-1, 0,outH-1
    ]);
    const M = cv.getPerspectiveTransform(srcTri, dstTri);
    const dsize = new cv.Size(outW, outH);
    cv.warpPerspective(src, dst, M, dsize, cv.INTER_LINEAR, cv.BORDER_CONSTANT, new cv.Scalar());

    const outCanvas = document.createElement('canvas');
    outCanvas.width = outW; outCanvas.height = outH;
    cv.imshow(outCanvas, dst);
    const dataUrl = outCanvas.toDataURL('image/jpeg', 0.92);

    src.delete(); dst.delete(); M.delete(); srcTri.delete(); dstTri.delete();
    return { dataUrl, w: outW, h: outH };
  }

  // ---- Recorte rectangular simple (sin OpenCV) como respaldo ----
  function rectFallbackCrop(canvasEl, corners){
    const xs = Object.values(corners).map(p=>p.x);
    const ys = Object.values(corners).map(p=>p.y);
    const sx = Math.min(...xs), sy = Math.min(...ys);
    const sw = Math.max(...xs) - sx, sh = Math.max(...ys) - sy;
    const outCanvas = document.createElement('canvas');
    outCanvas.width = Math.max(1, Math.round(sw));
    outCanvas.height = Math.max(1, Math.round(sh));
    outCanvas.getContext('2d').drawImage(canvasEl, sx, sy, sw, sh, 0, 0, sw, sh);
    return { dataUrl: outCanvas.toDataURL('image/jpeg', 0.92), w: outCanvas.width, h: outCanvas.height };
  }

  // =================================================================
  // PAGES VIEW
  // =================================================================
  function renderPages(){
    if(state.pages.length === 0){
      pagesContainer.innerHTML = `<div class="empty-state">
        Todavía no has escaneado nada.<br>Ve a la pestaña "Cámara" y captura tu primera página.
      </div>`;
      return;
    }
    let html = '<div class="grid">';
    state.pages.forEach((p, i)=>{
      html += `
        <div class="page-card">
          <span class="num">${i+1}</span>
          <button class="del" data-id="${p.id}">✕</button>
          <img src="${p.dataUrl}" alt="Página ${i+1}">
          ${p.cropped ? '<span class="tag">Recortado</span>' : ''}
        </div>`;
    });
    html += `<div class="add-card" id="add-more">+</div></div>
      <div class="btn-row" style="margin-top:16px;">
        <button class="btn btn-secondary" id="btn-clear">Borrar todo</button>
        <button class="btn btn-primary" id="btn-generate">Generar PDF</button>
      </div>`;
    pagesContainer.innerHTML = html;

    pagesContainer.querySelectorAll('.del').forEach(b=>{
      b.addEventListener('click', ()=>{
        const id = Number(b.dataset.id);
        state.pages = state.pages.filter(p=>p.id!==id);
        updateBadge(); renderPages();
      });
    });
    document.getElementById('add-more').addEventListener('click', ()=> switchTab('camera'));
    document.getElementById('btn-clear').addEventListener('click', ()=>{
      state.pages = []; updateBadge(); renderPages();
    });
    document.getElementById('btn-generate').addEventListener('click', generatePdf);
  }

  // =================================================================
  // PDF GENERATION
  // =================================================================
  function generatePdf(){
    if(state.pages.length === 0) return;
    const { jsPDF } = window.jspdf;
    const first = state.pages[0];
    const orientation = first.w > first.h ? 'l' : 'p';
    const pdf = new jsPDF({ orientation, unit:'pt', format:'a4' });

    state.pages.forEach((p, i)=>{
      if(i>0) pdf.addPage(undefined, p.w > p.h ? 'l' : 'p');
      const pageW = pdf.internal.pageSize.getWidth();
      const pageH = pdf.internal.pageSize.getHeight();
      const imgRatio = p.w / p.h;
      const pageRatio = pageW / pageH;
      let drawW, drawH;
      if(imgRatio > pageRatio){ drawW = pageW; drawH = pageW / imgRatio; }
      else { drawH = pageH; drawW = pageH * imgRatio; }
      const x = (pageW - drawW)/2, y = (pageH - drawH)/2;
      pdf.addImage(p.dataUrl, 'JPEG', x, y, drawW, drawH);
    });

    const blob = pdf.output('blob');
    if(state.pdfBlobUrl) URL.revokeObjectURL(state.pdfBlobUrl);
    state.pdfBlobUrl = URL.createObjectURL(blob);

    document.getElementById('pdf-empty').style.display = 'none';
    document.getElementById('pdf-content').style.display = 'block';
    document.getElementById('pdf-frame').src = state.pdfBlobUrl;
    document.getElementById('pdf-name').textContent = 'documento-escaneado.pdf';
    document.getElementById('pdf-size').textContent = (blob.size/1024).toFixed(0) + ' KB · ' + state.pages.length + ' pág.';

    switchTab('pdf');
  }

  document.getElementById('btn-download').addEventListener('click', ()=>{
    if(!state.pdfBlobUrl) return;
    const a = document.createElement('a');
    a.href = state.pdfBlobUrl; a.download = 'documento-escaneado.pdf';
    document.body.appendChild(a); a.click(); a.remove();
  });
  document.getElementById('btn-newscan').addEventListener('click', ()=>{
    state.pages = []; updateBadge(); switchTab('camera');
  });

  // ---------- Init ----------
  updateBadge();
  startCamera();
  window.addEventListener('beforeunload', stopCamera);
})();
</script>

</body>
</html>
