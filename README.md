# rubis.
from flask import Flask, render_template_string, request, session, redirect, jsonify
import sqlite3
import random
from werkzeug.utils import secure_filename
import os
from datetime import datetime, timedelta

app = Flask(__name__)
app.secret_key = "rubis_secret"
DB_NAME = "rubis.db"

# --- Pasta de uploads ---
UPLOAD_FOLDER = "uploads"
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER

# --- Inicializar banco de dados ---
def init_db():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT UNIQUE,
            password TEXT,
            nickname TEXT,
            plan TEXT DEFAULT 'Free',
            plus_trial_used INTEGER DEFAULT 0,
            pro_plus_trial_used INTEGER DEFAULT 0,
            trial_start TIMESTAMP
        )
    """)
    c.execute("""
        CREATE TABLE IF NOT EXISTS conversations (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            title TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    c.execute("""
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            conversation_id INTEGER,
            role TEXT,
            content TEXT
        )
    """)
    conn.commit()
    conn.close()

init_db()

# --- Templates ---
login_template = """
<!DOCTYPE html>
<html lang="pt-br">
<head>
<meta charset="UTF-8">
<title>Rubis Login</title>
<style>
body{font-family:sans-serif;display:flex;justify-content:center;align-items:center;height:100vh;background:#f0f2f5;}
form{background:#fff;padding:30px;border-radius:10px;box-shadow:0 2px 10px rgba(0,0,0,0.1);}
input{display:block;width:100%;padding:10px;margin:10px 0;border-radius:5px;border:1px solid #ccc;}
button{padding:10px;width:100%;border:none;border-radius:5px;background:#4CAF50;color:#fff;font-weight:bold;cursor:pointer;}
a{display:block;text-align:center;margin-top:10px;color:#555;text-decoration:none;}
</style>
</head>
<body>
<form action="/login" method="post">
<h2>Rubis</h2>
<input type="email" name="email" placeholder="Email" required>
<input type="password" name="password" placeholder="Senha" required>
<button type="submit">Login</button>
<a href="/register_page">Não tem conta? Cadastre-se</a>
</form>
</body>
</html>
"""

register_template = """
<!DOCTYPE html>
<html lang="pt-br">
<head>
<meta charset="UTF-8">
<title>Rubis Cadastro</title>
<style>
body{font-family:sans-serif;display:flex;justify-content:center;align-items:center;height:100vh;background:#f0f2f5;}
form{background:#fff;padding:30px;border-radius:10px;box-shadow:0 2px 10px rgba(0,0,0,0.1);}
input{display:block;width:100%;padding:10px;margin:10px 0;border-radius:5px;border:1px solid #ccc;}
button{padding:10px;width:100%;border:none;border-radius:5px;background:#2196F3;color:#fff;font-weight:bold;cursor:pointer;}
a{display:block;text-align:center;margin-top:10px;color:#555;text-decoration:none;}
</style>
</head>
<body>
<form action="/register" method="post">
<h2>Rubis</h2>
<input type="text" name="nickname" placeholder="Nickname" required>
<input type="email" name="email" placeholder="Email" required>
<input type="password" name="password" placeholder="Senha" required>
<button type="submit">Cadastrar</button>
<a href="/">Já tem conta? Login</a>
</form>
</body>
</html>
"""

chat_template = """
<!DOCTYPE html>
<html lang="pt-br">
<head>
<meta charset="UTF-8">
<title>Rubis</title>
<style>
body{margin:0;font-family:sans-serif;background:#f0f2f5;}
header{display:flex;justify-content:space-between;align-items:center;padding:10px;background:#fff;box-shadow:0 2px 5px rgba(0,0,0,0.1);}
header .name{font-weight:bold;font-size:18px;}
header .buttons button{margin-left:5px;padding:5px 10px;border:none;border-radius:5px;cursor:pointer;font-size:18px;}
#chat-container{display:flex;height:calc(100vh - 50px);}
#sidebar{width:250px;background:#fff;box-shadow:2px 0 5px rgba(0,0,0,0.1);position:fixed;left:-250px;top:50px;height:100%;overflow-y:auto;transition:left 0.3s cubic-bezier(0.25, 1, 0.5, 1);padding:10px;}
#sidebar.open{left:0;}
#chat{flex:1;margin-left:0;padding:10px;display:flex;flex-direction:column;overflow-y:auto;height:calc(100vh - 50px);}
.message{max-width:60%;padding:10px;margin:5px;border-radius:10px;}
.user{background:#4CAF50;color:#fff;align-self:flex-end;}
.bot{background:#e0e0e0;align-self:flex-start;}
#input-area{display:flex;padding:10px;background:#fff;position:fixed;bottom:0;width:100%;box-shadow:0 -2px 5px rgba(0,0,0,0.1);}
#input-area input{flex:1;padding:10px;border-radius:5px;border:1px solid #ccc;}
#input-area button{margin-left:5px;padding:10px;border:none;border-radius:5px;background:#2196F3;color:#fff;cursor:pointer;}
</style>
</head>
<body>
<header>
<div class="name">Rubis</div>
<div class="buttons">
<button id="new-convo">+</button>
<button id="upgrade">🛒</button>
<button id="menu">…</button>
<button id="history">≡</button>
</div>
</header>

<div id="chat-container">
<aside id="sidebar">
<h3>Histórico</h3>
<div id="conversations"></div>
</aside>
<div id="chat"></div>
</div>

<div id="input-area">
<input type="text" id="message" placeholder="Digite sua mensagem...">
<button id="send">Enviar</button>
<button id="upload">📎</button>
</div>

<script>
const chat = document.getElementById('chat');
const sidebar = document.getElementById('sidebar');
document.getElementById('history').onclick = ()=>sidebar.classList.toggle('open');

document.getElementById('send').onclick = async ()=>{
    let msg = document.getElementById('message').value;
    if(!msg) return;
    chat.innerHTML += `<div class="message user">${msg}</div>`;
    document.getElementById('message').value='';
    let res = await fetch('/send_message',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({message:msg})});
    let data = await res.json();
    chat.innerHTML += `<div class="message bot">${data.answer}</div>`;
    chat.scrollTop = chat.scrollHeight;
}

document.getElementById('new-convo').onclick = ()=>{fetch('/nova_conversa').then(()=>location.reload())}
document.getElementById('upgrade').onclick = ()=>alert('Upgrade de plano em breve!'); 
document.getElementById('menu').onclick = ()=>alert('Menu de contas em breve!');
</script>
</body>
</html>
"""

# --- Funções básicas ---
def get_user_by_email(email):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("SELECT * FROM users WHERE email=?",(email,))
    user=c.fetchone()
    conn.close()
    return user

def add_user(nickname,email,password):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    try:
        c.execute("INSERT INTO users (nickname,email,password) VALUES (?,?,?)",(nickname,email,password))
        conn.commit()
    except:
        conn.close()
        return False
    conn.close()
    return True

def add_message(user_id, role, content, conversation_id=None):
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    if conversation_id:
        c.execute("INSERT INTO messages (conversation_id, role, content) VALUES (?,?,?)",(conversation_id,role,content))
    conn.commit()
    conn.close()

def generate_answer(msg):
    msg_lower = msg.lower()
    perguntas = ["?", "como", "o que", "quem", "qual", "onde", "quando", "por que", "porquê"]
    resumo_keywords = ["resuma", "resumir", "resumo", "resumido"]
    casuais = ["Oi! 😎","E aí, tudo bem?","Olá! Como vai?","Oi, que bom te ver aqui!"]
    if any(k in msg_lower for k in resumo_keywords):
        return {"answer": f"Resumo solicitado: {msg}\nAqui está uma versão resumida."}
    if any(p in msg_lower for p in perguntas):
        respostas = [
            f"Explicação detalhada sobre: {msg}\nCom exemplos e contexto.",
            f"Resumo de {msg}:\n- Pontos principais\n- Informações relevantes\n- Exemplos práticos",
            f"Palavras e conceitos importantes de {msg}:\n1. Conceito 1\n2. Conceito 2\n3. Como aplicar"
        ]
        return {"answer":respostas[0]}
    return {"answer":random.choice(casuais)}

# --- Rotas ---
@app.route("/")
def index():
    if "user_id" in session:
        return redirect("/chat")
    return render_template_string(login_template)

@app.route("/register_page")
def# Chat-rubis
