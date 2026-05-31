import os
import time
from flask import Flask, render_template_string, request, jsonify
from google import genai

app = Flask(__name__)

# Sunucunun API Anahtarını sistemden okumasını sağlıyoruz
api_key = os.environ.get("GEMINI_API_KEY", "AQ.Ab8RN6I59rNFQ3Xtga9WKCtxWiTLYTKNYVqi-aNKRDxNT9zLXw")

try:
    client = genai.Client(api_key=api_key)
except Exception as e:
    print(f"API Kurulum Hatası: {e}")

WEB_ARAYUZU = """
<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Akademik Asistan Web v24</title>
    <style>
        * { box-sizing: border-box; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        body { background-color: #1a202c; color: #e2e8f0; margin: 0; padding: 15px; display: flex; justify-content: center; }
        .phone-container { width: 100%; max-width: 480px; background: #2d3748; border-radius: 12px; padding: 20px; box-shadow: 0 10px 25px rgba(0,0,0,0.3); }
        h2 { color: #f6e05e; text-align: center; margin-top: 0; font-size: 20px; }
        .input-group { margin-bottom: 15px; }
        label { display: block; margin-bottom: 5px; font-size: 14px; color: #cbd5e0; }
        input, select, textarea { width: 100%; padding: 12px; border: 1px solid #4a5568; background: #1a202c; color: white; border-radius: 6px; font-size: 15px; }
        input:focus, select:focus, textarea:focus { border-color: #3182ce; outline: none; }
        .btn { width: 100%; background: #e53e3e; color: white; padding: 12px; border: none; border-radius: 6px; font-size: 16px; font-weight: bold; cursor: pointer; margin-top: 10px; }
        .btn-analyze { background: #3182ce; }
        .result-box { background: #1a202c; padding: 15px; border-left: 4px solid #3182ce; border-radius: 4px; margin-top: 15px; display: none; font-size: 14px; line-height: 1.5; max-height: 300px; overflow-y: auto; color: #fff; }
        .perf-badge { color: #f6e05e; font-size: 13px; font-weight: bold; margin-top: 10px; display: none; background: #1a202c; padding: 8px; border-radius: 4px; }
        .loader { border: 4px solid #f3f3f3; border-top: 4px solid #3182ce; border-radius: 50%; width: 24px; height: 24px; animation: spin 1s linear infinite; display: inline-block; vertical-align: middle; margin-right: 10px; }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    </style>
</head>
<body>
    <div class="phone-container">
        <h2>🛡️ Akademik Asistan (Canlı İnternet Sitesi)</h2>
        <div id="login-section">
            <div class="input-group">
                <label>Kullanıcı Adı:</label>
                <input type="text" id="username" placeholder="ahmet">
            </div>
            <div class="input-group">
                <label>Şifre:</label>
                <input type="password" id="password" placeholder="1234">
            </div>
            <div class="input-group">
                <label>Sistem Rolü:</label>
                <select id="role">
                    <option value="Geliştirici">👑 Sistem Geliştirici (Admin)</option>
                    <option value="Hoca">🎓 Akademisyen / Hoca</option>
                    <option value="Öğrenci">👨‍🎓 Üniversite Öğrencisi</option>
                </select>
            </div>
            <button class="btn" onclick="login()">Sisteme Güvenli Giriş Yap</button>
        </div>
        <div id="main-section" style="display:none;">
            <div style="text-align:right; font-size:12px; color:#68d391; margin-bottom:10px; font-weight:bold;" id="license-bar"></div>
            <div class="input-group">
                <label>İşlem Modu:</label>
                <select id="mode">
                    <option value="genel">🎓 Genel Akademik Danışman</option>
                    <option value="sinav">📝 Sınav Sorusu Hazırlama</option>
                    <option value="ozet">📚 Ders Notu Özetleme</option>
                </select>
            </div>
            <div class="input-group">
                <label>İçerik Metni:</label>
                <textarea id="content" rows="5" placeholder="Sorunuzu buraya yazın..."></textarea>
            </div>
            <button class="btn btn-analyze" onclick="runAnalysis()">Gelişmiş Analizi Başlat</button>
            <div id="perf" class="perf-badge"></div>
            <div id="result" class="result-box"></div>
        </div>
    </div>
    <script>
        let userRole = "";
        function login() {
            const user = document.getElementById('username').value;
            const pass = document.getElementById('password').value;
            const role = document.getElementById('role').value;
            if(user.toLowerCase().trim() === 'ahmet' && pass.trim() === '1234') {
                userRole = role;
                document.getElementById('login-section').style.display = 'none';
                document.getElementById('main-section').style.display = 'block';
                document.getElementById('license-bar').innerText = "🔐 Yetki: " + role + " | Durum: AKTİF";
            } else { alert("Hatalı Giriş!"); }
        }
        async function runAnalysis() {
            const mode = document.getElementById('mode').value;
            const content = document.getElementById('content').value;
            const resBox = document.getElementById('result');
            const perfBox = document.getElementById('perf');
            if(!content.trim()) { alert("Lütfen içerik alanını boş bırakmayın!"); return; }
            resBox.style.display = "block";
            resBox.innerHTML = '<div class="loader"></div> <span>Yapay zeka yanıt hazırlıyor...</span>';
            try {
                const response = await fetch('/api/analyze', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({mode: mode, content: content})
                });
                const data = await response.json();
                resBox.innerHTML = data.result.replace(/\\n/g, "<br>");
                perfBox.style.display = "block";
                perfBox.innerText = "⏱️ Yapay Zeka Hızı: " + data.latency;
            } catch(e) { resBox.innerHTML = "❌ Sunucu yanıtı işlenirken hata oluştu."; }
        }
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    return render_template_string(WEB_ARAYUZU)

@app.route('/api/analyze', methods=['POST'])
def api_analyze():
    data = request.json or {}
    mode = data.get('mode', 'genel')
    content = data.get('content', '')
    start = time.time()
    
    if mode == 'sinav':
        prompt = f"Aşağıdaki metnden 3 sınav sorusu ve cevapları çıkar: {content}"
    elif mode == 'ozet':
        prompt = f"Aşağıdaki metni özetle: {content}"
    else:
        prompt = f"Şu soruyu detaylıca yanıtla: {content}"
        
    try:
        response = client.models.generate_content(model='gemini-2.5-flash', contents=prompt)
        result_text = response.text
    except Exception as e:
        result_text = f"Yapay Zeka Bağlantı Hatası: {str(e)}"
        
    end = time.time()
    latency = f"{round(end - start, 2)} saniye"
    return jsonify({"result": result_text, "latency": latency})

if __name__ == '__main__':
    port = int(os.environ.get("PORT", 5000))
    app.run(host='0.0.0.0', port=port)
    
