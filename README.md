import os, json, tempfile, requests, textwrap
from flask import Flask, redirect, session, request, jsonify, Response
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import Flow
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from gtts import gTTS
import google.generativeai as genai
from moviepy.editor import ImageClip, AudioFileClip, concatenate_videoclips
from PIL import Image, ImageDraw

app = Flask(__name__)
app.secret_key = os.environ.get('SECRET_KEY', 'autotube-secret-999')

GEMINI_KEY    = os.environ.get('GEMINI_API_KEY')
UNSPLASH_KEY  = os.environ.get('UNSPLASH_KEY')
CLIENT_ID     = os.environ.get('GOOGLE_CLIENT_ID')
CLIENT_SECRET = os.environ.get('GOOGLE_CLIENT_SECRET')
REDIRECT_URI  = os.environ.get('REDIRECT_URI')

SCOPES = [
    'https://www.googleapis.com/auth/youtube.upload',
    'https://www.googleapis.com/auth/userinfo.email',
    'openid'
]

genai.configure(api_key=GEMINI_KEY)

# ── Ana Sayfa (HTML içinde) ────────────────────────────
HTML = """<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>AutoTube</title>
<link href="https://fonts.googleapis.com/css2?family=Syne:wght@700;800&family=DM+Sans:wght@300;400;500&display=swap" rel="stylesheet">
<style>
:root{--bg:#0a0a0f;--card:#111118;--border:#1e1e2e;--accent:#e8ff47;--green:#47ffb2;--text:#f0f0f8;--muted:#6b6b88;--red:#ff4757}
*{margin:0;padding:0;box-sizing:border-box}
body{background:var(--bg);color:var(--text);font-family:'DM Sans',sans-serif;min-height:100vh;display:flex;align-items:center;justify-content:center;padding:24px}
body::before{content:'';position:fixed;inset:0;background-image:linear-gradient(rgba(232,255,71,.03) 1px,transparent 1px),linear-gradient(90deg,rgba(232,255,71,.03) 1px,transparent 1px);background-size:60px 60px;pointer-events:none}
.wrap{width:100%;max-width:500px;position:relative;z-index:1}
.logo{text-align:center;margin-bottom:36px}
.logo-icon{width:52px;height:52px;background:var(--accent);border-radius:14px;display:inline-flex;align-items:center;justify-content:center;font-size:24px;margin-bottom:12px}
.logo h1{font-family:'Syne',sans-serif;font-size:34px;font-weight:800;letter-spacing:-1.5px}
.logo h1 span{color:var(--accent)}
.logo p{color:var(--muted);font-size:14px;margin-top:6px;line-height:1.5}
.card{background:var(--card);border:1px solid var(--border);border-radius:20px;padding:28px;margin-bottom:12px}
.btn-google{width:100%;padding:15px;background:#fff;color:#333;border:none;border-radius:12px;font-family:'DM Sans',sans-serif;font-size:15px;font-weight:500;cursor:pointer;display:flex;align-items:center;justify-content:center;gap:12px;transition:all .2s;text-decoration:none}
.btn-google:hover{background:#f0f0f0;transform:translateY(-1px)}
.connected{display:none;align-items:center;gap:10px;padding:13px 15px;background:rgba(71,255,178,.08);border:1px solid rgba(71,255,178,.2);border-radius:11px;font-size:14px;color:var(--green)}
.connected.show{display:flex}
.dot{width:8px;height:8px;background:var(--green);border-radius:50%;animation:blink 2s infinite}
@keyframes blink{0%,100%{opacity:1}50%{opacity:.3}}
.logout{margin-left:auto;font-size:12px;color:var(--muted);cursor:pointer;text-decoration:underline}
.field{margin-bottom:16px}
label{display:block;font-size:13px;color:var(--muted);margin-bottom:6px}
input,select{width:100%;background:var(--bg);border:1px solid var(--border);border-radius:10px;padding:12px 14px;color:var(--text);font-family:'DM Sans',sans-serif;font-size:15px;outline:none;transition:border-color .2s;-webkit-appearance:none}
input:focus,select:focus{border-color:var(--accent)}
select option{background:#111}
.pills{display:flex;gap:7px;flex-wrap:wrap}
.pill{padding:8px 15px;border-radius:9px;border:1px solid var(--border);background:transparent;color:var(--muted);font-family:'DM Sans',sans-serif;font-size:13px;cursor:pointer;transition:all .2s}
.pill:hover{border-color:var(--accent);color:var(--text)}
.pill.on{background:var(--accent);border-color:var(--accent);color:#000;font-weight:500}
.btn-main{width:100%;padding:17px;background:var(--accent);color:#000;border:none;border-radius:13px;font-family:'Syne',sans-serif;font-size:17px;font-weight:800;cursor:pointer;transition:all .2s}
.btn-main:hover{transform:translateY(-2px);box-shadow:0 12px 36px rgba(232,255,71,.25)}
.btn-main:disabled{opacity:.35;cursor:not-allowed;transform:none;box-shadow:none}
.progress-box{display:none;background:var(--card);border:1px solid var(--border);border-radius:20px;padding:24px;margin-top:12px}
.progress-box.show{display:block}
.prog-label{display:flex;justify-content:space-between;font-size:13px;color:var(--muted);margin-bottom:7px}
.prog-bar{height:5px;background:var(--border);border-radius:10px;overflow:hidden;margin-bottom:18px}
.prog-fill{height:100%;background:linear-gradient(90deg,var(--accent),var(--green));border-radius:10px;width:0;transition:width .5s ease}
.log{background:var(--bg);border:1px solid var(--border);border-radius:9px;padding:13px;font-family:monospace;font-size:12px;line-height:1.9;max-height:170px;overflow-y:auto;color:var(--muted)}
.l-ok{color:var(--green)}.l-info{color:var(--accent)}.l-err{color:var(--red)}
.result-box{display:none;background:linear-gradient(135deg,#0f1a0a,#0a0f1a);border:1px solid var(--green);border-radius:20px;padding:32px;text-align:center;margin-top:12px}
.result-box.show{display:block}
.result-box h2{font-family:'Syne',sans-serif;font-size:22px;font-weight:800;color:var(--green);margin:10px 0 7px}
.result-box p{color:var(--muted);font-size:14px}
.btn-yt{display:inline-block;margin-top:18px;padding:12px 26px;background:var(--green);color:#000;border-radius:10px;font-family:'Syne',sans-serif;font-weight:700;text-decoration:none;transition:all .2s}
.btn-yt:hover{transform:translateY(-2px)}
.btn-reset{display:block;margin:10px auto 0;padding:11px 22px;background:transparent;color:var(--muted);border:1px solid var(--border);border-radius:10px;font-family:'Syne',sans-serif;font-weight:700;cursor:pointer;transition:all .2s}
.btn-reset:hover{color:var(--text);border-color:var(--text)}
</style>
</head>
<body>
<div class="wrap">
  <div class="logo">
    <div class="logo-icon">▶</div>
    <h1>Auto<span>Tube</span></h1>
    <p>Google ile giriş yap, konu seç —<br>video otomatik YouTube'a yüklenir.</p>
  </div>

  <div class="card">
    <div class="connected" id="connBox">
      <div class="dot"></div>
      <span id="connEmail">Bağlandı</span>
      <span class="logout" onclick="location.href='/logout'">Çıkış</span>
    </div>
    <a href="/login" class="btn-google" id="btnGoogle">
      <svg width="20" height="20" viewBox="0 0 24 24">
        <path fill="#4285F4" d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z"/>
        <path fill="#34A853" d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z"/>
        <path fill="#FBBC05" d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l2.85-2.22.81-.62z"/>
        <path fill="#EA4335" d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z"/>
      </svg>
      Google ile Giriş Yap
    </a>
  </div>

  <div class="card">
    <div class="field">
      <label>Dil</label>
      <div class="pills" id="langP">
        <button class="pill on" onclick="pick('lang','tr',this)">🇹🇷 Türkçe</button>
        <button class="pill" onclick="pick('lang','az',this)">🇦🇿 Azərbaycan</button>
        <button class="pill" onclick="pick('lang','en',this)">🇬🇧 English</button>
      </div>
    </div>
    <div class="field">
      <label>Konu <span style="color:var(--muted);font-size:12px">(boş bırakırsan AI seçer)</span></label>
      <input type="text" id="topicIn" placeholder="örn: 2025'te pasif gelir yolları">
    </div>
    <div class="field">
      <label>Kategori</label>
      <select id="catSel">
        <option value="education">📚 Eğitim</option>
        <option value="tech">💻 Teknoloji</option>
        <option value="motivation">🔥 Motivasyon</option>
        <option value="business">💼 İş & Kariyer</option>
        <option value="health">🧘 Sağlık</option>
      </select>
    </div>
    <div class="field">
      <label>Yayın</label>
      <div class="pills" id="visP">
        <button class="pill on" onclick="pick('vis','public',this)">🌍 Herkese Açık</button>
        <button class="pill" onclick="pick('vis','unlisted',this)">🔗 Sadece Link</button>
        <button class="pill" onclick="pick('vis','private',this)">🔒 Gizli</button>
      </div>
    </div>
  </div>

  <button class="btn-main" id="btnGen" onclick="startGen()">⚡ Video Üret ve Yükle</button>

  <div class="progress-box" id="progBox">
    <div class="prog-label"><span id="progTxt">Başlatılıyor...</span><span id="progPct">0%</span></div>
    <div class="prog-bar"><div class="prog-fill" id="progFill"></div></div>
    <div class="log" id="logBox"></div>
  </div>

  <div class="result-box" id="resBox">
    <div style="font-size:44px">🎉</div>
    <h2>Video Yayında!</h2>
    <p id="resTitle"></p>
    <a href="#" class="btn-yt" id="resLink" target="_blank">YouTube'da Gör →</a>
    <button class="btn-reset" onclick="reset()">Yeni Video Üret</button>
  </div>
</div>

<script>
const S={lang:'tr',vis:'public'};
function pick(k,v,b){S[k]=v;b.closest('.pills').querySelectorAll('.pill').forEach(x=>x.classList.remove('on'));b.classList.add('on')}
function log(m,c=''){const b=document.getElementById('logBox');const d=document.createElement('div');d.className=c;d.textContent='['+new Date().toLocaleTimeString()+'] '+m;b.appendChild(d);b.scrollTop=b.scrollHeight}
function prog(p,t){document.getElementById('progFill').style.width=p+'%';document.getElementById('progPct').textContent=p+'%';document.getElementById('progTxt').textContent=t}

(async()=>{
  try{
    const d=await fetch('/me').then(r=>r.json());
    if(d.ok){
      document.getElementById('btnGoogle').style.display='none';
      document.getElementById('connBox').classList.add('show');
      document.getElementById('connEmail').textContent=d.email+' ✓';
    }
  }catch(e){}
})();

async function startGen(){
  const btn=document.getElementById('btnGen');
  const me=await fetch('/me').then(r=>r.json());
  if(!me.ok){alert('Önce Google ile giriş yap!');return;}
  btn.disabled=true;
  document.getElementById('progBox').classList.add('show');
  document.getElementById('resBox').classList.remove('show');
  document.getElementById('logBox').innerHTML='';
  log('Gemini bağlanıyor...','l-info');
  prog(10,'Script yazılıyor...');
  const steps=[
    [25,'Ses üretiliyor...','✓ Script hazır','l-ok',3000],
    [45,'Görseller indiriliyor...','✓ Ses hazır','l-ok',7000],
    [65,'Video oluşturuluyor...','✓ Görseller hazır','l-ok',11000],
    [82,'YouTube\'a yükleniyor...','✓ Video hazır','l-ok',16000],
  ];
  steps.forEach(([p,pt,lt,lc,ms])=>setTimeout(()=>{prog(p,pt);log(lt,lc);},ms));
  try{
    const res=await fetch('/generate',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({topic:document.getElementById('topicIn').value.trim(),lang:S.lang,category:document.getElementById('catSel').value,visibility:S.vis})});
    const data=await res.json();
    if(data.error)throw new Error(data.error);
    prog(100,'Tamamlandı!');
    log('🎉 Yüklendi: '+data.url,'l-ok');
    setTimeout(()=>{
      document.getElementById('progBox').classList.remove('show');
      document.getElementById('resBox').classList.add('show');
      document.getElementById('resTitle').textContent=data.title;
      document.getElementById('resLink').href=data.url;
    },800);
  }catch(e){log('HATA: '+e.message,'l-err');btn.disabled=false;}
}
function reset(){document.getElementById('resBox').classList.remove('show');document.getElementById('progBox').classList.remove('show');document.getElementById('btnGen').disabled=false;document.getElementById('topicIn').value='';}
</script>
</body>
</html>"""

