# danirib
Barbearia Marcos
from flask import Flask, render_template_string, request, redirect, url_for
from datetime import datetime, timedelta
import urllib.parse

app = Flask(__name__)

# ----------------------------
# BANCO DE DADOS EM MEMÓRIA
# ----------------------------
pacotes = []

# ----------------------------
# DASHBOARD HTML (Chart.js)
# ----------------------------
DASHBOARD_HTML = """
<!DOCTYPE html>
<html lang="pt-br">
<head>
<meta charset="utf-8">
<title>Dashboard - Barber Rustique</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<style>
    body { background:#111; color:#eee; font-family: Arial, sans-serif; padding:20px; }
    h1 { color:#80183b; text-align:center; margin-bottom:20px; }
    .top { display:flex; justify-content:space-between; align-items:center; gap:20px; margin-bottom:20px; }
    .card { background:#1b1b1b; padding:18px; border-radius:12px; border:1px solid #2a2a2a; width:220px; text-align:center; }
    .cards { display:flex; gap:20px; flex-wrap:wrap; justify-content:center; margin-bottom:30px; }
    canvas { background:#fff; border-radius:8px; padding:12px; display:block; margin:auto; }
    a.voltar { display:inline-block; margin-top:20px; padding:10px 16px; background:#80183b; color:#fff; border-radius:8px; text-decoration:none; }
    .small { font-size:14px; color:#ccc; }
</style>
</head>
<body>
    <div class="top">
        <div>
            <h1>Dashboard de Pacotes - Barber Rustique</h1>
            <p class="small">Visão: pacotes fechados por mês (baseado na data de início).</p>
        </div>
        <div>
            <a href="/" class="voltar">↩ Voltar ao Sistema</a>
        </div>
    </div>

    <div class="cards">
        <div class="card">
            <div class="small">Total de Pacotes</div>
            <h2 style="color:#fff; margin:6px 0;">{{ valores|sum }}</h2>
        </div>
        <div class="card">
            <div class="small">Mês com mais pacotes</div>
            <h2 style="color:#fff; margin:6px 0;">{{ labels[valores.index(max(valores))] if valores else '—' }}</h2>
        </div>
        <div class="card">
            <div class="small">Período exibido</div>
            <h2 style="color:#fff; margin:6px 0;">{{ labels[0] if labels else '—' }} → {{ labels[-1] if labels else '—' }}</h2>
        </div>
    </div>

    <h3 style="text-align:center; color:#ddd;">Pacotes Fechados por Mês</h3>
    <canvas id="graficoBarra" width="900" height="300"></canvas>

    <h3 style="text-align:center; color:#ddd; margin-top:30px;">Evolução Mensal</h3>
    <canvas id="graficoLinha" width="900" height="300"></canvas>

<script>
const labels = {{ labels|tojson }};
const valores = {{ valores|tojson }};

// Barra
new Chart(document.getElementById('graficoBarra'), {
    type: 'bar',
    data: {
        labels: labels,
        datasets: [{
            label: 'Pacotes fechados',
            data: valores,
            borderWidth: 1,
            backgroundColor: 'rgba(128,24,59,0.8)',
        }]
    },
    options: {
        scales: {
            y: { beginAtZero: true, ticks: { precision:0 } }
        }
    }
});

// Linha
new Chart(document.getElementById('graficoLinha'), {
    type: 'line',
    data: {
        labels: labels,
        datasets: [{
            label: 'Evolução mensal',
            data: valores,
            tension: 0.3,
            borderWidth: 2,
            borderColor: 'rgba(128,24,59,0.9)',
            fill: true,
            backgroundColor: 'rgba(128,24,59,0.15)'
        }]
    },
    options: {
        scales: {
            y: { beginAtZero: true, ticks: { precision:0 } }
        }
    }
});
</script>
</body>
</html>
"""

