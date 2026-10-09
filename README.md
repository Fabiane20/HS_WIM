# 🚚 HS-WIM — Painel Operacional de Triagem e Monitoramento

![Python](https://img.shields.io/badge/Python-3.8%2B-blue?logo=python)
![Streamlit](https://img.shields.io/badge/Streamlit-1.28%2B-FF4B4B?logo=streamlit)
![Plotly](https://img.shields.io/badge/Plotly-5.0%2B-3F4F75?logo=plotly)
![License](https://img.shields.io/badge/Licença-MIT-green)

O **HS-WIM (High-Speed Weigh-In-Motion)** é um painel operacional interativo desenvolvido para monitoramento, saneamento estatístico e triagem automática de veículos de carga em rodovias, operando em alta velocidade sem necessidade de parada no leito da via.

A aplicação foi desenhada de acordo com as especificações de limites legais de **PBT (Peso Bruto Total)** definidos pela **Portaria nº 268 do DENATRAN**, fornecendo métricas operacionais, análise de excesso de peso, perfil de velocidade e telemetria por eixo em tempo real.

---

## 📌 Principais Funcionalidades

- **🔐 Autenticação e Segurança:** Sistema de login integrado para restrição de acesso ao painel de triagem.
- **📈 KPIs Operacionais em Tempo Real:**
  - Total de veículos monitorados.
  - Quantidade e porcentagem de veículos com excesso de peso.
  - Maior excesso registrado e variação em relação ao limite legal.
  - Taxa de eficácia de autuações válidas (acima da tolerância legal de 5%).
  - Comparativo de velocidade média (geral vs. veículos infratores).
- **📊 Visualizações Dinâmicas (Plotly):**
  - **Distribuição por Faixa de Excesso:** Histograma em escala logarítmica separando veículos por faixas de tolerância e gravidade da infração (0-5%, 5-10%, 10-20%, 20-30% e >30%).
  - **Comparativo de Velocidade:** Análise física da velocidade média dos veículos pesados.
  - **Funil de Triagem Automática:** Acompanhamento da conversão do fluxo: *Monitorados → Com Excesso → Autuados*.
- **🛰️ Card de Telemetria Crítica:** Exibição do veículo com maior infração do período, com detalhamento de placa (OCR), classe regulamentar, velocidade, peso por eixo (direcional, trativo, tandem) e enquadramento legal.
- **🌐 Suporte a Deploy via Cloudflare Tunnel:** Scripts prontos para execução em ambientes em nuvem (ex: Google Colab) gerando um link público seguro sem necessidade de liberação de portas.

---

## 🛠️ Tecnologias Utilizadas

- **Linguagem:** Python 3.x
- **Dashboard:** [Streamlit](https://streamlit.io/)
- **Manipulação de Dados:** [Pandas](https://pandas.pydata.org/), [NumPy](https://numpy.org/)
- **Visualização de Dados:** [Plotly](https://plotly.com/python/)
- **Leitura de Arquivos:** OpenPyXL (Excel / CSV)
- **Túnel de Conectividade:** [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)

---

## 📐 Classes de Veículos e Limites Legais (Portaria nº 268/DENATRAN)

O sistema realiza o mapeamento automático das seguintes classes de veículos comerciais e seus respetivos limites de PBT em toneladas:

| Classe de Veículo | Limite de PBT (t) | Descrição do Agrupamento |
| :--- | :---: | :--- |
| **C2** | `16,0 t` | Caminhão Rígido (2 Eixos) |
| **C3** | `23,0 t` | Caminhão Rígido (3 Eixos) |
| **C4** | `29,0 t` | Caminhão Rígido (4 Eixos) |
| **2S1 / 2S2 / 2S3** | `27,0 t` a `41,5 t` | Cavalo Mecânico + Semirreboque |
| **3S2 / 3S3 / C3-S3** | `41,5 t` a `48,5 t` | Combinações de 5 e 6 Eixos |
| **3T4 / 3S3-S2** | `57,0 t` a `74,0 t` | Bi-trem e Rodotrem |

---

## 🚀 Como Executar o Projeto

### Opção 1: Execução no Google Colab (Recomendado)

1. Abra um novo notebook no [Google Colab](https://colab.research.google.com/).
2. Cole o código do projeto no notebook.
3. Execute as células sequencialmente.
4. O console exibirá a URL do Cloudflare Tunnel:
   ```text
   ============================================================
    LINK DO DASHBOARD OPERACIONAL (ACESSO PROTEGIDO):
    [https://xxxx-xxxx-xxxx.trycloudflare.com](https://xxxx-xxxx-xxxx.trycloudflare.com)
   ============================================================
