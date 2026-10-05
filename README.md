import streamlit as st
import pandas as pd
from datetime import datetime
from streamlit_gsheets import GSheetsConnection

# Configuración de pantalla ligera y responsiva para las tabletas de 4GB de RAM
st.set_page_config(page_title="Control de Tiempos y Producción", layout="wide")
st.title("🏭 Sistema de Tiempos y Control - Lentes Frames")

# ---------------------------------------------------------
# CONEXIÓN INTEGRADA CON LA HOJA DE EXCEL EN LA NUBE
# ---------------------------------------------------------
try:
    conn = st.connection("gsheets", type=GSheetsConnection)
    df_excel = conn.read()
except:
    if "db_respaldo" not in st.session_state:
        st.session_state.db_respaldo = pd.DataFrame()
    df_excel = st.session_state.db_respaldo

# Mantener el registro temporal de los cronómetros en la tableta del operador
if "inicio_orden" not in st.session_state:
    st.session_state.inicio_orden = None
if "pausa_orden" not in st.session_state:
    st.session_state.pausa_orden = None
if "tiempo_muerto_acumulado" not in st.session_state:
    st.session_state.tiempo_muerto_acumulado = 0

# Menú de navegación lateral
rol = st.sidebar.radio("Selecciona tu Rol:", ["Operador (Tableta)", "Supervisor (Reporte de Tiempos y Merma)"])

