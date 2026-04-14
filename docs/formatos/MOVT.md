# Formato MOVT*.txt - Movimientos

Basado en la estructura general de archivos de importación al SUA y en las convenciones del IMSS.
Cada línea corresponde a un movimiento.

## Estructura

| Posición | Longitud | Campo | Descripción |
|---|---|---|---|
| 1 | 11 | `nss` | Número de Seguridad Social. |
| 12 | 2 | `tipo_movimiento` | Tipo de movimiento (02=Baja, 07=Modificación de salario, 08=Alta). |
| 14 | 8 | `fecha` | Fecha del movimiento en formato DDMMAAAA. |
| 22 | 11 | `salario_diario` | Salario diario. (Formato monetario, revisar docs). |

*(Nota: En un entorno de producción, estos formatos deben confirmarse con los manuales oficiales del IMSS)*
