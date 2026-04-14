# Reglas de Validación SUA

## RFC
- Estructura: 4 letras + 6 dígitos (fecha) + 3 homoclave
- Personas físicas: 13 caracteres
- Personas morales: 12 caracteres
- Validar dígito verificador
- Lista negra de RFC genéricos (XAXX010101000, etc.)

## CURP
- 18 caracteres con estructura definida
- Verificar coherencia con nombre y fecha de nacimiento

## NSS
- 11 dígitos
- Algoritmo de dígito verificador (Luhn modificado del IMSS)
- Validar que la subdelegación en los primeros 2 dígitos sea válida

## Registro Patronal
- Formato: `AAA-BBBBB-CC-D`
- Los primeros 3 dígitos = delegación
- Validar dígito verificador

## Otros
- Contraseñas: MD5
- Nombres: reemplazar `*` → `'` y `#` → `/`
- TDES (tipo descuento): valores `"1"`, `"2"`, `"3"`
