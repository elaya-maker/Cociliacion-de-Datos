# Cociliacion-de-Datos
import pandas as pd

# 1. Cargar el Libro de Ventas (Saltando los encabezados decorativos)
# El archivo 'Libro.csv' tiene datos reales a partir de la fila 5
df_libro = pd.read_csv('01-FEBECA LVF 16-07-2025 AL 31-07-2025.xlsx - Libro .csv', skiprows=4)

# 2. Cargar el Listado Contable
df_conta = pd.read_csv('01-FEBECA LVF 16-07-2025 AL 31-07-2025.xlsx - 213041006.csv')

# --- LIMPIEZA DE DATOS ---
# Limpiar nombres de columnas
df_libro.columns = df_libro.columns.str.strip()
df_conta.columns = df_conta.columns.str.strip()

# Normalizar RIF (Quitar espacios y poner en mayúsculas)
df_libro['Num. R.I.F.'] = df_libro['Num. R.I.F.'].str.strip().str.upper()
df_conta['Nit'] = df_conta['Nit'].str.strip().str.upper()

# --- CONCILIACIÓN ---
# Agrupar Libro por RIF (Sumamos 'IVA Retenido por Comprador')
ventas_agrupado = df_libro.groupby('Num. R.I.F.')['IVA Retenido por Comprador'].sum().reset_index()

# Agrupar Contabilidad por RIF (Sumamos 'Débitos Local' ya que las retenciones suelen cargarse ahí)
# Nota: Verifica si en tu caso es 'Débitos Local' o 'Créditos Local'
conta_agrupado = df_conta.groupby('Nit')['Débitos Local'].sum().reset_index()

# Cruzar ambos datos
conciliacion = pd.merge(
    ventas_agrupado, 
    conta_agrupado, 
    left_on='Num. R.I.F.', 
    right_on='Nit', 
    how='outer'
).fillna(0)

# Calcular diferencia
conciliacion['Diferencia'] = conciliacion['IVA Retenido por Comprador'] - conciliacion['Débitos Local']

# Filtrar solo lo que no cuadra (margen de 0.01 por decimales)
diferencias = conciliacion[conciliacion['Diferencia'].abs() > 0.01]

# Renombrar para mayor claridad
diferencias = diferencias.rename(columns={
    'Num. R.I.F.': 'RIF',
    'IVA Retenido por Comprador': 'Monto_Libro',
    'Débitos Local': 'Monto_Conta'
})

print("### REPORTE DE DIFERENCIAS ENCONTRADAS ###")
print(diferencias[['RIF', 'Monto_Libro', 'Monto_Conta', 'Diferencia']])

# Guardar a Excel para revisar
# diferencias.to_excel('diferencias_retenciones.xlsx', index=False)
import streamlit as st
import pandas as pd
import io

# Configuración de la página
st.set_page_config(page_title="Conciliador de Retenciones", layout="wide")

st.title("📊 Conciliación de Retenciones IVA")

# --- FUNCIONES DE PROCESAMIENTO (CON CACHÉ PARA EVITAR ERRORES DE DOM) ---
@st.cache_data
def procesar_archivos(file_libro, file_conta):
    # 1. Cargar Libro de Ventas (Salto de 4 filas según tu archivo)
    df_libro = pd.read_csv(file_libro, skiprows=4)
    df_libro.columns = df_libro.columns.str.strip()
    
    # 2. Cargar Contabilidad
    df_conta = pd.read_csv(file_conta)
    df_conta.columns = df_conta.columns.str.strip()

    # 3. Normalizar RIF (Eliminar guiones, espacios y asegurar mayúsculas)
    def clean_rif(text):
        if pd.isna(text): return ""
        return str(text).replace("-", "").replace(" ", "").upper()

    df_libro['RIF_KEY'] = df_libro['Num. R.I.F.'].apply(clean_rif)
    df_conta['RIF_KEY'] = df_conta['Nit'].apply(clean_rif)

    # 4. Agrupar montos
    # En el libro sumamos 'IVA Retenido por Comprador'
    libro_agrupado = df_libro.groupby('RIF_KEY')['IVA Retenido por Comprador'].sum().reset_index()
    
    # En contabilidad sumamos 'Débitos Local'
    conta_agrupado = df_conta.groupby('RIF_KEY')['Débitos Local'].sum().reset_index()

    # 5. Cruzar datos
    df_final = pd.merge(
        libro_agrupado, 
        conta_agrupado, 
        on='RIF_KEY', 
        how='outer'
    ).fillna(0)

    # 6. Calcular diferencias
    df_final['Diferencia'] = df_final['IVA Retenido por Comprador'] - df_final['Débitos Local']
    
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
