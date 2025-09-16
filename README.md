<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8" />
<title>Bot: Escrever em Espanhol</title>
<meta name="viewport" content="width=device-width,initial-scale=1" />
<style>
  body { font-family: Arial, sans-serif; background:#0f1720; color:#e6eef8; margin:0; padding:20px; }
  .card { max-width:820px; margin:20px auto; background:#0b1220; padding:18px; border-radius:10px; box-shadow:0 6px 20px rgba(0,0,0,.6); }
  h1{ margin:0 0 8px 0; font-size:20px; color:#ffd166 }
  label{ display:block; margin:10px 0 6px 0; font-size:13px; color:#a9c0df }
  textarea { width:100%; min-height:110px; resize:vertical; padding:10px; border-radius:6px; border:1px solid #213447; background:#08101a; color:#e6eef8; }
  .row{ display:flex; gap:8px; align-items:center; margin-top:10px; flex-wrap:wrap }
  button { background:#0077cc; color:white; border:none; padding:10px 12px; border-radius:6px; cursor:pointer; }
  button.secondary { background:#2b3948; }
  .small { font-size:13px; color:#9fb7d6 }
  .output { white-space:pre-wrap; font-family:inherit; }
  .footer { margin-top:12px; font-size:12px; color:#90a9c8 }
</style>
</head>
<body>
  <div class="card">
    <h1>Bot: Escrever em Espanhol (rápido)</h1>
    <div class="small">Digite em português (ou em qualquer idioma) e o bot traduzirá para <strong>espanhol</strong> e aplicará correções gramaticais.</div>

    <label for="inputText">Texto de entrada</label>
    <textarea id="inputText" placeholder="Escreva aqui... exemplo: 'Quero practicar mi español. Me ayudas?'"></textarea>

    <div class="row">
      <button id="btnTranslate">Traduzir & Corrigir → Espanhol</button>
      <button id="btnCopy" class="secondary">Copiar Saída</button>
      <label style="margin-left:auto">
        <input id="chkGrammar" type="checkbox" checked /> Aplicar correção gramatical (LanguageTool)
      </label>
    </div>

    <label for="outputText">Resultado em espanhol</label>
    <textarea id="outputText" readonly placeholder="Aqui aparecerá o espanhol corrigido..." class="output"></textarea>

    <div class="row" style="margin-top:8px">
      <button id="btnTranslateOnly" class="secondary">Só traduzir (sem correção)</button>
      <div class="small" style="margin-left:auto">Status: <span id="status">Pronto</span></div>
    </div>

    <div class="footer">
      <strong>Observações:</strong> o serviço usa APIs públicas (LibreTranslate + LanguageTool). Em alguns navegadores ou redes essas chamadas podem ser bloqueadas por CORS (se der erro, eu te ensino a contornar). Uso gratuito e sujeito a limites.
    </div>
  </div>

<script>
const btnTranslate = document.getElementById('btnTranslate');
const btnCopy = document.getElementById('btnCopy');
const btnTranslateOnly = document.getElementById('btnTranslateOnly');
const inputText = document.getElementById('inputText');
const outputText = document.getElementById('outputText');
const status = document.getElementById('status');
const chkGrammar = document.getElementById('chkGrammar');

function setStatus(t) { status.innerText = t; }

async function translateWithLibre(text, target='es') {
  // LibreTranslate público (pode mudar no futuro)
  const url = 'https://libretranslate.de/translate'; // endpoint público
  const body = {
    q: text,
    source: 'auto',
    target: target,
    format: 'text'
  };
  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  });
  if (!res.ok) throw new Error('Erro na tradução: ' + res.status);
  const j = await res.json();
  return j.translatedText;
}

async function grammarCheckWithLanguageTool(text, lang='es') {
  // LanguageTool pública
  const url = 'https://api.languagetool.org/v2/check';
  const form = new URLSearchParams();
  form.append('text', text);
  form.append('language', lang);
  // sugerir apenas 1 alternativa por match
  const res = await fetch(url, { method: 'POST', body: form });
  if (!res.ok) throw new Error('Erro na correção: ' + res.status);
  const j = await res.json();
  return j; // objeto com "matches"
}

function applyLanguageToolCorrections(text, matches) {
  // aplicamos substituições do final pro início pra não bagunçar offsets
  if (!matches || matches.length === 0) return text;
  // ordenar por offset decrescente
  matches.sort((a,b) => (b.offset - a.offset));
  let t = text;
  for (const m of matches) {
    if (!m.replacements || m.replacements.length === 0) continue;
    const rep = m.replacements[0].value;
    const start = m.offset;
    const end = m.offset + m.length;
    t = t.slice(0, start) + rep + t.slice(end);
  }
  return t;
}

btnTranslate.addEventListener('click', async () => {
  const text = inputText.value.trim();
  if (!text) { alert('Escreva algo primeiro.'); return; }
  try {
    setStatus('Traduzindo...');
    outputText.value = '';
    const translated = await translateWithLibre(text, 'es');
    setStatus('Traduzido. ' + (chkGrammar.checked ? 'Corrigindo...' : 'Pronto.'));
    let final = translated;
    if (chkGrammar.checked) {
      try {
        const res = await grammarCheckWithLanguageTool(translated, 'es');
        final = applyLanguageToolCorrections(translated, res.matches || []);
      } catch (e) {
        console.warn('LanguageTool falhou:', e);
        // deixamos o texto traduzido caso a correção falhe
      }
    }
    outputText.value = final;
    setStatus('Pronto');
  } catch (err) {
    console.error(err);
    setStatus('Erro — veja console');
    alert('Ocorreu um erro: ' + err.message + '\nSe for CORS, posso explicar como rodar localmente.');
  }
});

btnTranslateOnly.addEventListener('click', async () => {
  const text = inputText.value.trim();
  if (!text) { alert('Escreva algo primeiro.'); return; }
  try {
    setStatus('Traduzindo...');
    const translated = await translateWithLibre(text, 'es');
    outputText.value = translated;
    setStatus('Pronto');
  } catch (err) {
    console.error(err);
    setStatus('Erro — veja console');
    alert('Erro: ' + err.message);
  }
});

btnCopy.addEventListener('click', () => {
  const t = outputText.value;
  if (!t) { alert('Nada para copiar.'); return; }
  navigator.clipboard?.writeText(t).then(()=> {
    alert('Copiado!');
  }).catch(()=> {
    alert('Falha ao copiar — selecione e copie manualmente.');
  });
});
</script>
</body>
</html>
