# Informe Técnico: Funcionamiento del Tipo de Cambio (Banxico / SAT) en Odoo 20.0, SaaS (19.4) y <=19.0

---

## 1. Resumen

En el ecosistema de Odoo, el motor de tipo de cambio experimentó un **cambio arquitectónico** en la forma en que el sistema consulta y consume las tasas en transacciones contables (`res.currency`):

1. **En versiones tradicionales ($\le$ 19.0 LTS / On-premise):** El sistema opera bajo el esquema histórico: busca la tasa con fecha menor o igual (`<= date`) a la transacción y guarda la fecha exacta devuelta por el API de Banxico (fecha de publicación en el DOF).
2. **En la última versión SaaS (`saas-19.4`) y en Odoo 20.0:** El estándar global del núcleo cambió a buscar la tasa estrictamente anterior (`< date`) a la transacción (PR `odoo/odoo#231948`), con el objetivo de que las tasas apliquen a partir del día siguiente y no varíen durante el día.
3. **Ajuste para México en SaaS y v20.0:** Para compensar este cambio del núcleo y no desfasar la aplicación legal de las tasas del Diario Oficial de la Federación (DOF), el proveedor de Banxico ahora **desplaza la fecha de registro un día hacia atrás** (`effective_rate_date = foreign_rate_date - timedelta(days=1)`).

El resultado funcional es equivalente: **en todas las versiones una factura emitida en una fecha determinada toma la tasa oficial del DOF correspondiente**, pero en SaaS y v20.0 la fecha grabada en base de datos y la condición de búsqueda del ORM han cambiado sustancialmente.

---

## 2. Marco Normativo y Operativo en México

Conforme al Código Fiscal de la Federación (CFF Art. 20) y la Ley Monetaria de los Estados Unidos Mexicanos (Art. 8):
* El tipo de cambio aplicable para solventar obligaciones es el publicado en el **Diario Oficial de la Federación (DOF)** el día hábil bancario inmediato anterior al de la causación.
* Banxico publica en su API la serie oficial **`SF60653`** (*Tipo de cambio pesos por dólar E.U.A. para solventar obligaciones pagaderas en la República Mexicana fecha de publicación en el DOF*).
* La fecha entregada por el API corresponde a la fecha de publicación en el DOF.

---

## 3. Comparativa Técnica: $\le$ 19.0 vs SaaS (19.4) y Odoo 20.0

| Componente / Lógica | Odoo $\le$ 19.0 (LTS / On-premise) | Odoo SaaS (saas-19.4) y Odoo 20.0 | Justificación y Enlaces |
| :--- | :--- | :--- | :--- |
| **Búsqueda en BD (`_get_rates`)** | Fecha menor o igual (`<= date`). | Estrictamente menor (`< date`). | Estándar global de Odoo para evitar fluctuaciones intradía.<br>🔗 [PR odoo/odoo#231948](https://github.com/odoo/odoo/pull/231948) |
| **Fecha guardada (`res.currency.rate.name`)** | Guarda la fecha exacta devuelta por Banxico (fecha DOF). | Guarda la fecha devuelta por Banxico **menos 1 día**. | Al evaluar con `< date`, si no se restara un día, la factura ignoraría la tasa del día actual.<br>🔗 [Líneas 746-750 de res_config_settings.py](https://github.com/odoo/enterprise/blob/20.0/currency_rate_live/models/res_config_settings.py#L746-L750) |
| **Registro de MXN** | Inicializa con `Date.today()` en 19.0. | Inserta explícitamente `MXN` con valor 1.0 en la fecha efectiva calculada (`effective_rate_date`). | Garantiza paridad uniforme para la fecha de corte.<br>🔗 [Commit bce1db4c](https://github.com/odoo/enterprise/commit/bce1db4c) |
| **Configuración Proxy IAP** | Fijo a `PROXY_URL` en versiones previas. | Configurable mediante el parámetro del sistema `currency_rate_live.iap_proxy_url`. | Flexibilidad para pruebas y entornos locales.<br>🔗 [res_config_settings.py](https://github.com/odoo/enterprise/blob/20.0/currency_rate_live/models/res_config_settings.py) |

---

## 4. Referencias y Enlaces a Repositorios

Para auditoría técnica o revisión en el código fuente, consultar las siguientes referencias directas:

* **PR Core (Base / Community) — Cambio de regla `< date`:**  
  🔗 [odoo/odoo#231948: [IMP] base: Currency Rates Fetching](https://github.com/odoo/odoo/pull/231948)
* **PR Enterprise — Integración del cambio en módulos Enterprise:**  
  🔗 [odoo/enterprise#97967: [IMP] base: Currency Rates Fetching](https://github.com/odoo/enterprise/pull/97967)
* **PR Enterprise — Ajuste del desfase para Banxico:**  
  🔗 [odoo/enterprise#120366: [FIX] currency_rate_live: Bank of Mexico Correct Rate](https://github.com/odoo/enterprise/pull/120366)
* **Commit específico del ajuste en Banxico:**  
  🔗 [Commit bce1db4c en odoo/enterprise](https://github.com/odoo/enterprise/commit/bce1db4c)
* **Ubicación del desfase en SaaS (saas-19.4):**  
  🔗 [res_config_settings.py en saas-19.4](https://github.com/odoo/enterprise/blob/saas-19.4/currency_rate_live/models/res_config_settings.py)
* **Ubicación exacta del desfase en el archivo de v20.0:**  
  🔗 [res_config_settings.py#L746-L750 en odoo/enterprise (20.0)](https://github.com/odoo/enterprise/blob/20.0/currency_rate_live/models/res_config_settings.py#L746-L750)
* **Implementación previa tradicional en v19.0 LTS (sin desfase):**  
  🔗 [res_config_settings.py en odoo/enterprise (19.0)](https://github.com/odoo/enterprise/blob/19.0/currency_rate_live/models/res_config_settings.py)

---

## 5. Puntos Clave para el Equipo de Implementación y Soporte

1. **Diferenciación entre LTS y SaaS:**  
   Las bases de datos en versiones estables LTS tradicionales ($\le$ 19.0) mantienen la lógica clásica (`<= date`). En cambio, clientes alojados en Odoo Online / SaaS (como `saas-19.4`) y en la nueva versión mayor **Odoo 20.0** ya incorporan la nueva lógica (`< date`).
2. **Fecha visible en pantalla en SaaS y v20.0:**  
   En la interfaz de Odoo (`Contabilidad > Configuración > Monedas > Tasas`), los usuarios observarán que la tasa tiene registrada la fecha del día anterior al que efectivamente aplica. **Esto es el comportamiento esperado en SaaS y v20.0** y responde a la regla de consulta `< date`.
3. **Cargas manuales y migraciones hacia SaaS / v20.0:**  
   Si se importan tipos de cambio de manera manual o vía script en entornos SaaS o v20.0, **deben fecharse con un día de anticipación** a la fecha en que deben surtir efecto, o de lo contrario el sistema no las tomará en cuenta el día de la transacción. En versiones $\le$ 19.0 se cargan con la fecha del día en que deben surtir efecto.
4. **Certeza fiscal:**  
   En todos los entornos ($\le$ 19.0, SaaS y v20.0), las facturas y pólizas conservan el valor oficial que marca el DOF para la fecha correspondiente.
