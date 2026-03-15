# Setup — Gym Tracker + Dieta → Google Sheets

## Passo a passo

### 1. Subir planilha
- Upload `Paulo_Kamimura_v3.xlsx` no Google Drive
- Abra com Google Sheets (já tem a aba Config)

### 2. Colar o Apps Script
- Na planilha: **Extensões → Apps Script**
- Apague tudo, cole o código abaixo
- Salve (💾)

### 3. Criar aba Dieta (se não existir)
- Rode **`criarAbaDieta`** → cria aba com plano alimentar
- A tabela de descanso/cadência já está dentro de cada aba A, B, C, D

### 4. Testar
- Selecione **`testeGet`** e rode
- Veja o log (View → Logs). Deve mostrar exercícios encontrados

### 5. Deploy
- **Implantar → Nova implantação**
- Tipo: **App da Web**
- Executar como: **Eu**
- Quem tem acesso: **Qualquer pessoa**
- Clique **Implantar**, copie a URL

### 6. Configurar no app
- Abra PauloApp.html
- Cole a URL, toque **Conectar**

### ⚠️ Após mudanças no código
**Implantar → Gerenciar implantações → ✏️ → Nova versão → Implantar**

---

## Código — cole TUDO no Apps Script

```javascript
function doGet(e) {
  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var result = { workouts: {}, diet: null };
    var names = ['A','B','C','D'];
    for (var n = 0; n < names.length; n++) {
      var sh = ss.getSheetByName(names[n]);
      if (sh) result.workouts[names[n]] = parseWorkout(sh);
    }
    var dsh = ss.getSheetByName('Dieta');
    if (dsh) result.diet = parseDiet(dsh);
    var json = JSON.stringify(result);
    // JSONP support: if callback param exists, wrap in function call
    var callback = e && e.parameter && e.parameter.callback;
    if (callback) {
      return ContentService.createTextOutput(callback + '(' + json + ')')
        .setMimeType(ContentService.MimeType.JAVASCRIPT);
    }
    return ContentService.createTextOutput(json).setMimeType(ContentService.MimeType.JSON);
  } catch(err) {
    var errJson = JSON.stringify({error: err.message});
    var cb = e && e.parameter && e.parameter.callback;
    if (cb) {
      return ContentService.createTextOutput(cb + '(' + errJson + ')')
        .setMimeType(ContentService.MimeType.JAVASCRIPT);
    }
    return ContentService.createTextOutput(errJson).setMimeType(ContentService.MimeType.JSON);
  }
}

function doPost(e) {
  try {
    var body = '';
    if (e.postData) body = e.postData.contents || '';
    if (!body && e.parameter && e.parameter.data) body = e.parameter.data;
    if (!body) return jsonOut({success:false, error:'Corpo vazio'});
    var data = JSON.parse(body);
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var sheet = ss.getSheetByName(data.workout);
    if (!sheet) return jsonOut({success:false, error:'Aba nao encontrada: '+data.workout});
    var col = findNextCol(sheet, data);
    if (col > 30) return jsonOut({success:false, error:'Planilha cheia'});
    var count = 0;
    var exercises = data.exercises || [];
    for (var i = 0; i < exercises.length; i++) {
      var sets = exercises[i].sets || [];
      for (var j = 0; j < sets.length; j++) {
        var s = sets[j];
        if (s.w && s.r && s.row) {
          sheet.getRange(s.row, col).setValue(s.w);
          sheet.getRange(s.row, col + 1).setValue(s.r);
          count++;
        }
      }
    }
    return jsonOut({success:true, count:count, column:colToLetter(col)});
  } catch(err) {
    return jsonOut({success:false, error:err.message});
  }
}

function jsonOut(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj)).setMimeType(ContentService.MimeType.JSON);
}

function parseWorkout(sheet) {
  var data = sheet.getDataRange().getValues();
  var result = {title: String(data[0][0] || ''), mobility: [], exercises: [], rest: [], cadence: ''};
  var i = 0;

  // Mobility: find items with "NxNN" pattern in column B
  for (i = 0; i < data.length; i++) {
    var a = String(data[i][0] || '').trim();
    var b = String(data[i][1] || '').trim();
    if (a && b && b.match(/^\d+x\d+/)) {
      result.mobility.push(a + ' ' + b);
    }
  }

  // Rest table: find "Descanso" in col A, then read rest rows below
  for (i = 0; i < data.length; i++) {
    var a = String(data[i][0] || '').trim();
    if (a === 'Descanso') {
      // Cadence is in col D of the "Descanso" row or the next row
      var cad = String(data[i][3] || '').trim();
      if (cad && cad !== 'Cadência' && cad !== 'Cadencia') result.cadence = cad;
      // Next row might have "REPS" header with cadence in D
      if (i + 1 < data.length) {
        var nextA = String(data[i+1][0] || '').trim();
        if (nextA === 'REPS') {
          var cadVal = String(data[i+1][3] || '').trim();
          if (cadVal) result.cadence = cadVal;
          i += 2; // skip header row, start reading data
        } else {
          i += 1;
        }
      }
      // Read rest rows until empty
      while (i < data.length) {
        var repLabel = String(data[i][0] || '').trim();
        if (!repLabel) break;
        var secStr = String(data[i][2] || '').trim();
        // Parse rest seconds: could be "40", "40-70", "90-120", "120+"
        var restMin = 60, restMax = 60;
        if (secStr.indexOf('-') >= 0) {
          var parts = secStr.split('-');
          restMin = parseInt(parts[0]) || 60;
          restMax = parseInt(parts[1]) || restMin;
        } else if (secStr.indexOf('+') >= 0) {
          restMin = parseInt(secStr) || 120;
          restMax = restMin + 60;
        } else {
          restMin = parseInt(secStr) || 60;
          restMax = restMin;
        }
        // Parse rep range from label
        var repsMin = 1, repsMax = 99;
        if (repLabel.indexOf('+') >= 0) {
          repsMin = parseInt(repLabel) || 20;
          repsMax = 99;
        } else if (repLabel.indexOf('menos') >= 0 || repLabel.indexOf('<') >= 0) {
          repsMin = 1;
          repsMax = parseInt(repLabel.replace(/\D/g, '')) || 8;
        } else if (repLabel.indexOf(' a ') >= 0) {
          var rp = repLabel.split(' a ');
          repsMin = parseInt(rp[0]) || 1;
          repsMax = parseInt(rp[1]) || 99;
        }
        result.rest.push({ label: repLabel, repsMin: repsMin, repsMax: repsMax, rest: restMin, restMax: restMax });
        i++;
      }
      break; // found rest table, stop
    }
  }

  // Exercises: find rows where NEXT row has "Reps Alvo" in col B
  for (i = 0; i < data.length - 2; i++) {
    var nextB = String(data[i+1] ? data[i+1][1] || '' : '').trim();
    if (nextB === 'Reps Alvo') {
      var exName = String(data[i][0] || '').trim();
      if (!exName) continue;
      var exercise = {name: exName, sets: []};
      // Skip: name(i), header(i+1), sub-header(i+2) → sets start at i+3
      for (var r = i + 3; r < data.length; r++) {
        var setType = String(data[r][0] || '').trim();
        if (setType === 'VOLUME' || setType === '') break;
        var setObj = {
          type: setType,
          row: r + 1,
          targetReps: String(data[r][1] || ''),
          effort: String(data[r][2] || ''),
          obs: String(data[r][3] || ''),
          history: []
        };
        for (var c = 4; c < data[r].length - 1; c += 2) {
          var w = data[r][c];
          var rr = data[r][c+1];
          if (w !== '' && w !== null && w !== undefined && !isNaN(Number(w)) && Number(w) > 0) {
            setObj.history.push({w: Number(w), r: Number(rr) || 0});
          }
        }
        exercise.sets.push(setObj);
      }
      if (exercise.sets.length > 0) result.exercises.push(exercise);
    }
  }
  return result;
}

function parseDiet(sheet) {
  var data = sheet.getDataRange().getValues();
  var diet = {macros:{}, meals:[], fixedItems:[], ali:{}, frutas:[], tips:[]};
  var section = '', group = '';
  for (var i = 0; i < data.length; i++) {
    var a = String(data[i][0] || '').trim();
    if (a === 'MACROS')    {section='macros'; continue;}
    if (a === 'REFEICOES') {section='meals'; continue;}
    if (a === 'ITENS_FIXOS'){section='fixed'; continue;}
    if (a === 'FRUTAS')    {section='frutas'; continue;}
    if (a === 'DICAS')     {section='tips'; continue;}
    if (a.indexOf('ALIMENTO_') === 0) {
      section='ali'; group=a.replace('ALIMENTO_','');
      diet.ali[group]={title:'',mult:'',emoji:'',items:[]};
      continue;
    }
    if (!a) continue;
    if (a==='chave'||a==='num'||a==='ref'||a==='nome') continue;
    if (a==='titulo' && section==='ali') {
      diet.ali[group].title=String(data[i][1]||'');
      diet.ali[group].mult=String(data[i][2]||'');
      diet.ali[group].emoji=String(data[i][3]||'');
      continue;
    }
    if (section==='macros') diet.macros[a]=String(data[i][1]||'');
    else if (section==='meals') diet.meals.push({num:Number(data[i][0])||0,name:String(data[i][1]||''),kcal:Number(data[i][2])||0,p:Number(data[i][3])||0,g:Number(data[i][4])||0,c:Number(data[i][5])||0,type:String(data[i][6]||'fixo')});
    else if (section==='fixed') diet.fixedItems.push({ref:Number(data[i][0])||0,item:String(data[i][1]||''),qty:String(data[i][2]||''),note:String(data[i][3]||'')});
    else if (section==='ali') diet.ali[group].items.push({name:String(data[i][0]||''),qty:String(data[i][1]||''),note:String(data[i][2]||'')});
    else if (section==='frutas') diet.frutas.push({name:String(data[i][0]||''),qty:String(data[i][1]||'')});
    else if (section==='tips') diet.tips.push(a);
  }
  return diet;
}

function findNextCol(sheet, data) {
  var exercises = data.exercises || [];
  var firstRow = 0;
  for (var i = 0; i < exercises.length && !firstRow; i++) {
    var sets = exercises[i].sets || [];
    for (var j = 0; j < sets.length; j++) {
      if (sets[j].row) {firstRow = sets[j].row; break;}
    }
  }
  if (!firstRow) return 5;
  var vals = sheet.getRange(firstRow, 5, 1, 26).getValues()[0];
  for (var c = 0; c < vals.length - 1; c += 2) {
    if (vals[c] === '' || vals[c] === null || vals[c] === undefined) return c + 5;
  }
  return 31;
}

function colToLetter(col) {
  var s = '';
  while (col > 0) {
    var m = (col - 1) % 26;
    s = String.fromCharCode(65 + m) + s;
    col = Math.floor((col - m) / 26);
  }
  return s;
}

function criarAbaDieta() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sh = ss.getSheetByName('Dieta');
  if (sh) {
    var ui = SpreadsheetApp.getUi();
    if (ui.alert('Aba "Dieta" existe. Sobrescrever?', ui.ButtonSet.YES_NO) !== ui.Button.YES) return;
    ss.deleteSheet(sh);
  }
  sh = ss.insertSheet('Dieta');
  var rows = [
    ['MACROS','','','','','',''],['chave','valor','','','','',''],
    ['kcal_dia','1690','','','','',''],['agua','3-4L/dia','','','','',''],
    ['livres_sem','1500','','','','',''],['proteina','160','','','','',''],
    ['gordura','45','','','','',''],['carboidrato','160','','','','',''],
    ['','','','','','',''],
    ['REFEICOES','','','','','',''],['num','nome','kcal','P','G','C','tipo'],
    [1,'Café da Manhã',340,33,11,27,'fixo'],[2,'Almoço',500,53,11,51,'montavel'],
    [3,'Ceia',254,17,10,24,'fixo'],[4,'Jantar',500,53,11,51,'montavel'],
    ['','','','','','',''],
    ['ITENS_FIXOS','','','','','',''],['ref','item','qtd','nota','','',''],
    [1,'Queijo branco','40g','','','',''],
    [1,'Fruta (1 porção)','ver tabela','','','',''],
    [1,'Whey concentrado','30g','','','',''],
    [1,'Iogurte natural integral','~85g (½ pote)','','','',''],
    [3,'Pão','~65g (2 fatias)','','','',''],
    [3,'Queijo branco ou mussarela','20g','','','',''],
    [3,'Requeijão light / creme ricota','30g','','','',''],
    ['','','','','','',''],
    ['ALIMENTO_A','','','','','',''],
    ['titulo','Carboidrato','1,5x','🍚','','',''],['nome','qtd','nota','','','',''],
    ['Arroz cozido','150g','','','','',''],['Macarrão cozido','150g','','','','',''],
    ['Macarrão cru','60g','','','','',''],['Batata cozida','345g','','','','',''],
    ['Abóbora cabotiã','420g','','','','',''],['Mandioca cozida','120g','','','','',''],
    ['Pão (2-3 fatias)','~100g','','','','',''],['Tapioca','75g','','','','',''],
    ['Rap10','3 un','','','','',''],
    ['','','','','','',''],
    ['ALIMENTO_B','','','','','',''],
    ['titulo','Complemento','1x','🫘','','',''],['nome','qtd','nota','','','',''],
    ['Feijão (caroços cozidos)','60g','','','','',''],
    ['Lentilha / grão de bico','30g','','','','',''],
    ['Arroz / macarrão cozido','30g','','','','',''],
    ['Batata doce cozida','35g','','','','',''],['Batata cozida','80g','','','','',''],
    ['1 fatia pão / 1 rap10','1 un','','','','',''],
    ['Tapioca','30g','','','','',''],['Molho de tomate','1 concha','','','','',''],
    ['','','','','','',''],
    ['ALIMENTO_C','','','','','',''],
    ['titulo','Proteína','1,5x','🍗','','',''],['nome','qtd','nota','','','',''],
    ['Peito de frango','150g','','','','',''],
    ['Filé mignon suíno','150g','','','','',''],
    ['Patinho/coxão/músculo','150g','não precisa óleo','','','',''],
    ['Tilápia grelhada/assada','150g','','','','',''],
    ['Atum natural (lata)','1,5 lata','','','','',''],
    ['Whey concentrado','60g','','','','',''],
    ['1 ovo + whey','1 ovo + 45g whey','manter ovos fixos','','','',''],
    ['2 ovos + whey','2 ovos + 30g whey','SEM azeite na salada','','','',''],
    ['','','','','','',''],
    ['FRUTAS','','','','','',''],['nome','qtd','','','','',''],
    ['Mamão, abacaxi, uva, laranja, pera, maçã','150g','','','','',''],
    ['Melão, melancia, morango','200g','','','','',''],
    ['Banana','70g','','','','',''],['Abacate','40g','','','','',''],
    ['','','','','','',''],
    ['DICAS','','','','','',''],
    ['Pode inverter a ordem das refeições','','','','','',''],
    ['Pode juntar duas refeições seguidas','','','','','',''],
    ['Pode separar itens pra comer em outro horário','','','','','',''],
    ['Fim de semana: pule ref. 3+4, coma fora com controle, +30g whey','','','','','',''],
    ['1500 kcal livres = ~3 pizzas OU 1 lanche+batata OU 5-6 esfihas','','','','','',''],
    ['Deslizou? Volte ao plano. Um erro não justifica abandonar o dia','','','','','',''],
  ];
  sh.getRange(1, 1, rows.length, 7).setValues(rows);
  sh.setColumnWidth(1, 250); sh.setColumnWidth(2, 200); sh.setColumnWidth(3, 80);
  var sections = ['MACROS','REFEICOES','ITENS_FIXOS','ALIMENTO_A','ALIMENTO_B','ALIMENTO_C','FRUTAS','DICAS'];
  for (var i = 0; i < rows.length; i++) {
    if (sections.indexOf(String(rows[i][0])) >= 0)
      sh.getRange(i+1,1,1,7).setBackground('#2D5F8A').setFontColor('#FFFFFF').setFontWeight('bold');
  }
  SpreadsheetApp.getUi().alert('Aba "Dieta" criada!');
}

function testeGet() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var names = ['A','B','C','D'];
  for (var n = 0; n < names.length; n++) {
    var sh = ss.getSheetByName(names[n]);
    if (sh) {
      var r = parseWorkout(sh);
      Logger.log(names[n] + ': ' + r.exercises.length + ' exercises, ' + r.mobility.length + ' mobility, ' + r.rest.length + ' rest rules, cadence=' + r.cadence);
      for (var i = 0; i < r.rest.length; i++) {
        Logger.log('  Rest: ' + r.rest[i].label + ' (' + r.rest[i].repsMin + '-' + r.rest[i].repsMax + ' reps) -> ' + r.rest[i].rest + '-' + r.rest[i].restMax + 's');
      }
      for (var i = 0; i < r.exercises.length; i++) {
        Logger.log('  ' + r.exercises[i].name + ' (' + r.exercises[i].sets.length + ' sets, row ' + r.exercises[i].sets[0].row + ')');
      }
    }
  }
  var dsh = ss.getSheetByName('Dieta');
  if (dsh) {
    var d = parseDiet(dsh);
    Logger.log('Diet: ' + d.meals.length + ' meals, ' + Object.keys(d.ali).length + ' groups, ' + d.tips.length + ' tips');
  } else {
    Logger.log('Aba Dieta nao encontrada');
  }
}
```