@app.route('/')
def index():
    return Response(HTML, mimetype='text/html')

# ── OAuth ──────────────────────────────────────────────
@app.route('/login')
def login():
    flow = _flow()
    url, state = flow.authorization_url(access_type='offline', prompt='consent')
    session['state'] = state
    return redirect(url)

@app.route('/oauth/callback')
def callback():
    flow = _flow()
    flow.fetch_token(authorization_response=request.url)
    c = flow.credentials
    session['token'] = c.token
    session['refresh_token'] = c.refresh_token
    info = requests.get('https://www.googleapis.com/oauth2/v1/userinfo',
                        headers={'Authorization': f'Bearer {c.token}'}).json()
    session['email'] = info.get('email', '')
    return redirect('/')

@app.route('/me')
def me():
    return jsonify({'ok': 'token' in session, 'email': session.get('email', '')})

@app.route('/logout')
def logout():
    session.clear()
    return redirect('/')

# ── Video Üretim ───────────────────────────────────────
@app.route('/generate', methods=['POST'])
def generate():
    if 'token' not in session:
        return jsonify({'error': 'Giriş yapılmamış'}), 401
    d = request.json
    try:
        script  = gemini_script(d.get('topic',''), d.get('lang','tr'), d.get('category','education'))
        audio   = make_audio(script['script'], d.get('lang','tr'))
        images  = get_images(script['topic'])
        video   = make_video(images, audio, script)
        url     = yt_upload(video, script, d.get('visibility','public'))
        return jsonify({'ok': True, 'url': url, 'title': script['title']})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

