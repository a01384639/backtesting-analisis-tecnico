# ======================================================
# ASISTENTE DE TRADING – INSTALACIÓN AUTOMÁTICA DE TODO
# ======================================================

import subprocess, sys

# --------- Instalar yfinance si falta ----------
try:
    import yfinance as yf
except ModuleNotFoundError:
    print("📦 Instalando yfinance...")
    subprocess.check_call([sys.executable, "-m", "pip", "install", "yfinance"])
    import yfinance as yf

# --------- Instalar matplotlib si falta ----------
try:
    import matplotlib.pyplot as plt
except ModuleNotFoundError:
    print("📦 Instalando matplotlib...")
    subprocess.check_call([sys.executable, "-m", "pip", "install", "matplotlib"])
    import matplotlib.pyplot as plt

import pandas as pd
import numpy as np
import re

plt.style.use("seaborn-v0_8")

# ======================================================
# 1. DETECTAR TIPO DE ACTIVO
# ======================================================

def tipo_activo(simbolo):
    opcion_regex = r"[A-Z]{1,5}\d{6}[CP]\d+"
    if re.match(opcion_regex, simbolo):
        return "opcion"
    return "accion"

# ======================================================
# 2. DESCARGAR DATOS
# ======================================================

def obtener_datos(simbolo):

    activo = tipo_activo(simbolo)

    if activo == "accion":
        print(f"\n📌 Detectado: ACCIÓN → {simbolo}")
        df = yf.Ticker(simbolo).history(period="1y")
        df = df[["Close"]]
        df.dropna(inplace=True)
        return df, activo
    
    else:
        print(f"\n📌 Detectado: CONTRATO DE OPCIÓN → {simbolo}")

        # extrae el ticker base
        base = simbolo[:re.search(r'\d{6}', simbolo).start()]
        print(f"   Subyacente detectado → {base}")

        df = yf.Ticker(base).history(period="1y")
        df = df[["Close"]]
        df.dropna(inplace=True)
        return df, activo

# ======================================================
# 3. INDICADORES
# ======================================================

def indicadores(df):

    df["EMA12"] = df["Close"].ewm(span=12).mean()
    df["EMA26"] = df["Close"].ewm(span=26).mean()
    df["MACD"] = df["EMA12"] - df["EMA26"]
    df["Signal"] = df["MACD"].ewm(span=9).mean()

    delta = df["Close"].diff()
    up = delta.clip(lower=0)
    down = -delta.clip(upper=0)
    rs = up.rolling(14).mean() / down.rolling(14).mean()
    df["RSI"] = 100 - (100 / (1 + rs))

    df["SMA20"] = df["Close"].rolling(20).mean()
    df["STD"] = df["Close"].rolling(20).std()
    df["Upper"] = df["SMA20"] + 2 * df["STD"]
    df["Lower"] = df["SMA20"] - 2 * df["STD"]

    return df.dropna()

# ======================================================
# 4. EVALUACIÓN DE COMPRA
# ======================================================

def analizar(df):

    macd_buy = df["MACD"].iloc[-1] > df["Signal"].iloc[-1]
    rsi_buy = df["RSI"].iloc[-1] < 35
    boll_buy = df["Close"].iloc[-1] <= df["Lower"].iloc[-1]
    tendencia = df["Close"].iloc[-50:].mean() > df["Close"].iloc[-200:].mean()

    razones = []
    if macd_buy: razones.append("✓ MACD alcista")
    if rsi_buy: razones.append("✓ RSI bajo (sobreventa)")
    if boll_buy: razones.append("✓ Precio tocando banda inferior")
    if tendencia: razones.append("✓ Tendencia alcista")

    if len(razones) >= 2:
        return "✅ Sí es buen momento para comprar", razones
    
    return "❌ No es buen momento para comprar", razones

# ======================================================
# 5. SISTEMA PRINCIPAL
# ======================================================

print("\n🤖 Asistente de Trading\n")
simbolo = input("¿Qué quieres comprar hoy?: ").upper()

df, tipo = obtener_datos(simbolo)
df = indicadores(df)

decision, razones = analizar(df)

print("\n===============================")
print("🔍 RESULTADO")
print("===============================")
print("\n", decision)
print("\nRazones técnicas:")
for r in razones:
    print(" -", r)
    
# ======================================================
# 6. GRÁFICAS DETALLADAS (4 PANELES)
# ======================================================

fig, axs = plt.subplots(4, 1, figsize=(14, 16))
fig.suptitle(f"Análisis Técnico Completo de {simbolo}", fontsize=18, fontweight='bold')

# ---------------------- 1. PRECIO + BOLLINGER ----------------------
axs[0].plot(df["Close"], label="Precio", color="black")
axs[0].plot(df["Upper"], label="Banda Superior", color="red", alpha=0.6)
axs[0].plot(df["Lower"], label="Banda Inferior", color="green", alpha=0.6)
axs[0].plot(df["SMA20"], label="SMA20", color="blue", alpha=0.7)
axs[0].set_title("Precio con Bandas de Bollinger", fontsize=14)
axs[0].legend()
axs[0].grid(True)

# ---------------------- 2. MACD ----------------------
axs[1].plot(df["MACD"], label="MACD", color="blue")
axs[1].plot(df["Signal"], label="Señal", color="orange")
axs[1].axhline(0, color="gray", linewidth=1)
axs[1].set_title("MACD y Señal", fontsize=14)
axs[1].legend()
axs[1].grid(True)

# ---------------------- 3. RSI ----------------------
axs[2].plot(df["RSI"], label="RSI", color="purple")
axs[2].axhline(30, color="green", linestyle="--", alpha=0.7, label="Sobreventa (30)")
axs[2].axhline(70, color="red", linestyle="--", alpha=0.7, label="Sobrecompra (70)")
axs[2].set_title("RSI (Índice de Fuerza Relativa)", fontsize=14)
axs[2].legend()
axs[2].grid(True)

# ---------------------- 4. SMA50 y SMA200 ----------------------
axs[3].plot(df["Close"], label="Precio", color="black", alpha=0.4)
axs[3].plot(df["Close"].rolling(50).mean(), label="SMA50", color="blue")
axs[3].plot(df["Close"].rolling(200).mean(), label="SMA200", color="red")
axs[3].set_title("Tendencia (SMA50 vs SMA200)", fontsize=14)
axs[3].legend()
axs[3].grid(True)

plt.tight_layout(rect=[0, 0, 1, 0.97])
plt.show()

# ---------------------- DISCLAIMER ----------------------
print("\n⚠️  AVISO IMPORTANTE:")
print("Este análisis es únicamente con fines informativos y educativos.")
print("No constituye recomendación de compra o venta de activos financieros.\n")
