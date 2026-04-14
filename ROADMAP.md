# Roadmap: MCP Server para SUA (IMSS)
> Para agente de IA — proyecto open source  
> Objetivo: generar archivos TXT/XML importables al SUA oficial  
> Stack sugerido: Python + FastMCP

---

## Contexto para el agente

El SUA (Sistema Único de Autodeterminación) es el software oficial del IMSS para que patrones calculen y paguen cuotas obrero-patronales e INFONAVIT. Está desarrollado en VB6 con base de datos Access (.mdb). El sistema acepta importación de datos mediante archivos de texto con formato fijo definido por el IMSS.

El MCP server actuará como una capa de herramientas que Claude (u otro LLM) puede invocar para: validar datos, construir los archivos de importación correctamente formateados, y guiar al usuario en el proceso.

---

## Fase 0 — Investigación y documentación de formatos (CRÍTICA)

**Esta fase debe hacerse ANTES de escribir código.**

### 0.1 Obtener especificaciones oficiales

Buscar y descargar:
- Manual Técnico del SUA (versión 3.x) — disponible en imss.gob.mx
- Documento "Estructura de archivos de importación SUA"
- Guía de llenado de archivos IDSE (Intercambio de Datos con el SUA Empresarial)

Los archivos que el SUA puede importar son:
- `TRAB*.txt` — Trabajadores
- `AFIL*.txt` — Datos afiliatorios
- `MOVT*.txt` — Movimientos (altas, bajas, modificaciones de salario)
- `INCA*.txt` — Incapacidades
- `CRED*.txt` — Movimientos de crédito INFONAVIT
- `OBRA*.txt` — Registro de obra (sector construcción)

### 0.2 Reverse engineering del SUA.mdb

Instalar mdbtools en Linux/Mac:
```bash
# Fedora
sudo dnf install mdbtools

# Ubuntu/Debian
sudo apt install mdbtools
```

Extraer esquema completo:
```bash
mdb-tables -1 SUA.mdb                    # lista de tablas
mdb-schema SUA.mdb postgres > schema.sql # esquema SQL
mdb-export SUA.mdb Patron > patron.csv   # datos de ejemplo
mdb-export SUA.mdb Trabajador > trab.csv
mdb-export SUA.mdb Parametros > params.csv
```

Tablas ya identificadas por ingeniería inversa del EXE:
- `Patron` — datos del patrón (REG_PAT, RFC_PAT, NOM_PAT, ...)
- `Trabajador` — catálogo de trabajadores (NSS, RFC, CURP, ...)
- `Prima_RT` — prima de riesgo de trabajo
- `Resultado_ConfrontaBim` — resultado de confronta bimestral
- `TablaUMA` — valores de UMA históricos
- `TablaValINF` — valores INFONAVIT
- `IndexTablaParametros` — parámetros del sistema

### 0.3 Documentar reglas de validación

Ya identificadas:
- RFC formato: `AAAA-BBBBBB-CCC` (4-6-3 caracteres)
- Registro Patronal: `AAA-BBBBB-CC-D` (3-5-2-1)
- NSS: 11 dígitos
- CURP: 18 caracteres
- Contraseñas: MD5
- Nombres: reemplazar `*` → `'` y `#` → `/`
- TDES (tipo descuento): valores `"1"`, `"2"`, `"3"`

---

## Fase 1 — Setup del proyecto

### 1.1 Estructura de repositorio

