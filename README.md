# Informe Técnico: Funcionamiento del Tipo de Cambio (Banxico / SAT) en Odoo 20.0 vs 17.0, 18.0 y 19.0

---

## 1. Resumen

En **Odoo 20.0**, el motor de tipo de cambio experimentó un **cambio arquitectónico global** en la forma en que el sistema consulta y consume las tasas en transacciones contables (`res.currency`):

1. **En Odoo 17.0, 18.0 y 19.0**, el sistema opera bajo el esquema tradicional: busca la tasa con fecha menor o igual (`<= date`) a la transacción y guarda la fecha exacta devuelta por el API de Banxico.
2. **En Odoo 20.0**, el estándar global del núcleo cambió a buscar la tasa estrictamente anterior (`< date`) a la transacción (PR `odoo/odoo#231948`), con el objetivo de que las tasas apliquen a partir del día siguiente y no varíen durante el día.
3. **Ajuste para México en v20.0:** Para compensar este cambio del núcleo y no desfasar la aplicación legal de las tasas del Diario Oficial de la Federación (DOF), el proveedor de Banxico en Odoo 20.0 ahora **desplaza la fecha de registro un día hacia atrás**.

El resultado funcional es equivalente: **en todas las versiones una factura emitida en una fecha determinada toma la tasa oficial del DOF correspondiente**, pero en v20.0 la fecha grabada en base de datos y la condición de búsqueda han cambiado sustancialmente.

---

## 2. Marco Normativo y Operativo en México

Conforme al Código Fiscal de la Federación (CFF Art. 20) y la Ley Monetaria de los Estados Unidos Mexicanos (Art. 8):
* El tipo de cambio aplicable para solventar obligaciones es el publicado en el **Diario Oficial de la Federación (DOF)** el día hábil bancario inmediato anterior al de la causación.
* Banxico publica en su API la serie oficial **`SF60653`** (*Tipo de cambio pesos por dólar E.U.A. para solventar obligaciones pagaderas en la República Mexicana fecha de publicación en el DOF*).
* La fecha entregada por el API corresponde a la fecha de publicación en el DOF.

---

## 3. Comparativa Técnica: v17.0 / v18.0 / v19.0 vs v20.0

| Componente / Lógica | Odoo 17.0, 18.0 y 19.0 | Odoo 20.0 | Justificación y Enlaces |
| :--- | :--- | :--- | :--- |
| **Búsqueda en BD (`_get_rates`)** | Fecha menor o igual (`<= date`). | Estrictamente menor (`< date`). | Estándar global de Odoo para evitar fluctuaciones intradía.<br>🔗 [PR odoo/odoo#231948](https://github.com/odoo/odoo/pull/231948) |
| **Fecha guardada (`res.currency.rate.name`)** | Guarda la fecha exacta devuelta por Banxico (fecha DOF). | Guarda la fecha devuelta por Banxico **menos 1 día**. | Al evaluar con `< date`, si no se restara un día, la factura ignoraría la tasa del día actual.<br>🔗 [Líneas 746-750 de res_config_settings.py en 20.0](https://github.com/odoo/enterprise/blob/20.0/currency_rate_live/models/res_config_settings.py#L746-L750) |
| **Registro de MXN** | En 17/18 sin línea base explícita; en 19.0 inicializa con `Date.today()`. | Inserta explícitamente `MXN` con valor 1.0 en la fecha efectiva calculada (`effective_rate_date`). | Garantiza paridad uniforme para la fecha de corte.<br>🔗 [Commit bce1db4c](https://github.com/odoo/enterprise/commit/bce1db4c) |
| **Configuración Proxy IAP** | URL fija a `iap-services.odoo.com`. | Configurable mediante el parámetro del sistema `currency_rate_live.iap_proxy_url`. | Flexibilidad para pruebas y entornos locales.<br>🔗 [res_config_settings.py en 20.0](https://github.com/odoo/enterprise/blob/20.0/currency_rate_live/models/res_config_settings.py) |

---

## 4. Referencias y Enlaces a Repositorios

Para auditoría técnica o revisión en el código fuente, consultar las siguientes referencias directas:

* **PR Core (Base / Community) — Cambio de regla `< date` (a partir de v20.0):**  
  🔗 [odoo/odoo#231948: [IMP] base: Currency Rates Fetching](https://github.com/odoo/odoo/pull/231948)
* **PR Enterprise — Integración del cambio en módulos Enterprise:**  
  🔗 [odoo/enterprise#97967: [IMP] base: Currency Rates Fetching](https://github.com/odoo/enterprise/pull/97967)
* **PR Enterprise — Ajuste del desfase para Banxico en v20.0:**  
  🔗 [odoo/enterprise#120366: [FIX] currency_rate_live: Bank of Mexico Correct Rate](https://github.com/odoo/enterprise/pull/120366)
* **Commit específico del ajuste en Banxico (v20.0):**  
  🔗 [Commit bce1db4c en odoo/enterprise](https://github.com/odoo/enterprise/commit/bce1db4c)
* **Ubicación exacta del desfase en el archivo de v20.0:**  
  🔗 [res_config_settings.py#L746-L750 en odoo/enterprise (20.0)](https://github.com/odoo/enterprise/blob/20.0/currency_rate_live/models/res_config_settings.py#L746-L750)
* **Implementación previa tradicional en v19.0 (sin desfase):**  
  🔗 [res_config_settings.py en odoo/enterprise (19.0)](https://github.com/odoo/enterprise/blob/19.0/currency_rate_live/models/res_config_settings.py)
* **Consulta SQL tradicional en Core v19.0 (`<= date`):**  
  🔗 [res_currency.py en odoo/odoo (19.0)](https://github.com/odoo/odoo/blob/19.0/odoo/addons/base/models/res_currency.py)

---

## 5. Puntos Clave para el Equipo de Implementación y Soporte

1. **Continuidad entre v17, v18 y v19:**  
   Las versiones 17.0, 18.0 y 19.0 funcionan exactamente con la misma mecánica (búsqueda `<= date` y fecha idéntica a la del DOF). La ruptura de compatibilidad ocurre exclusivamente al migrar hacia **Odoo 20.0**.
2. **Fecha visible en pantalla en v20.0:**  
   En la interfaz de Odoo 20.0 (`Contabilidad > Configuración > Monedas > Tasas`), los usuarios observarán que la tasa tiene registrada la fecha del día anterior al que efectivamente aplica. **Esto es el comportamiento esperado en v20.0** y responde a la regla de consulta `< date`.
3. **Cargas manuales y migraciones hacia v20.0:**  
   Si se importan tipos de cambio de manera manual o vía script en Odoo 20.0, **deben fecharse con un día de anticipación** a la fecha en que deben surtir efecto, o de lo contrario el sistema no las tomará en cuenta el día de la transacción. En 17, 18 y 19 se cargan con la fecha del día en que deben surtir efecto.
4. **Certeza fiscal:**  
   En todas las versiones (17.0, 18.0, 19.0 y 20.0), las facturas y pólizas conservan el valor oficial que marca el DOF para la fecha correspondiente.