# ----------------------------
# MAIN APP HTML
# ----------------------------
HTML = """
<!DOCTYPE html>
<html lang='pt-br'>
<head>
<meta charset='UTF-8'>
<title>Barber Rustique - Controle de Pacotes</title>
<style>
    body { font-family: "Segoe UI", Arial, sans-serif; background:#111; color:#eee; margin:0; padding:0; }
    header { background:#80183b; padding:22px 18px; color:#fff; font-size:28px; font-weight:700; letter-spacing:1px; text-transform:uppercase; display:flex; justify-content:space-between; align-items:center; gap:12px; }
    header a { color:#fff; text-decoration:none; font-size:14px; background:rgba(255,255,255,0.06); padding:8px 12px; border-radius:8px; }
    .container { width:90%; max-width:1200px; margin:30px auto; background:#151515; border-radius:12px; padding:22px; box-shadow:0 8px 30px rgba(0,0,0,0.6); }
    h2 { margin-top:6px; color:#fff; font-size:20px; border-left:6px solid #80183b; padding-left:12px; }
    form { display:grid; grid-template-columns:1fr 1fr; gap:12px; margin-top:14px; align-items:end; }
    form .full { grid-column:1/-1; }
    input, select { padding:10px 12px; border-radius:8px; border:none; background:#222; color:#fff; font-size:15px; }
    button { padding:12px 14px; border-radius:10px; border:none; background:#80183b; color:#fff; font-size:16px; cursor:pointer; }
    button:hover { background:#a12452; }
    #busca { width:40%; padding:10px; border-radius:8px; border:none; background:#222; color:#fff; }
    table { width:100%; border-collapse:collapse; margin-top:18px; }
    th, td { padding:12px 10px; text-align:left; font-size:14px; border-bottom:1px solid #222; }
    th { background:#80183b; color:#fff; font-weight:700; }
    tr.ativo { background:#b6ffb3; color:#000; }
    tr.alerta { background:#fff6a0; color:#000; }
    tr.vencido { background:#ffb3b3; color:#000; }
    .whatsapp-btn { background:#25D366; padding:6px 10px; border-radius:6px; color:#fff; text-decoration:none; font-weight:700; }
    .enviado { background:#c8f7df; color:#000; padding:6px 8px; border-radius:6px; font-weight:700; display:inline-block; }
    .nao-enviado { background:#d0d0d0; color:#000; padding:6px 8px; border-radius:6px; font-weight:700; display:inline-block; }
    .actions a { color:#fff; text-decoration:none; margin-right:8px; font-weight:600; }
    .actions a:hover { text-decoration:underline; }
    @media (max-width:900px) {
        form { grid-template-columns:1fr; }
        #busca { width:100%; margin-top:10px; }
    }
</style>
</head>
<body>

<header>
    <div>Barber Rustique - Sistema de Pacotes</div>
    <div style="display:flex; gap:10px; align-items:center;">
        <a href="/dashboard">📊 Dashboard</a>
        <a href="/">🏠 Home</a>
    </div>
</header>

<div class="container">
    <h2>Cadastrar Novo Pacote</h2>
    <form method="POST" action="/add">
        <input name="cliente" placeholder="Nome do cliente" required>
        <input name="celular" placeholder="Celular (somente números) ex: 21999998888" required>
        <select name="pacote" required>
            <option value="Cabelo Ilimitado - 1 mês">Cabelo Ilimitado - 1 mês</option>
            <option value="Cabelo + Barba Ilimitado - 1 mês">Cabelo + Barba Ilimitado - 1 mês</option>
            <option value="Barba Ilimitado - 1 mês">Barba Ilimitado - 1 mês</option>
        </select>
        <input type="number" step="0.01" name="valor" placeholder="Valor do pacote (R$)" required>
        <input type="date" name="inicio" required>
        <div class="full"><button type="submit">Salvar Pacote</button></div>
    </form>

    <h2 style="margin-top:24px;">Pacotes Ativos</h2>
    <input id="busca" type="text" placeholder="Buscar cliente pelo nome..." onkeyup="filtrarClientes()">

    <table id="tabelaPacotes">
        <tr>
            <th>Cliente</th>
            <th>Celular</th>
            <th>Pacote</th>
            <th>Valor</th>
            <th>Início</th>
            <th>Término</th>
            <th>Mensagem</th>
            <th>Ações</th>
        </tr>

        {% for p in pacotes %}
        {% set hoje = datetime.utcnow().date() %}
        {% set fim_date = datetime.strptime(p.fim_display, '%d/%m/%Y').date() %}
        {% if fim_date < hoje %}
            {% set status = 'vencido' %}
        {% elif (fim_date - hoje).days <= 3 %}
            {% set status = 'alerta' %}
        {% else %}
            {% set status = 'ativo' %}
        {% endif %}

        <tr class="{{ status }}">
            <td class="nomeCliente">{{ p.cliente }}</td>
            <td>{{ p.celular }}</td>
            <td>{{ p.pacote }}</td>
            <td>R$ {{ '%.2f'|format(p.valor) }}</td>
            <td>{{ p.inicio_display }}</td>
            <td>{{ p.fim_display }}</td>
            <td>
                {% if p.mensagem_enviada %}
                    <span class="enviado">✔ Enviada</span>
                {% else %}
                    <span class="nao-enviado">Não enviada</span>
                {% endif %}
            </td>
            <td class="actions">
                <a href="/editar/{{ loop.index0 }}">Editar</a> |
                <a href="/renovar/{{ loop.index0 }}">Renovar</a> |
                <a href="/delete/{{ loop.index0 }}" onclick="return confirm('Excluir pacote?')">Excluir</a> |
                <a class="whatsapp-btn" href="/whatsapp/{{ loop.index0 }}" target="_blank">WhatsApp</a>
            </td>
        </tr>
        {% endfor %}
    </table>
</div>

<script>
function filtrarClientes(){
    let busca = document.getElementById('busca').value.toLowerCase();
    let linhas = document.querySelectorAll('#tabelaPacotes tr');
    for (let i=1; i<linhas.length; i++){
        let nomeCell = linhas[i].querySelector('.nomeCliente');
        if (!nomeCell) continue;
        let nome = nomeCell.innerText.toLowerCase();
        linhas[i].style.display = nome.includes(busca) ? '' : 'none';
    }
}
</script>

</body>
</html>
"""