```
sua-mcp/
├── README.md
├── LICENSE                    # MIT o Apache 2.0
├── pyproject.toml
├── src/
│   └── sua_mcp/
│       ├── __init__.py
│       ├── server.py          # Punto de entrada MCP
│       ├── tools/
│       │   ├── __init__.py
│       │   ├── patrones.py    # Herramientas de patrón
│       │   ├── trabajadores.py
│       │   ├── movimientos.py
│       │   ├── incapacidades.py
│       │   ├── infonavit.py
│       │   └── prima_rt.py
│       ├── validators/
│       │   ├── __init__.py
│       │   ├── rfc.py
│       │   ├── curp.py
│       │   ├── nss.py
│       │   └── campos.py
│       ├── generators/
│       │   ├── __init__.py
│       │   ├── trabajadores_txt.py
│       │   ├── movimientos_txt.py
│       │   ├── incapacidades_txt.py
│       │   └── infonavit_txt.py
│       ├── models/
│       │   ├── __init__.py
│       │   ├── patron.py      # Pydantic models
│       │   ├── trabajador.py
│       │   ├── movimiento.py
│       │   └── incapacidad.py
│       └── db/
│           ├── __init__.py
│           └── mdb_reader.py  # Lector opcional de .mdb
├── tests/
│   ├── test_validators.py
│   ├── test_generators.py
│   └── fixtures/              # Archivos TXT de ejemplo reales
└── docs/
    ├── formatos/              # Documentación de cada formato
    └── ejemplos/
```

### 1.2 Dependencias

```toml
# pyproject.toml
[project]
name = "sua-mcp"
version = "0.1.0"
dependencies = [
    "fastmcp>=0.1.0",          # Framework MCP
    "pydantic>=2.0",           # Validación de modelos
    "python-stdnum>=1.17",     # Validación RFC, CURP, NSS mexicanos
    "mdbtools-python",         # Leer .mdb (opcional)
    "rich",                    # Output en consola
]

[project.optional-dependencies]
dev = ["pytest", "pytest-cov", "ruff", "mypy"]
```

---

## Fase 2 — Modelos de datos (Pydantic)

Implementar en `src/sua_mcp/models/`:

### Patrón
```python
class Patron(BaseModel):
    reg_pat: str          # Registro Patronal (validar formato AAA-BBBBB-CC-D)
    rfc: str              # RFC (validar con python-stdnum)
    nombre: str
    domicilio: str
    municipio: str
    cp: str
    entidad: str
    telefono: Optional[str]
    actividad_economica: str
    delegacion: str
    clase_riesgo: Literal["I", "II", "III", "IV", "V"]
    prima_rt: Decimal
```

### Trabajador
```python
class Trabajador(BaseModel):
    nss: str              # 11 dígitos
    rfc: str
    curp: str             # 18 caracteres
    nombre: str
    ap_paterno: str
    ap_materno: str
    fecha_nacimiento: date
    sexo: Literal["H", "M"]
    tipo_salario: Literal["F", "V", "M"]   # Fijo, Variable, Mixto
    salario_diario: Decimal
    reg_pat: str
    fecha_afiliacion: date
    umf: str
    tipo_trabajador: str
    pensionado: bool = False
```

### Movimiento
```python
class TipoMovimiento(str, Enum):
    ALTA = "08"
    BAJA = "02"
    MOD_SALARIO = "07"
    REINGREISO = "08"
    # ... completar con catálogo IMSS

class Movimiento(BaseModel):
    nss: str
    reg_pat: str
    tipo_movimiento: TipoMovimiento
    fecha: date
    salario_diario: Optional[Decimal]
    causa_baja: Optional[str]
```

---

## Fase 3 — Validadores

Implementar en `src/sua_mcp/validators/`:

### RFC (`rfc.py`)
- Estructura: 4 letras + 6 dígitos (fecha) + 3 homoclave
- Personas físicas: 13 caracteres
- Personas morales: 12 caracteres
- Usar `python-stdnum`: `from stdnum.mx import rfc`
- Validar dígito verificador
- Lista negra de RFC genéricos (XAXX010101000, etc.)

### CURP (`curp.py`)
- 18 caracteres con estructura definida
- Validar con `from stdnum.mx import curp`
- Verificar coherencia con nombre y fecha de nacimiento

### NSS (`nss.py`)
- 11 dígitos
- Algoritmo de dígito verificador (Luhn modificado del IMSS)
- Validar que la subdelegación en los primeros 2 dígitos sea válida

### Registro Patronal (`campos.py`)
- Formato: `AAA-BBBBB-CC-D`
- Los primeros 3 dígitos = delegación
- Validar dígito verificador

---

## Fase 4 — Generadores de archivos

Implementar en `src/sua_mcp/generators/`:

