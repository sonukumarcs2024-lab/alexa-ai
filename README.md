<!doctype html>
<html lang="hi">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Mini Alexa AI Assistant</title>
  <style>
    :root{
      --bg:#0f1724;
      --card:#0b1220;
      --accent:#06b6d4;
      --text:#e6eef8;
      --muted:#94a3b8;
    }

    *{box-sizing:border-box;}
    body{
      margin:0;
      font-family:Inter, system-ui, Arial, sans-serif;
      background:linear-gradient(180deg,#071023 0%, #0b1220 100%);
      color:var(--text);
      min-height:100vh;
      display:flex;
      align-items:center;
      justify-content:center;
      padding:20px;
    }

    .container{
      width:100%;
      max-width:820px;
      background:rgba(255,255,255,0.03);
      border-radius:12px;
      padding:20px;
      box-shadow:0 8px 30px rgba(2,6,23,0.6);
    }

    h1{margin:0 0 12px 0;font-size:22px;}
    .box{padding:12px;background:rgba(255,255,255,0.02);border-radius:8px;}
    .status{display:flex;gap:12px;align-items:center;}
    #micBtn{
      width:64px;height:64px;border-radius:12px;border:none;font-size:28px;
      background:linear-gradient(180deg, rgba(255,255,255,0.03), rgba(255,255,255,0.01));
      color:var(--text);cursor:pointer;box-shadow:0 4px 14px rgba(0,0,0,0.5);
    }
    .badge{padding:6px 10px;border-radius:999px;font-weight:600;margin-bottom:6px;display:inline-block;}
    .badge.off{background:#27323a;color:var(--muted);}
    .badge.on{background:var(--accent);color:#012b2f;}
    #transcript{font-size:14px;color:var(--muted);max-width:620px;word-wrap:break-word;}

    .controls{margin-top:12px;display:flex;gap:8px;flex-wrap:wrap;}
    .controls button{padding:8px 12px;border-radius:8px;border:none;background:rgba(255,255,255,0.03);color:var(--text);cursor:pointer;}

    .response{margin-top:16px;}
    #responseText{background:rgba(255,255,255,0.02);padding:10px;border-radius:8px;color:var(--text);min-height:36px;}

    .help{margin-top:18px;}
    .help ul{display:grid;grid-template-columns:repeat(2,1fr);gap:6px;padding-left:20px;}
    .help li{color:var(--muted);font-size:14px;}

    footer{margin-top:14px;color:var(--muted);font-size:13px;text-align:center;}
  </style>
</head>
<body>
  <div class="container">
    <h1>Mini Alexa Assistant</h1>

    <div class="box">
      <div class="status">
        <button id="micBtn" title="Start/Stop Listening">🎙️</button>
        <div>
          <div id="listeningBadge" class="badge off">OFF</div>
          <div id="transcript">Kuchh bolen... (transcript yahan aayega)</div>
        </div>
      </div>

      <div class="controls">
        <button id="startBtn">Start Listening</button>
        <button id="stopBtn">Stop</button>
        <button id="speakSample">Sample: "Alexa, hello"</button>
      </div>

      <div class="response">
        <h3>Assistant Response</h3>
        <div id="responseText">Yahan assistant ka jawab aayega.</div>
      </div>

      <div class="help">
        <h4>Commands (examples)</h4>
        <ul id="commandsList">
          <!-- Commands list populated by JS -->
        </ul>
      </div>

    </div>

    <footer>Tip: bolte waqt "alexa" shamil kar sakte ho (optional)</footer>
  </div>

  <script>
    // script.js - Mini Alexa Assistant
    // Simple voice assistant using Web Speech API (Recognition + Synthesis).
    // Place this file as script.js and make sure index.html loads it.

    (() => {
      // Compatibility
      const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
      const synth = window.speechSynthesis;
      const micBtn = document.getElementById('micBtn');
      const startBtn = document.getElementById('startBtn');
      const stopBtn = document.getElementById('stopBtn');
      const speakSample = document.getElementById('speakSample');
      const transcriptEl = document.getElementById('transcript');
      const responseEl = document.getElementById('responseText');
      const badge = document.getElementById('listeningBadge');
      const commandsList = document.getElementById('commandsList');

      // If browser doesn't support recognition
      if (!SpeechRecognition) {
        transcriptEl.innerText = "Speech Recognition unsupported in this browser.";
        micBtn.disabled = true;
        startBtn.disabled = true;
        stopBtn.disabled = true;
        speakSample.disabled = true;
        return;
      }

      const recog = new SpeechRecognition();
      recog.lang = 'en-US'; // Hindi-English mixed works okay; change to 'hi-IN' for Hindi bias
      recog.interimResults = false;
      recog.maxAlternatives = 1;
      recog.continuous = false;

      let listening = false;

      // List of supported commands (simple patterns)
      const commands = [
        {keys: ['hello', 'hi', 'hey', 'namaste'], action: () => reply("Hello! main aapki madad ke liye hoon.")},
        {keys: ['what time', 'time now', 'kya samay'], action: () => {
          const now = new Date();
          const t = now.toLocaleTimeString();
          reply("Samay hai " + t);
        }},
        {keys: ['open google', 'open google.com', 'google kholo'], action: () => {
          reply("Google khol raha hoon");
          window.open('https://www.google.com', '_blank');
        }},
        {keys: ['open youtube', 'youtube kholo'], action: () => {
          reply("YouTube khol raha hoon");
          window.open('https://www.youtube.com', '_blank');
        }},
        {keys: ['open facebook', 'facebook kholo'], action: () => { reply("Facebook khol raha hoon"); window.open('https://www.facebook.com', '_blank'); }},
        {keys: ['search for', 'search'], action: (text) => {
          // extract after 'search' phrase
          const q = text.replace(/.*search (for)?/i, '').trim();
          if (q.length === 0) return reply("Kya search karna hai? keh do");
          reply("Searching for " + q);
          window.open('https://www.google.com/search?q=' + encodeURIComponent(q), '_blank');
        }},
        {keys: ['set timer', 'set alarm', 'alarm'], action: (text) => {
          // tries to find number of seconds or minutes in text
          const m = text.match(/(\d+)\s*(second|seconds|sec|s)/i);
          const mm = text.match(/(\d+)\s*(minute|minutes|min|m)/i);
          let ms = 0;
          if (m) ms = parseInt(m[1],10)*1000;
          else if (mm) ms = parseInt(mm[1],10)*60000;
          else return reply("Timer kitna set karun? seconds ya minutes batao.");
          reply("Timer set ho gaya.");
          setTimeout(()=> {
            speak("Timer khatam hua.");
            alert("Timer finished!");
          }, ms);
        }},
        {keys: ['change background', 'background color', 'change color', 'change background to'], action: (text) => {
          const colors = text.match(/(red|green|blue|black|white|yellow|pink|purple|orange|teal)/i);
          if (colors) {
            document.body.style.background = colors[0];
            reply("Background " + colors[0] + " kar diya.");
          } else {
            // random color
            document.body.style.background = '#'+Math.floor(Math.random()*16777215).toString(16);
            reply("Background change kar diya.");
          }
        }},
        {keys: ['increase font', 'larger text', 'font size up'], action: () => {
          document.body.style.fontSize = (parseInt(getComputedStyle(document.body).fontSize) + 2) + 'px';
          reply("Font size badha diya.");
        }},
        {keys: ['decrease font', 'smaller text', 'font size down'], action: () => {
          document.body.style.fontSize = Math.max(12, parseInt(getComputedStyle(document.body).fontSize) - 2) + 'px';
          reply("Font size ghata diya.");
        }},
        {keys: ['tell me a joke', 'joke', 'hasya'], action: () => {
          reply("Ek joke: Teacher: Agar tumhara exam fail ho gaya to tum kya karoge? Student: Main phir se college join karunga... par sir, college pehle se bhara hua hai!");
        }},
        {keys: ['what is your name', 'who are you', 'tum kaun ho'], action: () => reply("Main ek mini Alexa-jaise assistant hoon.")},
        {keys: ['play'], action: () => reply("Music play karne ka option disabled hai.")},
        {keys: ['stop', 'quit', 'close'], action: () => reply("Theek hai, main ruk raha hoon.")},
        {keys: ['read text', 'read this'], action: () => {
          // speak page content
          const t = document.body.innerText.slice(0, 400);
          speak(t);
          reply("Page ka kuchh hissa bol diya.");
        }},
        {keys: ['calculate', 'what is', 'solve'], action: (text) => {
          // Try find expression e.g. "calculate 5+3" or "what is 12 * 3"
          const expr = text.replace(/.*(calculate|solve|what is|kya hai)/i, '').trim();
          if (!expr) return reply("Kya calculate karna hai?");
          // allow only numbers and +-*/().%
          if (!/^[0-9+\-*/().%\s]+$/.test(expr)) return reply("Main unsafe expressions nahi chala sakta.");
          try {
            // safe eval: use Function
            const value = Function('"use strict"; return (' + expr + ')')();
            reply(expr + " = " + value);
          } catch (e) {
            reply("Calculation samajh nahi aaya.");
          }
        }},
        {keys: ['help', 'what can you do', 'commands'], action: () => {
          reply("Main kuch basic commands samajhta hoon: time batana, web kholna, search karna, timer set karna, font change etc. Screen par commands list dekho.");
        }},
        {keys: ['thank you', 'thanks', 'shukriya'], action: () => reply("Aapka swagat hai!")},
      ];

      // Populate commands list in UI
      const sampleCommands = [
        'Alexa, hello',
        'Alexa, what time is it',
        'Alexa, open YouTube',
        'Alexa, search for football highlights',
        'Alexa, set timer for 10 seconds',
        'Alexa, change background to blue',
        'Alexa, increase font',
        'Alexa, tell me a joke',
        'Alexa, calculate 12+7*3',
        'Alexa, read text',
      ];
      sampleCommands.forEach(c => {
        const li = document.createElement('li'); li.textContent = c; commandsList.appendChild(li);
      });

      // helpers
      function speak(text) {
        if (!synth) return;
        const u = new SpeechSynthesisUtterance(text);
        // choose voice if available
        const voices = synth.getVoices();
        if (voices && voices.length) {
          u.voice = voices.find(v=>v.lang.includes('en')) || voices[0];
        }
        u.lang = 'en-US';
        synth.cancel(); // stop previous
        synth.speak(u);
      }

      function reply(text) {
        responseEl.innerText = text;
        speak(text);
      }

      // command processor
      function processCommand(raw) {
        if (!raw) return;
        let text = raw.toLowerCase().trim();
        // optional wake word: remove "alexa" if present
        text = text.replace(/^alexa[,!\s]*/i, '').trim();

        // try to match commands in order
        for (const cmd of commands) {
          for (const k of cmd.keys) {
            if (text.includes(k)) {
              try { cmd.action(text); return; } catch(e) { reply("Kuchh galat hua."); return; }
            }
          }
        }
        // fallback: try generic search
        reply("Maaf kijiye, maine command nahi samjha. Main search kar doon? Searching...");
        window.open('https://www.google.com/search?q=' + encodeURIComponent(raw), '_blank');
      }

      // recognition events
      recog.onstart = () => {
        listening = true;
        badge.className = 'badge on';
        badge.innerText = 'LISTENING';
      };
      recog.onend = () => {
        listening = false;
        badge.className = 'badge off';
        badge.innerText = 'OFF';
      };
      recog.onerror = (e) => {
        listening = false;
        badge.className = 'badge off';
        badge.innerText = 'ERROR';
        transcriptEl.innerText = 'Error: ' + (e.error || 'unknown');
      };
      recog.onresult = (ev) => {
        const t = ev.results[0][0].transcript;
        transcriptEl.innerText = t;
        processCommand(t);
      };

      // UI buttons
      function startListening() {
        try {
          recog.start();
        } catch (e) {
          // sometimes start throws if already started; ignore
        }
      }
      function stopListening() {
        try { recog.stop(); } catch(e){}
      }

      micBtn.addEventListener('click', () => {
        if (!listening) startListening(); else stopListening();
      });
      startBtn.addEventListener('click', startListening);
      stopBtn.addEventListener('click', stopListening);
      speakSample.addEventListener('click', () => {
        const sample = 'alexa hello';
        transcriptEl.innerText = sample;
        processCommand(sample);
      });

      // Keyboard shortcut: press 'l' to start/stop
      window.addEventListener('keydown', (e) => {
        if (e.key.toLowerCase() === 'l') {
          if (!listening) startListening(); else stopListening();
        }
      });

      // show initial message
      reply("Assistant tayyar hai. Bol kar dekho — optionally 'Alexa' se shuru karo.");
    })();
  </script>
</body>
</html>