# ----------------------------
# CLASSE DE PACOTES
# ----------------------------
class Pacote:
    def __init__(self, cliente, pacote, inicio_iso, celular=None, valor=0.0):
        # inicio_iso must be 'YYYY-MM-DD' string
        self.cliente = cliente
        self.pacote = pacote
        self.celular = celular
        self.valor = float(valor)
        self.inicio_raw = inicio_iso  # 'YYYY-MM-DD' used for computations & dashboard
        self.inicio_display = datetime.strptime(inicio_iso, "%Y-%m-%d").strftime("%d/%m/%Y")
        # fim_display uses 30 dias a partir do inicio_raw
        fim_dt = datetime.strptime(inicio_iso, "%Y-%m-%d") + timedelta(days=30)
        self.fim_display = fim_dt.strftime("%d/%m/%Y")
        self.mensagem_enviada = False

# ----------------------------
# ROTAS
# ----------------------------

@app.route('/')
def index():
    # ordenar por fim (data dd/mm/YYYY)
    pacotes_ordenados = sorted(pacotes, key=lambda x: datetime.strptime(x.fim_display, "%d/%m/%Y"))
    return render_template_string(HTML, pacotes=pacotes_ordenados, datetime=datetime)

@app.route('/add', methods=['POST'])
def add():
    cliente = request.form.get('cliente')
    celular = request.form.get('celular')
    pacote_tipo = request.form.get('pacote')
    valor = request.form.get('valor', 0)
    inicio = request.form.get('inicio')  # 'YYYY-MM-DD'

    novo = Pacote(cliente=cliente, pacote=pacote_tipo, inicio_iso=inicio, celular=celular, valor=valor)
    pacotes.append(novo)
    return redirect(url_for('index'))

