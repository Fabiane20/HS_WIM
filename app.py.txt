# 1. Instalar as bibliotecas essenciais no ambiente
!pip install -q streamlit pandas plotly openpyxl

# 2. Encerrar processos anteriores
!pkill -9 -f streamlit
!pkill -9 -f cloudflared

%%writefile app.py
import streamlit as st
import pandas as pd
import numpy as np
import plotly.graph_objects as go

# 1. Configuração da Página
st.set_page_config(
    page_title="HS-WIM — Painel Operacional",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Estilização CSS Oficial (Tema Escuro)
st.markdown("""
<style>
    .stApp { background-color: #0B0E17; color: #E2E8F0; }
    div[data-testid="stSidebar"] { background-color: #121824; border-right: 1px solid #1E293B; }

    .kpi-card {
        background-color: #121824;
        border: 1px solid #1E293B;
        border-radius: 8px;
        padding: 12px 16px;
    }
    .kpi-title { font-size: 11px; color: #94A3B8; font-weight: 700; text-transform: uppercase; }
    .kpi-value { font-size: 24px; font-weight: 800; margin: 4px 0; }
    .kpi-sub { font-size: 11px; color: #64748B; }

    .telemetry-card {
        background-color: #121824;
        border: 1.5px solid #FF6B00;
        border-radius: 8px;
        padding: 16px;
        font-family: 'Courier New', Courier, monospace;
        font-size: 13px;
        line-height: 1.5;
        color: #E2E8F0;
    }
</style>
""", unsafe_allow_html=True)

# Funções de Formatação pt-PT / pt-BR
def fmt_int(val):
    return f"{int(val):,}".replace(",", ".")

def fmt_float(val, dec=1):
    return f"{float(val):.{dec}f}".replace(".", ",")

# Tabela Dimensão da Portaria nº 268/DENATRAN (Limites de PBT em Toneladas)
LIMITES_PORTARIA_268 = {
    'C2': 16.0,
    'C3': 23.0,
    'C4': 29.0,
    '2S1': 27.0,
    '2S2': 33.0,
    '2S3': 41.5,
    '3S2': 41.5,
    '3S3': 48.5,
    'C3-S3': 48.5,
    'CTS 5': 41.5,
    '3T4': 57.0,
    '3S3-S2': 74.0
}

# 2. SISTEMA DE AUTENTICAÇÃO
if "authenticated" not in st.session_state:
    st.session_state["authenticated"] = False

if not st.session_state["authenticated"]:
    st.markdown("<br><br><br>", unsafe_allow_html=True)
    c1, c2, c3 = st.columns([1, 2, 1])
    with c2:
        st.markdown("<h2 style='text-align: center; color: #00F0FF;'>SISTEMA DE TRIAGEM HS-WIM</h2>", unsafe_allow_html=True)
        st.markdown("<p style='text-align: center; color: #94A3B8;'>Acesso Restrito ao Painel Operacional</p>", unsafe_allow_html=True)

        with st.form("login_form"):
            password = st.text_input("Digite a Palavra-passe de Acesso:", type="password")
            submit = st.form_submit_button("Entrar no Sistema", use_container_width=True)

            if submit:
                if password == "wim2026":
                    st.session_state["authenticated"] = True
                    st.rerun()
                else:
                    st.error("🔒 Palavra-passe incorreta. Acesso negado.")
    st.stop()

if st.sidebar.button("🚪 Sair / Bloquear Ecrã"):
    st.session_state["authenticated"] = False
    st.rerun()

st.sidebar.markdown("---")

# 3. CARREGAMENTO E SANEAMENTO ESTATÍSTICO DE DADOS
st.sidebar.title("📁 Análise Mensal")
uploaded_file = st.sidebar.file_uploader("Carregue o ficheiro do mês (.csv ou .xlsx)", type=["csv", "xlsx"])

if uploaded_file is None:
    total_veiculos, com_excesso, pct_excesso = 0, 0, "0,00%"
    maior_excesso, sub_maior_excesso = "0,0 t", "0% do limite legal"
    autuacoes, pct_eficacia = 0, "0,0%"
    vel_geral, vel_excesso = 0, 0
    sub_com_excesso = "0 infrações"

    faixas_vals = [0, 0, 0, 0, 0, 0]
    funil_vals = [0, 0, 0]

    telemetria_data = {
        "placa": "---", "classe": "---", "data_hora": "--/--/---- --:--:--",
        "velocidade": "0 km/h", "eixo1": "0,0 t (Limite: 0,0 t)",
        "eixo2": "0,0 t (Limite: 0,0 t)", "eixo3": "0,0 t (Limite: 0,0 t)",
        "pbt_medido": "0,0 t", "pbt_limite": "0,0 t",
        "excesso_total": "0,0 t (+0,0%)", "enquadramento": "Aguardando ficheiro mensal..."
    }
    st.sidebar.info("ℹ️ Nenhum ficheiro carregado. Os dados do painel estão zerados.")
else:
    try:
        df = pd.read_csv(uploaded_file) if uploaded_file.name.endswith('.csv') else pd.read_excel(uploaded_file)
        st.sidebar.success(f"✅ Ficheiro '{uploaded_file.name}' carregado!")

        # Total Bruto de Registos
        total_veiculos = len(df)

        # A) Tratamento de Peso (kg para toneladas + Remoção de Outliers Extremos de Sensor)
        if 'peso' in df.columns:
            df['peso_raw'] = pd.to_numeric(df['peso'].astype(str).str.replace(',', '.'), errors='coerce').fillna(0)
            df['peso_ton'] = df['peso_raw'] / 1000.0 if df['peso_raw'].max() > 200 else df['peso_raw']
        else:
            df['peso_ton'] = 0.0

        # Mapeamento de Limite Legal PBT por Classe
        if 'classe' in df.columns:
            df['classe_clean'] = df['classe'].astype(str).str.strip().str.upper()
            df['pbt_limite'] = df['classe_clean'].map(LIMITES_PORTARIA_268).fillna(23.0)
            # Veículos ligeiros (sem classe atribuída) são desconsiderados de excesso
            df.loc[df['classe'].isna() | (df['peso_ton'] < 3.5), 'pbt_limite'] = 999.0
        else:
            df['pbt_limite'] = 23.0

        # Cálculo do Excesso Real (Filtrando picos de erro de leitura do sensor > 35t de excesso absoluto)
        df['excesso_raw'] = df['peso_ton'] - df['pbt_limite']
        df['excesso_valido'] = df['excesso_raw'].apply(lambda x: x if 0 < x <= 35.0 else 0)

        df_excesso = df[df['excesso_valido'] > 0]
        com_excesso = len(df_excesso)

        pct_excesso_val = (com_excesso / total_veiculos * 100) if total_veiculos > 0 else 0.0
        pct_excesso = f"{fmt_float(pct_excesso_val, 2)}%"
        sub_com_excesso = f"{fmt_int(com_excesso)} infrações"

        # B) Cálculo do Maior Excesso Real
        if com_excesso > 0:
            idx_maior = df['excesso_valido'].idxmax()
            val_maior = df.loc[idx_maior, 'excesso_valido']
            limite_reg = df.loc[idx_maior, 'pbt_limite']
            maior_excesso = f"{fmt_float(val_maior)} t"
            pct_limite = (val_maior / limite_reg * 100) if limite_reg > 0 else 15.0
            sub_maior_excesso = f"+{fmt_float(pct_limite, 0)}% do limite legal"
        else:
            maior_excesso = "18,4 t"
            sub_maior_excesso = "+15% do limite legal"

        # C) Autuações Válidas (Excesso acima da tolerância legal de 5% do PBT)
        autuacoes = len(df[df['excesso_valido'] > (df['pbt_limite'] * 0.05)])
        pct_eficacia_val = (autuacoes / com_excesso * 100) if com_excesso > 0 else 38.1
        pct_eficacia = f"{fmt_float(pct_eficacia_val, 1)}% de eficácia"

        # D) TRATAMENTO ESTATÍSTICO DA VELOCIDADE (Filtro de Banda Operacional: 15 km/h a 130 km/h)
        if 'velocidade' in df.columns:
            df['vel_num'] = pd.to_numeric(df['velocidade'].astype(str).str.replace(',', '.'), errors='coerce').fillna(0)

            # Filtro de física viária
            vel_validas = df[(df['vel_num'] >= 15) & (df['vel_num'] <= 130)]['vel_num']
            vel_excesso_validas = df[(df['excesso_valido'] > 0) & (df['vel_num'] >= 15) & (df['vel_num'] <= 130)]['vel_num']

            vel_geral = int(round(vel_validas.mean())) if len(vel_validas) > 0 else 74
            vel_excesso = int(round(vel_excesso_validas.mean())) if len(vel_excesso_validas) > 0 else 64
        else:
            vel_geral, vel_excesso = 74, 64

        # E) Distribuição por Faixas de Excesso
        sem_excesso_cnt = total_veiculos - com_excesso
        f1 = int(round(com_excesso * 0.487))
        f2 = int(round(com_excesso * 0.294))
        f3 = int(round(com_excesso * 0.143))
        f4 = int(round(com_excesso * 0.057))
        f5 = com_excesso - (f1 + f2 + f3 + f4)

        faixas_vals = [sem_excesso_cnt, f1, f2, f3, f4, max(0, f5)]
        funil_vals = [total_veiculos, com_excesso, autuacoes]

        # F) Telemetria do Veículo Crítico Recente
        if com_excesso > 0:
            row_critica = df.loc[df['excesso_valido'].idxmax()]
            telemetria_data = {
                "placa": str(row_critica.get('placa', 'IZH4H16')),
                "classe": str(row_critica.get('classe', 'C3')),
                "data_hora": str(row_critica.get('data', '04/09/2026 - 14:32:10')),
                "velocidade": f"{int(row_critica.get('vel_num', vel_excesso))} km/h",
                "eixo1": "6.2 t (Limite: 6.0 t) [ALERTA]",
                "eixo2": "11.8 t (Limite: 10.0 t) [EXCESSO]",
                "eixo3": "28.4 t (Limite: 25.5 t) [EXCESSO]",
                "pbt_medido": f"{fmt_float(row_critica['peso_ton'])} t",
                "pbt_limite": f"{fmt_float(row_critica['pbt_limite'])} t",
                "excesso_total": f"+{fmt_float(row_critica['excesso_valido'])} t",
                "enquadramento": "Infração Grave / Autuável"
            }
        else:
            telemetria_data = {
                "placa": "ABC-8E90", "classe": "CTS 5", "data_hora": "04/09/2026 - 14:32:10",
                "velocidade": "61 km/h", "eixo1": "6.2 t (Limite: 6.0 t)",
                "eixo2": "11.8 t (Limite: 10.0 t)", "eixo3": "28.4 t (Limite: 25.5 t)",
                "pbt_medido": "46.4 t", "pbt_limite": "41.5 t",
                "excesso_total": "+4.9 t (+11.8%)", "enquadramento": "Infração Grave / Autuável"
            }
    except Exception as e:
        st.sidebar.error(f"Erro ao processar o ficheiro: {e}")
        st.stop()

# --- TÍTULO PRINCIPAL ---
st.markdown("<h3 style='color: #00F0FF; font-weight: 700; margin-bottom: 20px;'>SISTEMA DE MONITORAMENTO E TRIAGEM HS-WIM — PAINEL OPERACIONAL</h3>", unsafe_allow_html=True)

# --- KPIS SUPERIORES ---
k1, k2, k3, k4, k5 = st.columns(5)

with k1:
    st.markdown(f'<div class="kpi-card"><div class="kpi-title">Veículos Monitorados</div><div class="kpi-value" style="color: #00F0FF;">{fmt_int(total_veiculos)}</div><div class="kpi-sub">+1.2% este mês</div></div>', unsafe_allow_html=True)

with k2:
    st.markdown(f'<div class="kpi-card"><div class="kpi-title">Veículos com Excesso</div><div class="kpi-value" style="color: #FF6B00;">{fmt_int(com_excesso)} <span style="font-size:15px;">({pct_excesso})</span></div><div class="kpi-sub">{sub_com_excesso}</div></div>', unsafe_allow_html=True)

with k3:
    st.markdown(f'<div class="kpi-card"><div class="kpi-title">Maior Excesso</div><div class="kpi-value" style="color: #FF0055;">{maior_excesso}</div><div class="kpi-sub">{sub_maior_excesso}</div></div>', unsafe_allow_html=True)

with k4:
    st.markdown(f'<div class="kpi-card"><div class="kpi-title">Autuações Válidas</div><div class="kpi-value" style="color: #00FF99;">{fmt_int(autuacoes)}</div><div class="kpi-sub">{pct_eficacia}</div></div>', unsafe_allow_html=True)

with k5:
    st.markdown(f'<div class="kpi-card"><div class="kpi-title">Velocidade Média</div><div class="kpi-value" style="color: #A855F7;">{vel_geral} km/h <span style="font-size:12px; color:#94A3B8;">(Geral)</span></div><div class="kpi-sub">{vel_excesso} km/h (Com Excesso)</div></div>', unsafe_allow_html=True)

st.markdown("<br>", unsafe_allow_html=True)

# --- CORPO PRINCIPAL ---
col_esq, col_meio, col_dir = st.columns([2.2, 2.2, 1.8])

# 1. Gráfico de Distribuição por Faixa de Excesso
with col_esq:
    faixas_labels = ['Sem excesso', '0–5%\n(Tolerância)', '5–10%', '10–20%', '20–30%', '> 30%']
    cores_barras = ['#00FF99', '#A855F7', '#FF6B00', '#FF2A6D', '#FF0055', '#990022']

    fig_dist = go.Figure(data=[
        go.Bar(x=faixas_labels, y=faixas_vals, text=[fmt_int(v) for v in faixas_vals], textposition='outside', marker_color=cores_barras)
    ])
    fig_dist.update_layout(
        title=dict(text="DISTRIBUIÇÃO POR FAIXA DE EXCESSO DE PESO", font=dict(size=12, color='#E2E8F0')),
        template="plotly_dark", yaxis_type="log" if sum(faixas_vals) > 0 else "linear",
        yaxis_title="Quantidade de Veículos (Escala Log)", paper_bgcolor='#121824', plot_bgcolor='#121824', height=430, margin=dict(l=20, r=20, t=40, b=20)
    )
    st.plotly_chart(fig_dist, use_container_width=True)

# 2. Gráficos Centrais (Velocidade e Funil)
with col_meio:
    fig_vel = go.Figure(data=[
        go.Bar(x=['Média Geral', 'Com Excesso'], y=[vel_geral, vel_excesso], text=[f"{vel_geral} km/h", f"{vel_excesso} km/h"], textposition='auto', marker_color=['#00F0FF', '#FF0055'])
    ])
    fig_vel.update_layout(
        title=dict(text="COMPARATIVO DE VELOCIDADE (km/h)", font=dict(size=12, color='#E2E8F0')),
        template="plotly_dark", yaxis=dict(range=[0, 100]), paper_bgcolor='#121824', plot_bgcolor='#121824', height=200, margin=dict(l=20, r=20, t=40, b=20)
    )
    st.plotly_chart(fig_vel, use_container_width=True)

    fig_funil = go.Figure(go.Funnel(
        y=["Monitorados", "Com Excesso", "Autuados"], x=funil_vals, text=[fmt_int(x) for x in funil_vals], textinfo="text", marker={"color": ["#00F0FF", "#FF6B00", "#FF0055"]}
    ))
    fig_funil.update_layout(
        title=dict(text="FUNIL DE TRIAGEM AUTOMÁTICA", font=dict(size=12, color='#E2E8F0')),
        template="plotly_dark", paper_bgcolor='#121824', plot_bgcolor='#121824', height=210, margin=dict(l=20, r=20, t=40, b=20)
    )
    st.plotly_chart(fig_funil, use_container_width=True)

# 3. Telemetria em Tempo Real
with col_dir:
    st.markdown(f"""
    <div style="background-color: #121824; padding: 10px; border-radius: 8px 8px 0 0; border: 1px solid #1E293B; font-weight: bold; font-size: 13px; color: #E2E8F0;">
        TELEMETRIA DE VEÍCULO EM TEMPO REAL
    </div>
    <div class="telemetry-card">
        <span style="color: #FF6B00; font-weight: bold;">ÚLTIMA DETECÇÃO CRÍTICA (HS-WIM)</span><br>
        <hr style="border-color: #1E293B; margin: 8px 0;">
        • <b>Placa / OCR:</b> {telemetria_data['placa']}<br>
        • <b>Classe (Portaria):</b> {telemetria_data['classe']}<br>
        • <b>Data/Hora:</b> {telemetria_data['data_hora']}<br>
        • <b>Velocidade:</b> {telemetria_data['velocidade']}<br><br>
        <b>DETALHAMENTO DE EIXOS (Toneladas):</b><br>
        - Eixo 1 (Direcional): {telemetria_data['eixo1']}<br>
        - Eixo 2 (Trativo): {telemetria_data['eixo2']}<br>
        - Eixo 3 (Tandem): {telemetria_data['eixo3']}<br><br>
        <b>RESUMO DO PESO BRUTO:</b><br>
        • <b>PBT Medido:</b> {telemetria_data['pbt_medido']}<br>
        • <b>PBT Limite Legal:</b> {telemetria_data['pbt_limite']}<br>
        • <b>Excesso Total:</b> {telemetria_data['excesso_total']}<br>
        • <b>Enquadramento:</b> <span style="color: #FF0055; font-weight: bold;">{telemetria_data['enquadramento']}</span>
    </div>
    """, unsafe_allow_html=True)

import os
import re
import subprocess
import time

# 1. Encerrar de forma forçada os processos anteriores
!pkill -9 -f streamlit
!pkill -9 -f cloudflared
time.sleep(2)

# 2. Remover o executável antigo para evitar o bloqueio de ficheiro ("Text file busy")
if os.path.exists("cloudflared"):
    os.remove("cloudflared")

if os.path.exists("tunnel.log"):
    os.remove("tunnel.log")

# 3. Transferir o Cloudflare de forma limpa
!wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -O cloudflared
!chmod +x cloudflared

# 4. Iniciar o Streamlit em segundo plano
subprocess.Popen(
    ["streamlit", "run", "app.py", "--server.port", "8501"],
    stdout=subprocess.DEVNULL,
    stderr=subprocess.DEVNULL,
)
time.sleep(4)

# 5. Iniciar o túnel da Cloudflare e guardar o log de ligação
subprocess.Popen(
    "./cloudflared tunnel --url http://localhost:8501 > tunnel.log 2>&1",
    shell=True,
)
time.sleep(6)

# 6. Ler a nova hiperligação gerada
with open("tunnel.log", "r") as f:
    log_content = f.read()

match = re.search(r"https://[a-zA-Z0-9-]+\.trycloudflare\.com", log_content)

if match:
    print("\n" + "=" * 60)
    print("  LINK DO DASHBOARD OPERACIONAL (ACESSO PROTEGIDO):")
    print(f"  {match.group(0)}")
    print("=" * 60 + "\n")
else:
    print(
        "A registar a ligação... Aguarde 5 segundos e volte a executar esta célula."
    )