# ==========================================
# 1. INTERFAZ DEL OPERADOR (TABLETA EN PISO)
# ==========================================
if rol == "Operador (Tableta)":
    st.header("📋 Panel Operativo de la Maquinaria")
    
    # 👥 Listas Desplegables de Personal (Puedes cambiar estos nombres por los reales)
    lista_operadores = ["Juan Pérez", "María Rodríguez", "Carlos Gómez", "Ana Martínez", "Luis Hernández", "Sofía López"]
    lista_tecnicos = ["Ninguno / No requerido", "Ing. Ricardo Silva", "Ing. Fernando Torres", "Téc. Javier Méndez"]
    
    col_maq, col_ope, col_tec = st.columns(3)
    with col_maq:
        maquina = st.selectbox("Selecciona tu Máquina:", [f"TJ{str(i).zfill(2)}" for i in range(1, 12)])
    with col_ope:
        operador = st.selectbox("Selecciona tu Nombre (Operador):", lista_operadores)
    with col_tec:
        tecnico = st.selectbox("Técnico Responsable en Turno:", lista_tecnicos)
    
    st.divider()
    
    # 1. Selección del Área / Paso del proceso
    st.subheader("📍 1. Área / Paso del Proceso")
    paso_actual = st.radio(
        "¿En qué área vas a procesar esta orden?",
        ["Arbug (Moldeo)", "Asb (Ensamble/Proceso)", "Calidad (Inspección Final)"],
        horizontal=True
    )
    
    st.divider()
    st.subheader("🔍 2. Datos de la Orden (Escanee con Scanner Físico)")
    
    col1, col2, col3 = st.columns(3)
    with col1:
        po = st.text_input("Escanear Production Order (PO):", placeholder="Ej: 1874272966")
    with col2:
        cantidad_original_str = st.text_input("Escanear Cantidad de la Orden (Qty):", placeholder="Ej: 400")
    with col3:
        stock_id = st.text_input("Escanear Stock ID:", placeholder="Ej: 00200307762131")
        
    col4, col5 = st.columns(2)
    with col4:
        material = st.text_input("Escanear Material:", placeholder="Ej: 1FR0402A40")
    with col5:
        grid = st.text_input("Escanear Grid:", placeholder="Ej: BK172 AA")

    st.divider()

    # 3. Control de Tiempos mediante Botones
    st.subheader("⏱️ 3. Control de Estado de la Orden")
    st.write("Presiona el botón correspondiente para registrar los tiempos en tu área actual.")
    
    c_ini, c_pau, c_ter = st.columns(3)
    
    with c_ini:
        if st.button("▶️ INICIAR ORDEN", use_container_width=True, type="secondary"):
            st.session_state.inicio_orden = datetime.now()
            st.session_state.pausa_orden = None
            st.session_state.tiempo_muerto_acumulado = 0
            st.success(f"Orden {po} iniciada en {paso_actual} a las {st.session_state.inicio_orden.strftime('%H:%M:%S')}")

    with c_pau:
        if st.button("⏸️ PAUSAR ORDEN", use_container_width=True):
            if st.session_state.inicio_orden is not None:
                st.session_state.pausa_orden = datetime.now()
                st.warning("Orden en pausa. Registra el motivo del tiempo muerto abajo.")
            else:
                st.error("Primero debes dar clic en 'INICIAR ORDEN'")

    # Menú dinámico que aparece si la máquina está pausada
    motivo_paro = "Ninguno / Trabajando"
    if st.session_state.pausa_orden is not None:
        st.subheader("🚨 Registro de Tiempo Muerto")
        motivo_paro = st.selectbox(
            "Selecciona la causa del paro de maquinaria:",
            [
                "🔴 Detenida (Mal corte de navaja)", 
                "🔴 Detenida (Falla mecánica)", 
                "🛠️ Enviada a Tool Room", 
                "🔄 Cambio de molde", 
                "👥 Falta de Personal", 
                "👨‍🔧 Falta de Técnicos", 
                "Otro motivo"
            ]
        )
        minutos_muertos_actuales = int((datetime.now() - st.session_state.pausa_orden).total_seconds() / 60)
        st.caption(f"⏱️ Tiempo muerto corriendo en esta pausa: {minutos_muertos_actuales} min.")

    st.divider()

    # 4. Registro de Calidad y Merma
    st.subheader("⚠️ 4. Registro de Piezas Malas (Merma)")
    tiene_defectos = st.checkbox("¿Salieron piezas defectuosas en esta orden?")
    defecto = "Ninguno"
    cant_defecto = 0
    if tiene_defectos:
        col_def, col_cant_def = st.columns(2)
        with col_def:
            defecto = st.selectbox("Defecto de la pieza:", ["Burbuja", "Blush", "Pitting", "Incompletas", "Rayas", "Silver", "Otro"])
        with col_cant_def:
            cant_defecto = st.number_input("Cantidad de piezas afectadas:", min_value=1, value=1, step=1)

    # Cálculos matemáticos instantáneos para el operador
    qty_inicial = int(cantidad_original_str) if (cantidad_original_str and cantidad_original_str.isdigit()) else 0
    qty_buenas = qty_inicial - cant_defecto

    if qty_inicial > 0:
        st.info(f"📊 **Resumen del Lote:** Cantidad Escaneada: {qty_inicial} | Piezas Malas: {cant_defecto} | **Piezas Buenas Netas: {qty_buenas}**")

    # Botón Finalizador que envía los datos calculados a tu Excel
    with c_ter:
        if st.button("✅ TERMINAR ORDEN", use_container_width=True, type="primary"):
            if st.session_state.inicio_orden is not None:
                hora_fin = datetime.now()
                
                # Calcular minutos totales en la estación
                minutos_totales = int((hora_fin - st.session_state.inicio_orden).total_seconds() / 60)
                
                # Si se termina directo desde el estado de pausa, acumular el tiempo muerto final
                if st.session_state.pausa_orden is not None:
                    st.session_state.tiempo_muerto_acumulado += int((hora_fin - st.session_state.pausa_orden).total_seconds() / 60)
                
                # Calcular producción neta restando el tiempo muerto
                minutos_trabajados_netos = max(0, minutos_totales - st.session_state.tiempo_muerto_acumulado)

                # Estructura exacta que se acopla a tu hoja de Excel
                nuevo_registro = pd.DataFrame([{
                    "Fecha_Hora_Termino": hora_fin.strftime("%Y-%m-%d %H:%M:%S"),
                    "Maquina": maquina,
                    "Operador": operador,
                    "Tecnico_Asignado": tecnico,
                    "Area_Paso": paso_actual,
                    "PO": po,
                    "Cantidad_Original": qty_inicial,
                    "Cantidad_Defectuosa": cant_defecto,
                    "Cantidad_Buenas_Netas": qty_buenas,
                    "Defecto_Tipo": defecto,
                    "Stock_ID": stock_id,
                    "Material": material,
                    "Grid": grid,
                    "Motivo_Tiempo_Muerto": motivo_paro,
                    "Minutos_Totales_Area": minutos_totales,
                    "Minutos_Tiempo_Muerto": st.session_state.tiempo_muerto_acumulado,
                    "Minutos_Produccion_Neta": minutos_trabajados_netos
                }])
                
                try:
                    df_actualizado = pd.concat([df_excel, nuevo_registro], ignore_index=True)
                    conn.update(data=df_actualizado)
                    st.balloons()
                    st.success(f"¡Orden {po} completada en {paso_actual}! Datos guardados en el Excel.")
                    
                    # Reiniciar cronómetros internos de la tableta
                    st.session_state.inicio_orden = None
                    st.session_state.pausa_orden = None
                    st.session_state.tiempo_muerto_acumulado = 0
                except:
                    st.session_state.db_respaldo = pd.concat([st.session_state.db_respaldo, nuevo_registro], ignore_index=True)
                    st.success("¡Datos respaldados en la memoria de la tableta con éxito!")
            else:
                st.error("No puedes terminar una orden que no ha sido iniciada. Presiona 'INICIAR ORDEN' primero.")