@app.route('/editar/<int:id>', methods=['GET', 'POST'])
def editar(id):
    if id < 0 or id >= len(pacotes):
        return "Pacote não encontrado", 404
    pacote = pacotes[id]

    if request.method == 'POST':
        pacote.cliente = request.form.get('cliente')
        pacote.celular = request.form.get('celular')
        pacote.valor = float(request.form.get('valor', 0))
        pacote.pacote = request.form.get('pacote')
        novo_inicio = request.form.get('inicio')  # 'YYYY-MM-DD'
        pacote.inicio_raw = novo_inicio
        pacote.inicio_display = datetime.strptime(novo_inicio, "%Y-%m-%d").strftime("%d/%m/%Y")
        fim_dt = datetime.strptime(novo_inicio, "%Y-%m-%d") + timedelta(days=30)
        pacote.fim_display = fim_dt.strftime("%d/%m/%Y")
        # reset mensagem_enviada porque dados mudaram
        pacote.mensagem_enviada = False
        return redirect(url_for('index'))

    # edit form (simple)
    edit_html = """
    <!doctype html>
    <html lang="pt-br">
    <head><meta charset="utf-8"><title>Editar Pacote</title>
    <style>
        body { background:#111; color:#eee; font-family:Arial; padding:20px; }
        form { background:#151515; padding:18px; border-radius:10px; max-width:480px; margin:auto; }
        input, select { width:100%; padding:10px; margin:8px 0; border-radius:6px; border:none; background:#222; color:#fff; }
        button { padding:10px 12px; border-radius:8px; border:none; background:#80183b; color:#fff; cursor:pointer; }
        a { color:#ddd; text-decoration:none; display:inline-block; margin-top:10px; }
    </style>
    </head>
    <body>
    <h2 style="text-align:center;">Editar Pacote</h2>
    <form method="POST">
        <label>Cliente</label>
        <input name="cliente" value="{{ p.cliente }}" required>
        <label>Celular</label>
        <input name="celular" value="{{ p.celular }}" required>
        <label>Pacote</label>
        <input name="pacote" value="{{ p.pacote }}" required>
        <label>Valor (R$)</label>
        <input type="number" step="0.01" name="valor" value="{{ p.valor }}">
        <label>Início</label>
        <input type="date" name="inicio" value="{{ p.inicio_raw }}" required>
        <button type="submit">Salvar</button>
    </form>
    <div style="text-align:center;"><a href="{{ url_for('index') }}">↩ Voltar</a></div>
    </body></html>
    """
    return render_template_string(edit_html, p=pacotes[id])

@app.route('/renovar/<int:id>')
def renovar(id):
    if id < 0 or id >= len(pacotes):
        return "Pacote não encontrado", 404
    inicio_novo = datetime.utcnow().date()
    inicio_iso = inicio_novo.strftime("%Y-%m-%d")
    pacotes[id].inicio_raw = inicio_iso
    pacotes[id].inicio_display = inicio_novo.strftime("%d/%m/%Y")
    fim_dt = inicio_novo + timedelta(days=30)
    pacotes[id].fim_display = fim_dt.strftime("%d/%m/%Y")
    pacotes[id].mensagem_enviada = False
    return redirect(url_for('index'))

@app.route('/delete/<int:id>')
def delete(id):
    if id < 0 or id >= len(pacotes):
        return "Pacote não encontrado", 404
    pacotes.pop(id)
    return redirect(url_for('index'))

@app.route('/whatsapp/<int:id>')
def whatsapp(id):
    if id < 0 or id >= len(pacotes):
        return "Pacote não encontrado", 404
    p = pacotes[id]

    # Marca como enviado ANTES de redirecionar
    p.mensagem_enviada = True

    mensagem = (
        f"Olá {p.cliente}, tudo bem? Aqui é da Barber Rustique. "
        f"Seu pacote vence em {p.fim_display}. Aproveite para agendar sua visita!"
    )
    mensagem_codificada = urllib.parse.quote(mensagem)
    return redirect(f"https://wa.me/55{p.celular}?text={mensagem_codificada}")

@app.route('/dashboard')
def dashboard():
    # Conta pacotes por mês (baseado em inicio_raw)
    meses = {}
    for p in pacotes:
        try:
            dt = datetime.strptime(p.inicio_raw, "%Y-%m-%d")
        except Exception:
            # p.inicio_raw might be missing/invalid; skip
            continue
        chave = dt.strftime("%m/%Y")
        meses[chave] = meses.get(chave, 0) + 1

    # ordena por data (transforma 'mm/YYYY' -> 1/mm/YYYY for sorting)
    meses_ordenados = dict(sorted(
        meses.items(),
        key=lambda kv: datetime.strptime("01/" + kv[0], "%d/%m/%Y")
    ))

    labels = list(meses_ordenados.keys())
    valores = list(meses_ordenados.values())

    return render_template_string(DASHBOARD_HTML, labels=labels, valores=valores)

# ----------------------------
# RUN
# ----------------------------
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