Cada generador toma una lista de modelos Pydantic y produce el TXT con formato fijo que el SUA espera.

### Formato general de archivos SUA
Los archivos son de **ancho fijo** (no CSV). Cada campo tiene posición y longitud exacta. Ejemplo de trabajadores:

```
Posición  Longitud  Campo
1         11        NSS
12        13        RFC
25        18        CURP
43        40        Nombre
83        40        Apellido Paterno
...
```

El agente DEBE obtener la especificación exacta del manual técnico del IMSS antes de implementar esto. Los formatos varían entre versiones del SUA (3.6.x, 3.7.x).

### Implementación base
```python
class TrabajadoresGenerator:
    def generate(self, trabajadores: list[Trabajador], reg_pat: str) -> str:
        lines = []
        for t in trabajadores:
            line = self._format_record(t)
            lines.append(line)
        return "\n".join(lines)
    
    def _format_record(self, t: Trabajador) -> str:
        # Cada campo en posición y longitud exacta
        # Rellenar con espacios a la derecha, números con ceros a la izquierda
        return (
            t.nss.zfill(11) +
            t.rfc.ljust(13) +
            t.curp.ljust(18) +
            # ...
        )
    
    def write_file(self, trabajadores: list[Trabajador], path: Path, reg_pat: str):
        content = self.generate(trabajadores, reg_pat)
        path.write_text(content, encoding="latin-1")  # SUA usa latin-1, NO utf-8
```

**IMPORTANTE:** El SUA usa encoding **Windows-1252 (latin-1)**, no UTF-8. Esto es crítico.

---

## Fase 5 — Herramientas MCP

Implementar en `src/sua_mcp/tools/`. Estas son las funciones que Claude invocará.

### `server.py` — punto de entrada

```python
from fastmcp import FastMCP
from .tools import patrones, trabajadores, movimientos, incapacidades, infonavit

mcp = FastMCP("SUA MCP Server")

# Registrar todas las herramientas
mcp.include(patrones.router)
mcp.include(trabajadores.router)
mcp.include(movimientos.router)
mcp.include(incapacidades.router)
mcp.include(infonavit.router)

if __name__ == "__main__":
    mcp.run()
```

### Herramientas a implementar

#### Patrón
- `sua_validar_patron(reg_pat, rfc, nombre, ...)` → valida y devuelve errores
- `sua_crear_patron(...)` → crea registro de patrón en estado local
- `sua_listar_patrones()` → lista patrones registrados

#### Trabajadores
- `sua_validar_trabajador(nss, rfc, curp, ...)` → valida todos los campos
- `sua_agregar_trabajador(...)` → agrega trabajador al patrón activo
- `sua_listar_trabajadores(reg_pat)` → lista trabajadores del patrón
- `sua_generar_archivo_trabajadores(reg_pat, output_path)` → genera TXT

#### Movimientos
- `sua_registrar_alta(nss, reg_pat, fecha, salario)` → registra alta
- `sua_registrar_baja(nss, reg_pat, fecha, causa)` → registra baja
- `sua_registrar_mod_salario(nss, reg_pat, fecha, nuevo_salario)` → modifica salario
- `sua_generar_archivo_movimientos(reg_pat, periodo, output_path)` → genera TXT

#### Incapacidades
- `sua_registrar_incapacidad(nss, fecha_ini, fecha_fin, tipo, folio_imss)` → registra
- `sua_generar_archivo_incapacidades(reg_pat, periodo, output_path)` → genera TXT

#### INFONAVIT
- `sua_registrar_credito(nss, num_credito, tipo_descuento, valor)` → registra crédito
- `sua_suspender_credito(nss, num_credito, fecha)` → suspende crédito
- `sua_generar_archivo_infonavit(reg_pat, periodo, output_path)` → genera TXT

#### Prima de Riesgo de Trabajo
- `sua_calcular_prima_rt(reg_pat, ano, casos, dias_subsidiados, dias_incapacidad)` → calcula prima
- `sua_generar_reporte_prima_rt(reg_pat, ano)` → reporte