def gemini_script(topic, lang, category):
    lmap = {'tr':'Türkçe','az':'Azerbaycan Türkçesi','en':'English'}
    ln = lmap.get(lang,'Türkçe')
    t  = f'Konu: "{topic}"' if topic else f'{category} kategorisinde ilgi çekici bir konu seç.'
    prompt = f"""Sen YouTube içerik yazarısın. Dil: {ln}. {t}
SADECE JSON döndür:
{{"title":"(max 70 kar)","description":"(150-250 kar)","topic":"(konu)","script":"(~400 kelime anlatım)","tags":["t1","t2","t3","t4","t5"]}}"""
    m = genai.GenerativeModel('gemini-1.5-flash')
    r = m.generate_content(prompt)
    return json.loads(r.text.strip().replace('```json','').replace('```','').strip())

def make_audio(text, lang):
    codes = {'tr':'tr','az':'az','en':'en'}
    tts = gTTS(text=text, lang=codes.get(lang,'tr'), slow=False)
    p = tempfile.mktemp(suffix='.mp3')
    tts.save(p)
    return p

def get_images(topic):
    r = requests.get('https://api.unsplash.com/search/photos',
                     params={'query': topic, 'per_page': 8, 'orientation': 'landscape'},
                     headers={'Authorization': f'Client-ID {UNSPLASH_KEY}'})
    results = r.json().get('results', [])
    paths = []
    for item in results[:8]:
        p = tempfile.mktemp(suffix='.jpg')
        with open(p, 'wb') as f:
            f.write(requests.get(item['urls']['regular']).content)
        paths.append(p)
    while len(paths) < 4:
        img = Image.new('RGB', (1280, 720), '#0a0a0f')
        p = tempfile.mktemp(suffix='.png')
        img.save(p)
        paths.append(p)
    return paths

