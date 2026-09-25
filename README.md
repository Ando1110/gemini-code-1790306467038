Final Project LLMBased Tools and Gemini API Integration for Data Scientists
import os
import streamlit as st
import google.generativeai as genai
from dotenv import load_dotenv

load_dotenv()
api_key = os.getenv("GEMINI_API_KEY")

st.set_page_config(
    page_title="SafetyBot - K3 Assistant",
    page_icon="🦺",
    layout="wide"
)

# Sidebar Konfigurasi Parameter
with st.sidebar:
    st.title("⚙️ Konfigurasi Model")
    api_key_input = st.text_input("Gemini API Key", value=api_key or "", type="password")
    
    tone = st.selectbox(
        "Gaya Bahasa / Nada",
        ["Formal & Audit-Ready (SOP/Laporan)", "Ringkas & Taktis (Safety Talk Lapangan)"]
    )
    
    industry = st.selectbox(
        "Fokus Industri",
        ["Umum / Manufaktur", "Pertambangan", "Konstruksi & Bekerja di Ketinggian", "Telekomunikasi & Elektrikal"]
    )
    
    temp = st.slider("Kreativitas (Temperature)", min_value=0.0, max_value=1.0, value=0.2, step=0.05)
    
    if st.button("Reset Percakapan"):
        st.session_state.messages = []
        st.rerun()

# System Prompt Generator
system_instruction = f"""
Anda adalah SafetyBot, asisten ahli K3 (Keselamatan dan Kesehatan Kerja / HSE).
Fokus industri: {industry}.
Gaya respon: {tone}.

Panduan Kerja:
1. Prioritaskan keselamatan jiwa dan kepatuhan standar kerja selamat.
2. Ketika menganalisis bahaya, gunakan prinsip Hierarki Pengendalian Bahaya (Eliminasi, Substitusi, Rekayasa Teknik, Administrasi, APD).
3. Buat respon terstruktur dengan poin-poin yang mudah dipahami dan aplikatif di lapangan.
"""

# Inisialisasi Model
active_key = api_key_input or api_key
if active_key:
    genai.configure(api_key=active_key)
    model = genai.GenerativeModel(
        model_name="gemini-1.5-flash",
        generation_config={"temperature": temp, "top_p": 0.95},
        system_instruction=system_instruction
    )
else:
    model = None

# Inisialisasi Chat History
if "messages" not in st.session_state:
    st.session_state.messages = [
        {"role": "assistant", "content": "Halo! Saya **SafetyBot**, asisten K3 digital Anda. Ada pekerjaan, bahaya kerja, atau penyusunan mitigasi risiko (JSA) yang perlu didiskusikan hari ini?"}
    ]

# Header Utama
st.title("🦺 SafetyBot: Asisten Cerdas K3 & Mitigasi Bahaya")
st.caption("Solusi konsultasi cepat identifikasi bahaya, safety talk, dan penyusunan mitigasi risiko kerja.")

# Render Percakapan
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

# User Input
if prompt := st.chat_input("Tulis pertanyaan atau skenario kerja di sini..."):
    if not active_key:
        st.error("Silakan masukkan API Key Anda di sidebar terlebih dahulu.")
    else:
        st.session_state.messages.append({"role": "user", "content": prompt})
        with st.chat_message("user"):
            st.markdown(prompt)

        # Generate Response
        with st.chat_message("assistant"):
            message_placeholder = st.empty()
            full_response = ""
            
            # Format history untuk konteks LLM
            formatted_history = []
            for m in st.session_state.messages[:-1]:
                role = "user" if m["role"] == "user" else "model"
                formatted_history.append({"role": role, "parts": [m["content"]]})
            
            chat = model.start_chat(history=formatted_history)
            response = chat.send_message(prompt, stream=True)
            
            for chunk in response:
                if chunk.text:
                    full_response += chunk.text
                    message_placeholder.markdown(full_response + "▌")
            
            message_placeholder.markdown(full_response)
        
        st.session_state.messages.append({"role": "assistant", "content": full_response})
