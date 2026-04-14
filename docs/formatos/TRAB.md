# Formato TRAB*.txt - Trabajadores

Basado en la estructura general de archivos de importación al SUA y en las convenciones del IMSS.
Cada línea corresponde a un trabajador.

## Estructura

| Posición | Longitud | Campo | Descripción |
|---|---|---|---|
| 1 | 11 | `nss` | Número de Seguridad Social. Rellenar con ceros a la izquierda. |
| 12 | 13 | `rfc` | Registro Federal de Contribuyentes. Rellenar con espacios a la derecha. |
| 25 | 18 | `curp` | Clave Única de Registro de Población. Rellenar con espacios a la derecha. |
| 43 | 40 | `nombre` | Nombre(s) del trabajador. Rellenar con espacios a la derecha. |
| 83 | 40 | `ap_paterno` | Apellido paterno. Rellenar con espacios a la derecha. |
| 123 | 40 | `ap_materno` | Apellido materno. Rellenar con espacios a la derecha. |

*(Nota: En un entorno de producción, estos formatos deben confirmarse con los manuales oficiales del IMSS)*