# ==========================================
# 2. INTERFAZ DEL SUPERVISOR (REPORTES EN VIVO)
# ==========================================
elif rol == "Supervisor (Reporte de Tiempos y Merma)":
    st.header("📊 Panel de Eficiencia de Tiempos y Rendimiento")
    
    if df_excel.empty:
        st.info("Esperando que los operadores procesen órdenes en piso para desplegar las gráficas dinámicas.")
    else:
        tab1, tab2, tab3 = st.tabs(["⏱️ Tiempos de Producción vs Paros", "📉 Análisis de Defectos", "📦 Auditoría de Órdenes"])
        
        with tab1:
            st.subheader("Minutos de Tiempo Muerto Acumulado por Maquinaria")
            tiempo_muerto_maq = df_excel.groupby("Maquina")["Minutos_Tiempo_Muerto"].sum().reset_index()
            st.bar_chart(data=tiempo_muerto_maq, x="Maquina", y="Minutos_Tiempo_Muerto")
            
            st.subheader("Minutos Netos de Trabajo Efectivo")
            tiempo_produccion_maq = df_excel.groupby("Maquina")["Minutos_Produccion_Neta"].sum().reset_index()
st.bar_chart(data=tiempo_produccion_maq, x="Maquina", y="Minutos_Produccion_Neta")
with tab2:
st.subheader("Métricas de Defectos (Merma Acumulada)")
df_mermas = df_excel[df_excel["Defecto_Tipo"] != "Ninguno"]
if not df_mermas.empty:
merma_por_defecto = df_mermas.groupby("Defecto_Tipo")["Cantidad_Defectuosa"].sum().reset_index()
st.bar_chart(data=merma_por_defecto, x="Defecto_Tipo", y="Cantidad_Defectuosa")
else:
st.success("✅ Excelente calidad: Cero piezas defectuosas registradas en los lotes.")
with tab3:
st.subheader("Historial Completo Calculado")
st.dataframe(df_excel[[
"Fecha_Hora_Termino", "PO", "Area_Paso", "Maquina", "Operador",
"Cantidad_Original", "Cantidad_Buenas_Netas", "Minutos_Totales_Area", "Minutos_Tiempo_Muerto", "Minutos_Produccion_Neta", "Motivo_Tiempo_Muerto"
]], use_container_width=True)