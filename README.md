# RPG-grid
<script>
  let currentGridName = "grid1";
  let tokens = [];
  let rulerEnabled = true; // régua ligada por padrão

  // cria réguas (coluna / linha)
  function createRulers() {
    const h = document.getElementById("rulerH");
    const v = document.getElementById("rulerV");
    const cols = 20;
    const rows = 15;

    h.innerHTML = "";
    for (let i = 0; i < cols; i++) {
      const span = document.createElement("span");
      span.textContent = i;
      h.appendChild(span);
    }

    v.innerHTML = "";
    for (let i = 0; i < rows; i++) {
      const span = document.createElement("span");
      span.textContent = i;
      v.appendChild(span);
    }
  }

  // cria o grid
  function createGrid() {
    const grid = document.getElementById("grid");
    grid.innerHTML = "";
    const cols = 20;
    const rows = 15;

    for (let y = 0; y < rows; y++) {
      for (let x = 0; x < cols; x++) {
        const cell = document.createElement("div");
        cell.classList.add("cell");
        cell.dataset.x = x;
        cell.dataset.y = y;
        grid.appendChild(cell);
      }
    }
  }

  // lista grids salvos
  function loadGridList() {
    const select = document.getElementById("gridList");
    const saved = localStorage.getItem("rpg-grids");
    const grids = saved ? JSON.parse(saved) : {};

    select.innerHTML = "";
    Object.keys(grids).forEach(name => {
      const opt = document.createElement("option");
      opt.value = name;
      opt.textContent = name;
      select.appendChild(opt);
    });
  }

  // salva grid
  function saveGrid() {
    const saved = localStorage.getItem("rpg-grids");
    const grids = saved ? JSON.parse(saved) : {};

    grids[currentGridName] = {
      tokens: tokens.map(t => ({ id: t.id, x: t.x, y: t.y, src: t.src })),
    };

    localStorage.setItem("rpg-grids", JSON.stringify(grids));
    document.getElementById("message").textContent = `Grid "${currentGridName}" salvo.`;
    setTimeout(() => document.getElementById("message").textContent = "", 2000);

    loadGridList();
  }

  // carrega grid
  function loadGrid(name) {
    const saved = localStorage.getItem("rpg-grids");
    const grids = saved ? JSON.parse(saved) : {};
    const grid = grids[name];

    if (!grid) return;

    document.getElementById("grid").innerHTML = "";
    createGrid(); // redesenha

    tokens = [];

    grid.tokens.forEach(data => {
      const token = createToken(data.src, data.x, data.y);
      token.style.left = data.x * 40 + 5 + "px";
      token.style.top  = data.y * 40 + 5 + "px";
      tokens.push(token);
    });

    currentGridName = name;
    document.getElementById("message").textContent = `Grid "${name}" carregado.`;
    setTimeout(() => document.getElementById("message").textContent = "", 2000);
  }

  // cria token com medição
  function createToken(src, x = 0, y = 0) {
    const token = document.createElement("img");
    token.classList.add("token");
    token.src = src;

    token.style.left = x * 40 + 5 + "px";
    token.style.top  = y * 40 + 5 + "px";

    let isDragging = false;
    let dragStartX, dragStartY;

    // mouse
    token.addEventListener("mousedown", e => {
      if (e.button === 0) startDrag(e.clientX, e.clientY);
      e.preventDefault();
    });
    // touch
    token.addEventListener("touchstart", e => {
      if (e.touches.length === 1) {
        const t = e.touches[0];
        startDrag(t.clientX, t.clientY);
      }
    });

    function startDrag(clientX, clientY) {
      isDragging = true;
      const rect = document.getElementById("gridContainer").getBoundingClientRect();
      dragStartX = token.offsetLeft;
      dragStartY = token.offsetTop;

      updateRulerDistance(0); // mostra 0 no início
    }

    function moveDrag(clientX, clientY) {
      if (!isDragging || !rulerEnabled) return;
      const rect = document.getElementById("gridContainer").getBoundingClientRect();
      const left = clientX - rect.left - dragOffsetX;
      const top  = clientY - rect.top - dragOffsetY;

      token.style.left = left + "px";
      token.style.top  = top + "px";

      // diferença em pixels
      const dX = (left - dragStartX);
      const dY = (top - dragStartY);

      // 1 quadrado = 40px -> 1,5m
      const dxQuad = Math.abs(dX / 40);
      const dyQuad = Math.abs(dY / 40);

      // regra de diagonal: cada “diagonal” (mesmo tempo em X e Y) vale 2 quadrados
      const diag = Math.min(dxQuad, dyQuad);
      const orthoX = dxQuad - diag;
      const orthoY = dyQuad - diag;

      const distQuad = orthoX + orthoY + diag * 2; // diagonal = 2 quadrados
      const distM = distQuad * 1.5; // 1 quadrado = 1,5m

      updateRulerDistance(distM);
    }

    function endDrag() {
      if (!isDragging) return;
      isDragging = false;

      const rect = document.getElementById("gridContainer").getBoundingClientRect();
      const x = Math.floor((token.offsetLeft - 5) / 40);
      const y = Math.floor((token.offsetTop - 5) / 40);

      const cols = 20;
      const rows = 15;
      const xClamp = Math.max(0, Math.min(x, cols - 1));
      const yClamp = Math.max(0, Math.min(y, rows - 1));

      token.style.left = xClamp * 40 + 5 + "px";
      token.style.top  = yClamp * 40 + 5 + "px";

      const existing = tokens.find(t => t === token);
      if (existing) {
        existing.x = xClamp;
        existing.y = yClamp;
      }

      document.getElementById("message").textContent = "";
    }

    function updateRulerDistance(distM) {
      const msg = document.getElementById("message");
      if (!rulerEnabled || distM === null) {
        msg.textContent = "";
      } else {
        msg.textContent = `Distância: ~${distM.toFixed(1)}m`;
      }
    }

    // drag offset (mouse/touch)
    let dragOffsetX, dragOffsetY;

    document.addEventListener("mousemove", e => {
      if (!isDragging) return;
      dragOffsetX = e.clientX - token.offsetLeft;
      dragOffsetY = e.clientY - token.offsetTop;
      moveDrag(e.clientX, e.clientY);
    });

    document.addEventListener("touchmove", e => {
      if (!isDragging) return;
      e.preventDefault();
      const t = e.touches[0];
      dragOffsetX = t.clientX - token.offsetLeft;
      dragOffsetY = t.clientY - token.offsetTop;
      moveDrag(t.clientX, t.clientY);
    });

    document.addEventListener("mouseup", endDrag);
    document.addEventListener("touchend", endDrag);

    document.getElementById("gridContainer").appendChild(token);
    return token;
  }

  // inicialização
  function init() {
    createRulers();
    createGrid();
    loadGridList();

    // botão novo grid
    document.getElementById("btnNewGrid").addEventListener("click", () => {
      const name = prompt("Nome do novo grid:", "grid" + Date.now());
      if (!name) return;
      currentGridName = name;
      tokens = [];
      createGrid();
      document.getElementById("message").textContent = `Novo grid "${name}" criado.`;
      setTimeout(() => document.getElementById("message").textContent = "", 2000);
      loadGridList();
    });

    // botão salvar grid
    document.getElementById("btnSaveGrid").addEventListener("click", saveGrid);

    // botão carregar grid
    document.getElementById("btnLoadGrid").addEventListener("click", () => {
      const select = document.getElementById("gridList");
      const name = select.value;
      if (!name) return;
      loadGrid(name);
    });

    // input de imagem
    document.getElementById("imageInput").addEventListener("change", e => {
      const file = e.target.files[0];
      if (!file) return;
      const url = URL.createObjectURL(file);
      const token = createToken(url, 0, 0);
      tokens.push(token);
    });

    // novo botão: ligar/desligar régua (coloque no HTML abaixo do "Salva Grid")
    const controls = document.querySelector(".controls");
    const btnRuler = document.createElement("button");
    btnRuler.textContent = "Régua: ON";
    btnRuler.addEventListener("click", () => {
      rulerEnabled = !rulerEnabled;
      btnRuler.textContent = "Régua: " + (rulerEnabled ? "ON" : "OFF");
      if (!rulerEnabled) document.getElementById("message").textContent = "";
    });
    controls.appendChild(btnRuler);
  }

  init();
</script>
