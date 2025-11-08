<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Bureau d'Ordre — Gestion du Courrier</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    .card{border-radius:16px;background:#fff;box-shadow:0 12px 36px rgba(0,0,0,.08);padding:16px;}
    .btn{display:inline-flex;align-items:center;gap:.5rem;border:1px solid #e5e7eb;border-radius:12px;padding:.6rem 1rem;font-weight:600;cursor:pointer;font-size:.9rem;}
    .btn-primary{background:#1e3a8a;color:#fff;border-color:#1e40af;}
    .btn-ghost{background:#f8fafc;}
    .btn-toggle{min-width:10rem;justify-content:center;border-width:2px;}
    .btn-active{background:#1d4ed8;color:#fff;border-color:#1e40af;}
    .btn-inactive{background:#fff;color:#1f2937;border-color:#93c5fd;}
    .input{width:100%;border:1px solid #e5e7eb;border-radius:12px;padding:.6rem .9rem;font-size:.9rem;}
    .input-lg{padding:.9rem .9rem;font-size:.95rem;}
    .label{font-size:.8rem;font-weight:600;color:#334155;text-transform:uppercase;}
    .tbl-th{text-align:left;font-size:.7rem;font-weight:700;color:#475569;text-transform:uppercase;padding:.4rem;}
    .tbl-td{font-size:.85rem;color:#111827;padding:.4rem;}
    @media print{
      .no-print{display:none!important}
      body{background:#fff}
      .card{box-shadow:none}
      header{position:static!important}
    }
  </style>
</head>
<body class="bg-gray-50 text-gray-900">
<div class="max-w-7xl mx-auto p-4">
  <!-- ENTÊTE -->
  <header class="no-print sticky top-0 z-10 bg-white/95 backdrop-blur border rounded-lg">
    <div class="px-4 py-3 flex items-center justify-between">
      <div class="flex items-center gap-3">
        <div class="h-12 w-12 rounded-md border bg-blue-700"></div>
        <div>
          <div class="text-sm text-gray-500">Conseil Préfectoral Skhirate Témara</div>
          <div class="text-lg font-bold">Bureau d'Ordre — Gestion du Courrier</div>
        </div>
      </div>
      <div class="flex items-center gap-2">
        <input id="fileJson" type="file" accept=".json,application/json" class="hidden">
        <button class="btn" id="btnImport">Importer JSON</button>
        <button class="btn" id="btnExportPDF">Exporter PDF</button>
        <button class="btn btn-primary" id="btnNew">Nouveau courrier</button>
      </div>
    </div>
  </header>

  <!-- FILTRES -->
  <section class="my-4 p-3 card no-print">
    <div class="grid grid-cols-1 md:grid-cols-6 gap-3 items-end">
      <div class="md:col-span-2">
        <label class="label">Recherche</label>
        <input id="search" class="input" placeholder="Objet, n°, expéditeur, référence…">
      </div>
      <div>
        <label class="label">Type</label>
        <select id="filterType" class="input">
          <option value="">Tous</option>
          <option value="entrant">Entrant</option>
          <option value="sortant">Sortant</option>
        </select>
      </div>
      <div class="md:col-span-3 text-right text-xs text-gray-500">
        Données stockées dans ce navigateur (localStorage).
      </div>
    </div>
  </section>

  <!-- STATS -->
  <section id="stats" class="grid grid-cols-2 md:grid-cols-6 gap-3 mb-4 no-print"></section>

  <!-- TABLEAU -->
  <section class="card">
    <table class="min-w-full" id="tbl">
      <thead>
        <tr>
          <th class="tbl-th">Type</th>
          <th class="tbl-th">N°</th>
          <th class="tbl-th">Date</th>
          <th class="tbl-th">Expéditeur / Destinataire</th>
          <th class="tbl-th">Objet</th>
          <th class="tbl-th">Service</th>
          <th class="tbl-th">Priorité</th>
          <th class="tbl-th">Échéance</th>
          <th class="tbl-th">Statut</th>
          <th class="tbl-th no-print">Pièces</th>
          <th class="tbl-th no-print">Actions</th>
        </tr>
      </thead>
      <tbody id="tbody" class="divide-y"></tbody>
    </table>
  </section>
</div>

<script>
  // --- Base ---
  var KEY = "bo_courrier_v1";
  var SERVICES = [
    "Secrétariat",
    "Direction des Affaires de la Présidence",
    "Direction Générale des Services",
    "Service Mutualisation Coopération et Société Civile",
    "Service Gestion des Ressources humaines et Affaires Juridiques",
    "Service Des Affaires Financières et Patrimoine",
    "Service des Affaires Techniques et Développement et des Équipements",
    "Bureau de Gestion du Parc",
    "Bureau de Gestion des Archives",
    "Bureau d'information et Numérisation"
  ];
  function $(s,p){return (p||document).querySelector(s);}
  var ROWS = [];
  try { ROWS = JSON.parse(localStorage.getItem(KEY) || "[]") || []; } catch(e){ ROWS=[]; }
  function save(){ localStorage.setItem(KEY, JSON.stringify(ROWS)); }

  // --- Utils ---
  function dangerDate(e){
    if(!e) return "";
    var d = new Date(e), n = new Date();
    var diff = (d - n) / (1000*3600*24);
    if(diff < 0) return "bg-red-50";
    if(diff <= 3) return "bg-yellow-50";
    return "";
  }

  // --- Stats ---
  function renderStats(){
    var r = ROWS;
    var t = r.length;
    var e = r.filter(x=>x.type==="entrant").length;
    var s = r.filter(x=>x.type==="sortant").length;
    var enT = r.filter(x=>x.statut==="En traitement").length;
    var urg = r.filter(x=>x.priorite && x.priorite!=="Normal").length;
    var late = r.filter(x=>x.echeance && new Date(x.echeance) < new Date()).length;
    function c(ti,v){return '<div class="card"><div class="text-xs text-gray-500">'+ti+'</div><div class="text-2xl font-bold">'+v+'</div></div>';}
    $("#stats").innerHTML = c("Total",t)+c("Entrants",e)+c("Sortants",s)+c("En traitement",enT)+c("Urgents",urg)+c("Échéances dépassées",late);
  }

  // --- Tableau ---
  function renderTable(){
    var tb = $("#tbody");
    var fText = ($("#search").value || "").toLowerCase();
    var fType = $("#filterType").value || "";
    var rows = ROWS.slice();

    if(fType) rows = rows.filter(r=>r.type===fType);
    if(fText){
      rows = rows.filter(function(r){
        return (r.objet||"").toLowerCase().includes(fText) ||
               (r.numero||"").toLowerCase().includes(fText) ||
               (r.expediteur||"").toLowerCase().includes(fText) ||
               (r.destinataire||"").toLowerCase().includes(fText) ||
               (r.reference||"").toLowerCase().includes(fText);
      });
    }

    tb.innerHTML = "";
    if(rows.length === 0){
      var tr = document.createElement("tr");
      var td = document.createElement("td");
      td.colSpan = 11;
      td.className = "p-8 text-center text-sm text-gray-500";
      td.textContent = "Aucun courrier pour l’instant.";
      tr.appendChild(td);
      tb.appendChild(tr);
      return;
    }

    rows.forEach(function(r){
      var tr = document.createElement("tr");
      tr.className = dangerDate(r.echeance);

      function add(v){
        var td = document.createElement("td");
        td.className = "tbl-td";
        td.textContent = v || "";
        tr.appendChild(td);
      }

      add(r.type==="entrant"?"Entrant":"Sortant");
      add(r.numero);
      add(r.date);
      add(r.type==="entrant" ? (r.expediteur||"-") : (r.destinataire||"-"));
      add(r.objet);
      add(r.service);
      add(r.priorite);
      add(r.echeance || "-");
      add(r.statut);

      // Pièces
      var tdP = document.createElement("td");
      tdP.className = "tbl-td no-print";
      var hasPieces = r.attachments && r.attachments.length;
      if(hasPieces){
        var a = document.createElement("a");
        a.href = "#";
        a.textContent = "Voir";
        a.className = "text-blue-600 underline";
        a.onclick = function(ev){ev.preventDefault(); viewPieces(r);};
        tdP.appendChild(a);
      } else {
        tdP.textContent = "-";
        tdP.classList.add("text-gray-400");
      }
      tr.appendChild(tdP);

      // Actions
      var tdA = document.createElement("td");
      tdA.className = "tbl-td no-print";
      var aEdit = document.createElement("a");
      aEdit.href="#";
      aEdit.textContent="Modifier";
      aEdit.className="text-blue-600 underline mr-3";
      aEdit.onclick=function(ev){ev.preventDefault(); openForm(r);};
      var aDel = document.createElement("a");
      aDel.href="#";
      aDel.textContent="Supprimer";
      aDel.className="text-red-600 underline";
      aDel.onclick=function(ev){
        ev.preventDefault();
        if(confirm("Supprimer ce courrier ?")){
          ROWS = ROWS.filter(x=>x.id!==r.id);
          save(); renderAll();
        }
      };
      tdA.appendChild(aEdit);
      tdA.appendChild(aDel);
      tr.appendChild(tdA);

      tb.appendChild(tr);
    });
  }

  // --- Modal générique ---
  function openModal(title, contentEl){
    var old = document.getElementById("__modal");
    if(old) old.remove();
    var modal = document.createElement("div");
    modal.id = "__modal";
    modal.className = "fixed inset-0 z-50";
    modal.innerHTML =
      '<div class="absolute inset-0" style="background:#1f3b82;opacity:.9;"></div>'+
      '<div class="absolute inset-0 flex items-center justify-center p-4">'+
        '<div class="card w-full max-w-[95vw] max-h-[90vh] overflow-auto" style="background:#eaf1ff;">'+
          '<div class="flex items-center justify-between gap-3 mb-2">'+
            '<h3 class="text-lg font-semibold">'+title+'</h3>'+
            '<button class="btn btn-ghost" id="btnCloseModal">Fermer</button>'+
          '</div>'+
          '<div id="modalContent"></div>'+
        '</div>'+
      '</div>';
    document.body.appendChild(modal);
    var c = document.getElementById("modalContent");
    c.appendChild(contentEl);
    document.getElementById("btnCloseModal").onclick = function(){ modal.remove(); };
    modal.firstChild.onclick = function(){ modal.remove(); };
  }

  // --- Voir pièces ---
  function viewPieces(r){
    var wrap = document.createElement("div");
    if(r.attachments){
      r.attachments.forEach(function(a,idx){
        var label = document.createElement("div");
        label.className = "mb-1 text-sm font-semibold";
        label.textContent = (idx+1)+". "+(a.name||"Pièce jointe");
        wrap.appendChild(label);

        var viewer = document.createElement("div");
        viewer.className = "mb-4 border rounded-lg overflow-hidden bg-white";
        var src = a.dataUrl || a.url || "";
        if(a.mime && a.mime.indexOf("image/")===0){
          var img = document.createElement("img");
          img.src = src;
          img.className = "w-full h-auto";
          viewer.appendChild(img);
        } else if((a.mime==="application/pdf") || (a.name||"").toLowerCase().endsWith(".pdf")){
          var emb = document.createElement("embed");
          emb.src = src;
          emb.type = "application/pdf";
          emb.style.width="100%";
          emb.style.height="480px";
          viewer.appendChild(emb);
        } else if(src){
          var ifr = document.createElement("iframe");
          ifr.src = src;
          ifr.style.width="100%";
          ifr.style.height="480px";
          viewer.appendChild(ifr);
        } else {
          var span = document.createElement("div");
          span.className="p-2 text-xs text-gray-500";
          span.textContent="Pièce jointe non prévisualisable.";
          viewer.appendChild(span);
        }
        wrap.appendChild(viewer);
      });
    }
    openModal("Pièces jointes", wrap);
  }

  // --- Formulaire Nouveau / Modifier ---
  function openForm(initial){
    var f = initial ? JSON.parse(JSON.stringify(initial)) : {
      id: "c"+Date.now(),
      type: "entrant",
      numero: "",
      date: new Date().toISOString().slice(0,10),
      expediteur: "",
      destinataire: "",
      objet: "",
      instructions: "",
      service: SERVICES[0],
      mode: "Courrier",
      priorite: "Normal",
      echeance: "",
      statut: "Reçu",
      observation: "",
      reference: "",
      attachments: []
    };

    var grid = document.createElement("div");
    grid.className = "grid grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-3";

    // Type
    var typeBox = document.createElement("div");
    typeBox.className = "lg:col-span-4 md:col-span-3 border-2 border-blue-300 bg-blue-50 p-3 rounded-lg";
    typeBox.innerHTML =
      '<div class="font-semibold text-blue-800 mb-2">Type de courrier</div>'+
      '<div class="flex flex-wrap gap-3">'+
      '<button type="button" id="btnEntrant" class="btn btn-toggle">Entrant (Arrivée)</button>'+
      '<button type="button" id="btnSortant" class="btn btn-toggle">Sortant (Départ)</button>'+
      '</div>';
    grid.appendChild(typeBox);

    // Champs
    grid.innerHTML += `
      <div><div class="label">N° d'enregistrement</div><input id="f_numero" class="input" /></div>
      <div><div class="label">Date</div><input id="f_date" type="date" class="input" /></div>
      <div><div class="label" id="lblPerson">Expéditeur</div><input id="f_person" class="input" /></div>
      <div>
        <div class="label" id="lblService">Service destinataire</div>
        <select id="f_service" class="input">
          ${SERVICES.map(function(s){return '<option>'+s+'</option>';}).join("")}
        </select>
      </div>
      <div>
        <div class="label" id="lblMode">Mode de réception</div>
        <select id="f_mode" class="input">
          <option>Courrier</option>
          <option>Bureau d'Ordre</option>
          <option>Mail</option>
          <option>Plateforme (e-Parapheur, Majaliss...)</option>
          <option>Main propre</option>
        </select>
      </div>
      <div>
        <div class="label">Priorité</div>
        <select id="f_priorite" class="input">
          <option>Normal</option>
          <option>Urgent</option>
          <option>Très urgent</option>
        </select>
      </div>
      <div>
        <div class="label">Échéance</div>
        <input id="f_echeance" type="date" class="input" />
      </div>
      <div>
        <div class="label">Statut</div>
        <select id="f_statut" class="input">
          <option>Reçu</option>
          <option>Enregistré</option>
          <option>Distribué</option>
          <option>En traitement</option>
          <option>Répondu</option>
          <option>Clos</option>
        </select>
      </div>
      <div class="lg:col-span-4 md:col-span-3">
        <div class="label">Objet</div>
        <textarea id="f_objet" class="input input-lg" rows="3"></textarea>
      </div>
      <div class="lg:col-span-4 md:col-span-3">
        <div class="label">Instructions</div>
        <textarea id="f_instructions" class="input" rows="3"></textarea>
      </div>
      <div>
        <div class="label">Référence</div>
        <input id="f_reference" class="input" />
      </div>
      <div>
        <div class="label">Observations</div>
        <input id="f_obs" class="input" />
      </div>
      <div class="lg:col-span-4 md:col-span-3">
        <div class="label">Pièces jointes (PDF / Images)</div>
        <input id="f_files" type="file" class="input" multiple accept=".pdf,image/*" />
        <div id="filesList" class="mt-2 flex flex-wrap gap-2 text-xs"></div>
      </div>
    `;

    // Boutons bas
    var actions = document.createElement("div");
    actions.className = "lg:col-span-4 md:col-span-3 flex justify-end gap-2 mt-3";
    actions.innerHTML =
      '<button type="button" class="btn" id="btnCancel">Annuler</button>'+
      '<button type="button" class="btn btn-primary" id="btnSave">Enregistrer</button>';
    grid.appendChild(actions);

    // Remplir formulaire avec f
    function syncToForm(){
      $("#f_numero",grid).value = f.numero || "";
      $("#f_date",grid).value = f.date || "";
      $("#f_person",grid).value = (f.type==="entrant" ? f.expediteur : f.destinataire) || "";
      $("#f_objet",grid).value = f.objet || "";
      $("#f_instructions",grid).value = f.instructions || "";
      $("#f_service",grid).value = f.service || SERVICES[0];
      $("#f_mode",grid).value = f.mode || "Courrier";
      $("#f_priorite",grid).value = f.priorite || "Normal";
      $("#f_echeance",grid).value = f.echeance || "";
      $("#f_statut",grid).value = f.statut || "Reçu";
      $("#f_reference",grid).value = f.reference || "";
      $("#f_obs",grid).value = f.observation || "";
      $("#lblPerson",grid).textContent = (f.type==="entrant"?"Expéditeur":"Destinataire");
      $("#lblService",grid).textContent = (f.type==="entrant"?"Service destinataire":"Service expéditeur");
      $("#lblMode",grid).textContent = (f.type==="entrant"?"Mode de réception":"Mode d'envoi");
      var btnE = $("#btnEntrant",grid), btnS = $("#btnSortant",grid);
      btnE.classList.remove("btn-active","btn-inactive");
      btnS.classList.remove("btn-active","btn-inactive");
      if(f.type==="entrant"){btnE.classList.add("btn-active");btnS.classList.add("btn-inactive");}
      else{btnS.classList.add("btn-active");btnE.classList.add("btn-inactive");}
      renderFilesList();
    }

    function renderFilesList(){
      var c = $("#filesList",grid);
      c.innerHTML = "";
      if(!f.attachments) f.attachments = [];
      f.attachments.forEach(function(a,idx){
        var span = document.createElement("span");
        span.className = "inline-flex items-center gap-1 border px-2 py-1 rounded bg-white";
        span.textContent = a.name || ("Pièce "+(idx+1));
        var x = document.createElement("button");
        x.type="button"; x.textContent="×";
        x.onclick=function(){ f.attachments.splice(idx,1); renderFilesList(); };
        span.appendChild(x);
        c.appendChild(span);
      });
    }

    // Gestion des champs
    grid.addEventListener("input", function(e){
      var id = e.target.id, v = e.target.value;
      if(id==="f_numero") f.numero=v;
      if(id==="f_date") f.date=v;
      if(id==="f_objet") f.objet=v;
      if(id==="f_instructions") f.instructions=v;
      if(id==="f_reference") f.reference=v;
      if(id==="f_obs") f.observation=v;
      if(id==="f_person"){
        if(f.type==="entrant") f.expediteur=v;
        else f.destinataire=v;
      }
      if(id==="f_echeance") f.echeance=v;
    });
    grid.addEventListener("change", function(e){
      var id = e.target.id, v = e.target.value;
      if(id==="f_service") f.service=v;
      if(id==="f_mode") f.mode=v;
      if(id==="f_priorite") f.priorite=v;
      if(id==="f_statut") f.statut=v;
    });

    // Type boutons
    grid.addEventListener("click", function(e){
      if(e.target.id==="btnEntrant"){
        f.type="entrant"; syncToForm();
      }
      if(e.target.id==="btnSortant"){
        f.type="sortant"; syncToForm();
      }
    });

    // Upload pièces -> base64
    grid.querySelector("#f_files").addEventListener("change", function(e){
      var files = Array.prototype.slice.call(e.target.files || []);
      if(!files.length) return;
      var idx = 0;
      function readNext(){
        var file = files[idx++];
        if(!file){ renderFilesList(); return; }
        var reader = new FileReader();
        reader.onload = function(ev){
          f.attachments.push({
            name: file.name,
            mime: file.type,
            dataUrl: ev.target.result
          });
          readNext();
        };
        reader.readAsDataURL(file);
      }
      readNext();
      e.target.value = "";
    });

    // Actions
    grid.querySelector("#btnCancel").onclick = function(){
      var m = document.getElementById("__modal"); if(m) m.remove();
    };
    grid.querySelector("#btnSave").onclick = function(){
      // simple contrôle minimal
      if(!f.objet){
        alert("Veuillez saisir l'objet du courrier."); return;
      }
      var i = ROWS.findIndex(function(x){return x.id===f.id;});
      if(i>=0) ROWS[i]=f; else ROWS.unshift(f);
      save();
      var m = document.getElementById("__modal"); if(m) m.remove();
      renderAll();
    };

    syncToForm();
    openModal(initial ? "Modifier le courrier" : "Nouveau courrier", grid);
  }

  // --- Import JSON ---
  function handleImportFile(file){
    var reader = new FileReader();
    reader.onload = function(){
      try{
        var data = JSON.parse(reader.result);
        if(!Array.isArray(data)) throw new Error("Le JSON doit contenir un tableau.");
        var before = ROWS.length;
        var ids = {};
        ROWS.forEach(function(x){ ids[x.id] = true; });
        data.forEach(function(x,i){
          if(!x || !x.id) x.id = "imp_"+Date.now()+"_"+i;
          if(!ids[x.id]){ ROWS.push(x); ids[x.id]=true; }
        });
        save();
        renderAll();
        alert("Import terminé : " + (ROWS.length-before) + " lignes ajoutées.");
      }catch(err){
        alert("Erreur lors de la lecture du JSON : " + err.message);
      }
    };
    reader.readAsText(file, "utf-8");
  }

  // --- Boutons globaux ---
  document.getElementById("btnNew").onclick = function(){ openForm(null); };
  document.getElementById("btnImport").onclick = function(){ document.getElementById("fileJson").click(); };
  document.getElementById("fileJson").addEventListener("change", function(e){
    var f = e.target.files[0];
    if(f) handleImportFile(f);
    e.target.value = "";
  });
  document.getElementById("btnExportPDF").onclick = function(){ window.print(); };
  document.getElementById("search").oninput = renderAll;
  document.getElementById("filterType").onchange = renderAll;

  function renderAll(){ renderStats(); renderTable(); }
  renderAll();
</script>
</body>
</html>
