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
