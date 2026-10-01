# Informe Técnico: Funcionamiento del Tipo de Cambio (Banxico / SAT) en Odoo 20.0, SaaS (19.3) y <19

---

## 1. Resumen

En el ecosistema de Odoo, el motor de tipo de cambio experimentó un **cambio arquitectónico** en la forma en que el sistema consulta y consume las tasas en transacciones contables (`res.currency`):

1. **En versiones <19 (LTS / On-premise tradicionales):** El sistema opera bajo el esquema histórico: busca la tasa con fecha menor o igual (`<= date`) a la transacción y almacena la fecha exacta devuelta por el API de Banxico (fecha de publicación en el DOF).
2. **En la última versión SaaS (`saas-19.3`) y en Odoo 20.0:** El estándar global del núcleo cambió a buscar la tasa estrictamente anterior (`< date`) a la transacción (PR `odoo/odoo#231948`), con el objetivo de que las tasas apliquen a partir del día siguiente y no varíen durante el día.
3. **Ajuste para México en SaaS (19.3) y v20.0:** Para compensar este cambio del núcleo y no desfasar la aplicación legal de las tasas del Diario Oficial de la Federación (DOF), el proveedor de Banxico ahora **desplaza la fecha de registro un día hacia atrás** (`effective_rate_date = foreign_rate_date - timedelta(days=1)`).

> **Resultado Funcional:**  
> En todas las versiones (<19, SaaS 19.3 y 20.0), una factura emitida en una fecha determinada toma la tasa oficial del DOF correspondiente. La diferencia radica en que en SaaS y v20.0 la fecha guardada en base de datos y la consulta del ORM han cambiado sustancialmente.

---

## 2. Referencias y Enlaces a Repositorios

Para auditoría técnica o revisión en el código fuente, consultar las siguientes referencias directas:

* **PR Core (Base / Community) — Cambio de regla `< date`:**  
  🔗 [odoo/odoo#231948: [IMP] base: Currency Rates Fetching](https://github.com/odoo/odoo/pull/231948)
* **PR Enterprise — Integración del cambio en módulos Enterprise:**  
  🔗 [odoo/enterprise#97967: [IMP] base: Currency Rates Fetching](https://github.com/odoo/enterprise/pull/97967)
* **PR Enterprise — Ajuste del desfase para Banxico:**  
  🔗 [odoo/enterprise#120366: [FIX] currency_rate_live: Bank of Mexico Correct Rate](https://github.com/odoo/enterprise/pull/120366)
* **Commit específico del ajuste en Banxico:**  
  🔗 [Commit bce1db4c en odoo/enterprise](https://github.com/odoo/enterprise/commit/bce1db4c)
* **Ubicación del desfase en SaaS (saas-19.3):**  
  🔗 [res_config_settings.py#L746-L748 en saas-19.3](https://github.com/odoo/enterprise/blob/saas-19.3/currency_rate_live/models/res_config_settings.py#L746-L748)
* **Ubicación exacta del desfase en el archivo de v20.0:**  
  🔗 [res_config_settings.py#L746-L750 en 20.0](https://github.com/odoo/enterprise/blob/20.0/currency_rate_live/models/res_config_settings.py#L746-L750)

---

## 3. Puntos Clave para el Equipo de Implementación y Soporte

1. **Diferenciación entre LTS y SaaS:**  
   Las versiones tradicionales (<19) mantienen la lógica clásica (`<= date`). En cambio, clientes en Odoo Online / SaaS (como `saas-19.3`) y en la versión mayor **Odoo 20.0** ya incorporan la nueva lógica (`< date`).
2. **Fecha visible en pantalla en SaaS y v20.0:**  
   En la interfaz de Odoo (`Contabilidad > Configuración > Monedas > Tasas`), los usuarios observarán que la tasa tiene registrada la fecha del día anterior al que efectivamente aplica. **Esto es el comportamiento esperado en SaaS y v20.0** y responde a la regla de consulta `< date`.
3. **Cargas manuales y migraciones hacia SaaS / v20.0:**  
   Si se importan tipos de cambio de manera manual o vía script en entornos SaaS o v20.0, **deben fecharse con un día de anticipación** a la fecha en que deben surtir efecto, o de lo contrario el sistema no las tomará en cuenta el día de la transacción. En versiones <19 se cargan con la fecha del día en que deben surtir efecto.
4. **Certeza fiscal:**  
   En todos los entornos (<19, SaaS 19.3 y v20.0), las facturas y pólizas conservan el valor oficial que marca el DOF para la fecha correspondiente.

---

## 4. Ejemplo Práctico Comparativo

Supongamos el siguiente escenario real de facturación:
* **Fecha de la transacción:** 02 de Octubre de 2026.
* **Publicación Banxico en el DOF:** 02 de Octubre de 2026 a **$19.50 MXN / USD**.
* **Objetivo contable:** Emitir una factura por **$1,000 USD** en esa fecha.

### En Odoo <19 (Esquema Tradicional):
1. **Guardado en BD:** El cron guarda la tasa con fecha idéntica a la publicación:  
   `Fecha: 02/10/2026 | Tasa: 0.051282 (1 / 19.50)`
2. **Consulta al facturar:** Odoo busca con la condición `<= date`:  
   `WHERE name <= '2026-10-02'`
3. **Resultado:** Encuentra la tasa del 02/10/2026 $\rightarrow$ **Total: $19,500.00 MXN**.

### En SaaS (19.3) y Odoo 20.0 (Nuevo Esquema):
1. **Guardado en BD:** El cron resta 1 día antes de guardar:  
   `Fecha: 01/10/2026 | Tasa: 0.051282 (1 / 19.50)`
2. **Consulta al facturar:** Odoo busca con la condición `< date`:  
   `WHERE name < '2026-10-02'`
3. **Resultado:** Toma la tasa más reciente anterior al 02/Oct, que es la del 01/10/2026 (con los $19.50 del DOF) $\rightarrow$ **Total: $19,500.00 MXN**.

> **Conclusión del Ejemplo:**  
> El importe facturado en pesos es **exactamente el mismo ($19,500.00 MXN)** en ambos esquemas. El desplazamiento de fecha a `01/10/2026` en base de datos es el mecanismo técnico que garantiza que el nuevo operador `< date` tome la tasa correcta del día fiscal.
