import pandas as pd

def conciliar_contabilidad(file_libro, file_conta):
    # 1. Cargar datos
    df_libro = pd.read_csv(file_libro)
    df_conta = pd.read_csv(file_conta)

    # Limpieza de nombres de columnas
    df_libro.columns = df_libro.columns.str.strip()
    df_conta.columns = df_conta.columns.str.strip()

    # 2. Función de limpieza de RIF (Normalización)
    def clean_rif(text):
        if pd.isna(text): return ""
        return str(text).replace("-", "").replace(" ", "").upper()

    # 3. Función para limpiar montos (Convertir "1.000,50" a float 1000.50)
    def clean_currency(value):
        if isinstance(value, str):
            # Quitamos puntos de miles y cambiamos coma decimal por punto
            value = value.replace('.', '').replace(',', '.')
        return pd.to_numeric(value, errors='coerce') or 0.0

    # Aplicar limpiezas
    df_libro['RIF_KEY'] = df_libro['Num. R.I.F.'].apply(clean_rif)
    df_conta['RIF_KEY'] = df_conta['Nit'].apply(clean_rif)
    
    # Limpiar columnas de montos antes de agrupar
    df_libro['IVA Retenido por Comprador'] = df_libro['IVA Retenido por Comprador'].apply(clean_currency)
    df_conta['Débitos Local'] = df_conta['Débitos Local'].apply(clean_currency)

    # 4. Agrupar montos por RIF
    libro_agrupado = df_libro.groupby('RIF_KEY')['IVA Retenido por Comprador'].sum().reset_index()
    conta_agrupado = df_conta.groupby('RIF_KEY')['Débitos Local'].sum().reset_index()

    # 5. Cruzar datos (Outer join para no perder registros de ningún lado)
    df_final = pd.merge(
        libro_agrupado, 
        conta_agrupado, 
        on='RIF_KEY', 
        how='outer'
    ).fillna(0)

    # 6. Calcular diferencias y redondear a 2 decimales
    df_final['Diferencia'] = (df_final['IVA Retenido por Comprador'] - df_final['Débitos Local']).round(2)
    
    # Renombrar para mayor claridad en el reporte
    df_final.columns = ['RIF', 'Monto_Libro', 'Monto_Contabilidad', 'Diferencia']

    return df_final
# --- INTERFAZ DE USUARIO ---
st.info("Sube tus archivos para comenzar. El sistema recordará los datos aunque cambies de pestaña.")

col1, col2 = st.columns(2)
with col1:
    file_l = st.file_uploader("Subir Libro de Ventas (CSV)", type="csv", key="u_libro")
with col2:
    file_c = st.file_uploader("Subir Listado Contable 213041006 (CSV)", type="csv", key="u_conta")

if file_l and file_c:
    # Procesamiento
    with st.spinner("Conciliando datos..."):
        df_resultado = procesar_archivos(file_l, file_c)

    # Pestañas
    tab1, tab2 = st.tabs(["📌 Resumen de Diferencias", "📋 Datos Completos"])

    with tab1:
        st.subheader("Discrepancias Detectadas")
        # Mostramos solo los que tienen diferencia significativa (> 0.01)
        descuadres = df_resultado[df_resultado['Diferencia'].abs() > 0.01].copy()
        
        if not descuadres.empty:
            st.warning(f"Se encontraron {len(descuadres)} RIF con diferencias.")
            st.dataframe(descuadres, use_container_width=True, key="tabla_errores")
        else:
            st.success("¡Todo cuadra perfectamente!")

    with tab2:
        st.subheader("Vista General de Conciliación")
        st.dataframe(df_resultado, use_container_width=True, key="tabla_total")
else:
    st.warning("Por favor, sube ambos archivos para generar la conciliación.")