#### Utilidades
- `sua_validar_rfc(rfc)` → valida RFC mexicano
- `sua_validar_curp(curp)` → valida CURP
- `sua_validar_nss(nss)` → valida NSS con dígito verificador
- `sua_validar_registro_patronal(reg_pat)` → valida formato y dígito
- `sua_obtener_uma(fecha)` → devuelve valor de UMA para una fecha
- `sua_obtener_inpc(periodo)` → devuelve INPC para un período
- `sua_calcular_cuotas(salario, uma, tipo_trabajador)` → calcula cuotas IMSS

---

## Fase 6 — Persistencia local

El MCP server necesita guardar estado entre llamadas. Opciones:

**Opción A (simple): SQLite**
```python
# Una DB SQLite local por patrón/empresa
# Schema espejado del SUA.mdb una vez que se obtenga
import sqlite3
# o
from sqlalchemy import create_engine
```

**Opción B (sin DB): JSON/TOML files**
- Más simple para empezar
- Un archivo por patrón

Recomendación: empezar con SQLite + SQLAlchemy. El schema se define en `Fase 0.2` una vez que tengas el `SUA.mdb`.

---

## Fase 7 — Testing

### Tests críticos (sin estos no se puede liberar)
- Validador RFC: casos válidos, inválidos, RFC genéricos
- Validador NSS: dígito verificador correcto e incorrecto
- Validador CURP: todos los casos borde
- Generadores: comparar output byte a byte con archivos reales del SUA
- Encoding: verificar que el output es latin-1 y no UTF-8

### Fixtures necesarios
Conseguir archivos TXT reales generados por el SUA para usarlos como referencia en tests. Pedir a un contador que exporte algunos datos de prueba (sin datos reales de personas).

---

## Fase 8 — Documentación y release

### README mínimo
- Qué es el SUA y para qué sirve este MCP
- Instalación: `uvx sua-mcp` o `pip install sua-mcp`
- Configuración en Claude Desktop (`claude_desktop_config.json`)
- Ejemplos de uso con Claude
- Limitaciones conocidas

### Configuración para Claude Desktop
```json
{
  "mcpServers": {
    "sua": {
      "command": "uvx",
      "args": ["sua-mcp"],
      "env": {
        "SUA_DATA_DIR": "/home/usuario/.sua-mcp"
      }
    }
  }
}
```

### Publicación
- GitHub: `github.com/tu-usuario/sua-mcp`
- PyPI: `pip install sua-mcp`
- Registrar en el MCP directory oficial de Anthropic

---

## Orden de implementación recomendado para el agente

1. **Fase 0** completa antes de tocar código — sin los formatos exactos nada funciona
2. Modelos Pydantic (`Fase 2`) — define el contrato de datos
3. Validadores (`Fase 3`) — son independientes y testables sueltos
4. Generadores con tests (`Fase 4 + 7`) — el core del proyecto
5. Herramientas MCP (`Fase 5`) — wrapper de todo lo anterior
6. Persistencia (`Fase 6`) — al final, cuando el modelo de datos está estable
7. Docs y release (`Fase 8`)

---

## Riesgos y puntos de atención

| Riesgo | Mitigación |
|---|---|
| Formatos de archivo cambian entre versiones del SUA | Soportar múltiples versiones, parametrizar |
| Encoding latin-1 con caracteres especiales (ñ, acentos) | Tests exhaustivos con nombres reales mexicanos |
| Validación de NSS con algoritmo IMSS no documentado | Conseguir casos de prueba reales (válidos e inválidos) |
| IMSS puede cambiar los formatos sin previo aviso | Versionar los formatos en el código, suscribirse a avisos del IMSS |
| Datos fiscales/laborales sensibles | Aclarar en README que el servidor es local, sin telemetría |

---

## Recursos

- Manual del SUA: https://www.imss.gob.mx/sua
- Especificación IDSE: buscar "Instructivo de llenado IDSE" en imss.gob.mx
- FastMCP docs: https://github.com/jlowin/fastmcp
- python-stdnum (RFC/CURP): https://arthurdejong.org/python-stdnum/
- mdbtools: https://github.com/mdbtools/mdbtools