def make_video(imgs, audio_path, script):
    audio = AudioFileClip(audio_path)
    dur   = audio.duration / len(imgs)
    clips = []
    for p in imgs:
        img = Image.open(p).convert('RGB').resize((1280,720), Image.LANCZOS)
        draw = ImageDraw.Draw(img)
        draw.rectangle([0,660,1280,720], fill=(0,0,0))
        draw.text((20,672), script['title'][:70], fill=(232,255,71))
        out = tempfile.mktemp(suffix='.jpg')
        img.save(out)
        clips.append(ImageClip(out).set_duration(dur))
    video = concatenate_videoclips(clips, method='compose').set_audio(audio)
    out = tempfile.mktemp(suffix='.mp4')
    video.write_videofile(out, fps=24, codec='libx264', audio_codec='aac', logger=None)
    return out

def yt_upload(video_path, script, visibility):
    creds = Credentials(token=session['token'], refresh_token=session.get('refresh_token'),
                        token_uri='https://oauth2.googleapis.com/token',
                        client_id=CLIENT_ID, client_secret=CLIENT_SECRET)
    yt = build('youtube','v3', credentials=creds)
    body = {'snippet':{'title':script['title'],'description':script['description'],
                        'tags':script.get('tags',[]),'categoryId':'27'},
            'status':{'privacyStatus': visibility}}
    media = MediaFileUpload(video_path, mimetype='video/mp4', resumable=True)
    resp  = yt.videos().insert(part='snippet,status', body=body, media_body=media).execute()
    return f"https://www.youtube.com/watch?v={resp['id']}"

def _flow():
    return Flow.from_client_config(
        {"web":{"client_id":CLIENT_ID,"client_secret":CLIENT_SECRET,
                "redirect_uris":[REDIRECT_URI],
                "auth_uri":"https://accounts.google.com/o/oauth2/auth",
                "token_uri":"https://oauth2.googleapis.com/token"}},
        scopes=SCOPES)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT',5000)))
    
