# 1. Práctica 1. Construir un pipeline de recuperación utilizando Semantic Chunking y Embeddings sobre una base documental técnica.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 35 minutos |
| Complejidad | Media |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica construirás la base de recuperación semántica para un asistente técnico de soporte. Implementarás un pipeline que ingesta documentos Markdown, identifica encabezados y secciones, genera fragmentos semánticos trazables, calcula embeddings con `text-embedding-3-small` y recupera evidencia relevante mediante similitud coseno.

El resultado será un artefacto local en formato JSON Lines y un `KnowledgeRetrievalSkill` reutilizable por el harness del laboratorio anterior. Este laboratorio implementa la capa vectorial de una recuperación híbrida; en una evolución GraphRAG, los fragmentos recuperados podrán complementarse con entidades, relaciones y recorridos controlados de un grafo.

## Objetivos de aprendizaje

Al finalizar la práctica, podrás:

- [ ] Ingerir documentación técnica versionada desde `data/documents/`.
- [ ] Dividir documentos Markdown en fragmentos semánticos preservando encabezados, secciones y metadatos.
- [ ] Limitar los fragmentos a 450 tokens con un solapamiento de hasta 60 tokens.
- [ ] Generar embeddings con el modelo `text-embedding-3-small` y almacenarlos en JSON Lines.
- [ ] Recuperar fragmentos relevantes mediante similitud coseno e integrarlos en un `KnowledgeRetrievalSkill`.

## Prerrequisitos

### Conocimientos

- Conocimiento general de embeddings, similitud coseno y recuperación aumentada por generación (RAG).
- Conocimiento básico de JSON, JSON Lines y Python.
- Familiaridad con encabezados Markdown (`#`, `##`, `###`) y metadatos documentales.
- Haber completado y validado el harness del laboratorio `02-00-04`.

### Acceso y configuración requerida

- Repositorio local funcional en `~/genai-agent-labs`.
- Entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Archivo `.env` con una clave válida de OpenAI:

```dotenv
OPENAI_API_KEY=tu_clave_de_openai
```

- Acceso de red a la API de OpenAI.
- Cuota disponible para el modelo `text-embedding-3-small`.

> **Importante:** no agregues el archivo `.env` al repositorio. La clave nunca debe aparecer en código fuente, archivos JSONL versionados, salidas de consola compartidas ni capturas de pantalla.

## Entorno del laboratorio

### Recursos recomendados

| Recurso | Mínimo |
|---|---:|
| Procesador | 4 núcleos |
| Memoria RAM | 16 GB |
| Espacio libre en disco | 10 GB |
| Sistema operativo de referencia | Ubuntu 22.04.4 LTS |
| Python | 3.12.x |

### Dependencias principales

| Paquete | Versión recomendada | Uso |
|---|---:|---|
| `openai` | 1.12.0 o compatible | Generación de embeddings |
| `tiktoken` | 0.7.0 | Conteo de tokens |
| `pydantic` | 2.7.4 o compatible | Validación de modelos |
| `python-dotenv` | 1.0.1 | Carga segura de variables desde `.env` |
| `pytest` | Versión compatible | Pruebas automatizadas |

### Preparación inicial

1. Abre una terminal y accede al directorio obligatorio del curso:

```bash
cd ~/genai-agent-labs
```

2. Activa el entorno virtual compartido:

```bash
source .venv/bin/activate
```

3. Confirma la versión de Python:

```bash
python --version
```

4. Instala las dependencias requeridas si aún no están disponibles:

```bash
pip install "openai==1.12.0" "tiktoken==0.7.0" "pydantic==2.7.4" "python-dotenv==1.0.1" pytest
```

5. Crea la estructura de directorios del laboratorio:

```bash
mkdir -p data/documents
mkdir -p data/processed
mkdir -p src/skills
mkdir -p tests
mkdir -p reports
```

6. Verifica que `.env` esté ignorado por Git:

```bash
grep -qxF ".env" .gitignore || echo ".env" >> .gitignore
git status --short
```

**Resultado esperado**

La terminal debe mostrar una versión de Python 3.12.x y no debe mostrar el archivo `.env` como pendiente de confirmación.

**Verificación**

Ejecuta:

```bash
test -f .env && echo ".env encontrado"
test -d data/documents && echo "Directorio documental preparado"
test -d data/processed && echo "Directorio procesado preparado"
```

---

## Procedimiento paso a paso

### Paso 1. Crear la base documental técnica

**Objetivo**

Crear documentos Markdown técnicos con secciones explícitas que representen guías de API, despliegue, autenticación y gestión de incidencias para el agente de soporte.

**Instrucciones**

1. Crea el documento de guía de API:

```bash
cat > data/documents/api-guide.md <<'EOF'
# Guía de API del agente de soporte

## Autenticación de solicitudes

La API del agente de soporte requiere autenticación mediante un token Bearer.
El cliente debe enviar el encabezado Authorization con el formato `Bearer <token>`.
Los tokens no deben almacenarse en repositorios, imágenes de contenedor ni archivos de configuración versionados.

## Endpoint de consulta

El endpoint `POST /support/query` recibe una pregunta técnica y un identificador de solicitud.
La respuesta incluye una respuesta textual, las fuentes recuperadas y un identificador de trazabilidad.
Las solicitudes deben usar JSON y el encabezado `Content-Type: application/json`.

## Manejo de errores

El código HTTP 401 indica que el token es inválido o no fue enviado.
El código HTTP 403 indica que la identidad autenticada no tiene permisos para ejecutar la operación.
El código HTTP 429 indica que se excedió un límite de solicitudes y el cliente debe aplicar reintentos con espera exponencial.
EOF
```

2. Crea el documento de despliegue:

```bash
cat > data/documents/deployment-runbook.md <<'EOF'
# Runbook de despliegue del agente de soporte

## Variables de entorno

La aplicación obtiene secretos y configuraciones desde variables de entorno.
La variable OPENAI_API_KEY debe configurarse como secreto en el entorno de despliegue.
La variable OPENAI_MODEL puede definir el modelo de generación permitido por la aplicación.
Ninguna clave de API debe incluirse en el Dockerfile, en la imagen de contenedor o en el código fuente.

## Despliegue en Azure Container Apps

La aplicación se despliega como una Azure Container App.
La identidad administrada de la aplicación debe recibir únicamente los roles necesarios.
Para consultar secretos administrados, se debe asignar el rol mínimo que permita la operación requerida.
Los cambios de configuración deben validarse primero en un entorno de pruebas.

## Validación posterior al despliegue

Después del despliegue, consulte el endpoint `GET /health`.
El endpoint debe responder HTTP 200 y un cuerpo JSON con el estado `ok`.
Después, envíe una consulta de prueba a `POST /support/query` y valide que la respuesta incluya fuentes.
EOF
```

3. Crea el documento de política de autenticación:

```bash
cat > data/documents/authentication-policy.md <<'EOF'
# Política de autenticación y autorización

## Principio de mínimo privilegio

Las identidades humanas, identidades administradas y cuentas de servicio deben recibir solamente los permisos necesarios.
Los permisos se asignan mediante control de acceso basado en roles, también llamado RBAC.
No se deben usar credenciales compartidas para operaciones de producción.

## Identidad administrada

Una Azure Container App puede usar identidad administrada para autenticarse contra servicios de Azure.
La identidad administrada elimina la necesidad de almacenar secretos de Azure en la aplicación.
La asignación de roles debe realizarse en el ámbito más reducido posible, por ejemplo, un recurso específico.

## Gestión de secretos

Los secretos de terceros, como OPENAI_API_KEY, deben cargarse desde secretos de Container Apps o un almacén de secretos autorizado.
Los secretos nunca deben escribirse en registros de aplicación.
Cuando exista sospecha de exposición, la clave debe revocarse y reemplazarse inmediatamente.
EOF
```

4. Crea el documento de incidencias:

```bash
cat > data/documents/incident-runbook.md <<'EOF'
# Runbook de incidencias del agente de soporte

## Error 401 al invocar la API

Si el cliente recibe HTTP 401, confirme que el encabezado Authorization está presente.
Valide que el token tenga el formato Bearer y que no haya expirado.
No escriba el token completo en tickets, registros ni mensajes de chat.

## Error 429 del proveedor de modelos

Si el proveedor devuelve HTTP 429, reduzca la frecuencia de solicitudes.
Implemente reintentos con espera exponencial y un límite máximo de intentos.
Registre el código de estado, el identificador de solicitud y la latencia, pero no el contenido de secretos.

## Investigación de una respuesta sin fuentes

Si una respuesta no contiene fuentes recuperadas, valide que el archivo de embeddings exista.
Confirme que la consulta genera un embedding y que el umbral de similitud no sea demasiado restrictivo.
Revise que cada fragmento conserve document_id, source_path, section_title y chunk_id.
EOF
```

5. Revisa los documentos creados:

```bash
find data/documents -type f -name "*.md" -print
```

**Resultado esperado**

Deben existir cuatro archivos Markdown en `data/documents/`, cada uno con encabezados y secciones técnicas.

**Verificación**

Ejecuta:

```bash
grep -R "^## " data/documents
```

La salida debe listar secciones tales como `Autenticación de solicitudes`, `Variables de entorno`, `Identidad administrada` y `Error 429 del proveedor de modelos`.

---

### Paso 2. Implementar el chunker semántico

**Objetivo**

Implementar una división semántica que conserve el título de la sección, respete los límites documentales y genere fragmentos de máximo 450 tokens con solapamiento de 60 tokens.

**Instrucciones**

1. Crea el archivo `src/semantic_chunker.py`:

```bash
cat > src/semantic_chunker.py <<'PY'
from __future__ import annotations

import re
from dataclasses import dataclass
from pathlib import Path

import tiktoken


MAX_TOKENS = 450
OVERLAP_TOKENS = 60
ENCODING_NAME = "cl100k_base"


@dataclass
class Section:
    title: str
    text: str


def get_encoder():
    return tiktoken.get_encoding(ENCODING_NAME)


def count_tokens(text: str) -> int:
    return len(get_encoder().encode(text))


def split_markdown_sections(markdown_text: str) -> list[Section]:
    """
    Divide Markdown por encabezados de nivel 1 a 3.
    El contenido anterior al primer encabezado se conserva como 'Sin sección'.
    """
    heading_pattern = re.compile(r"^(#{1,3})\s+(.+?)\s*$", re.MULTILINE)
    matches = list(heading_pattern.finditer(markdown_text))

    if not matches:
        normalized = markdown_text.strip()
        return [Section(title="Sin sección", text=normalized)] if normalized else []

    sections: list[Section] = []
    preamble = markdown_text[: matches[0].start()].strip()
    if preamble:
        sections.append(Section(title="Sin sección", text=preamble))

    current_h1 = ""
    for index, match in enumerate(matches):
        level = len(match.group(1))
        heading = match.group(2).strip()
        content_start = match.end()
        content_end = matches[index + 1].start() if index + 1 < len(matches) else len(markdown_text)
        content = markdown_text[content_start:content_end].strip()

        if level == 1:
            current_h1 = heading
            section_title = heading
        else:
            section_title = f"{current_h1} > {heading}" if current_h1 else heading

        if content:
            sections.append(Section(title=section_title, text=content))

    return sections


def split_into_semantic_units(text: str) -> list[str]:
    """
    Conserva párrafos como primera unidad semántica.
    Si un párrafo es demasiado largo, lo divide por oraciones.
    """
    paragraphs = [part.strip() for part in re.split(r"\n\s*\n", text) if part.strip()]
    units: list[str] = []

    for paragraph in paragraphs:
        if count_tokens(paragraph) <= MAX_TOKENS:
            units.append(paragraph)
            continue

        sentences = re.split(r"(?<=[.!?])\s+", paragraph)
        buffer: list[str] = []

        for sentence in sentences:
            candidate = " ".join(buffer + [sentence]).strip()
            if buffer and count_tokens(candidate) > MAX_TOKENS:
                units.append(" ".join(buffer))
                buffer = [sentence]
            else:
                buffer.append(sentence)

        if buffer:
            units.append(" ".join(buffer))

    return units


def tail_by_tokens(text: str, token_count: int) -> str:
    encoder = get_encoder()
    tokens = encoder.encode(text)
    return encoder.decode(tokens[-token_count:]) if tokens else ""


def chunk_section(section: Section) -> list[str]:
    """
    Agrupa unidades semánticas sin exceder MAX_TOKENS.
    El siguiente chunk inicia con hasta OVERLAP_TOKENS del chunk anterior.
    """
    units = split_into_semantic_units(section.text)
    if not units:
        return []

    chunks: list[str] = []
    current = ""

    for unit in units:
        candidate = f"{current}\n\n{unit}".strip() if current else unit

        if current and count_tokens(candidate) > MAX_TOKENS:
            chunks.append(current)
            overlap = tail_by_tokens(current, OVERLAP_TOKENS)
            current = f"{overlap}\n\n{unit}".strip()
        else:
            current = candidate

    if current:
        chunks.append(current)

    return chunks


def build_chunks(source_path: Path, root_directory: Path) -> list[dict]:
    markdown_text = source_path.read_text(encoding="utf-8")
    document_id = source_path.stem
    relative_path = str(source_path.relative_to(root_directory.parent))

    chunks: list[dict] = []
    chunk_number = 1

    for section in split_markdown_sections(markdown_text):
        for text in chunk_section(section):
            chunks.append(
                {
                    "document_id": document_id,
                    "source_path": relative_path,
                    "section_title": section.title,
                    "chunk_id": f"{document_id}-{chunk_number:03d}",
                    "text": text,
                    "token_count": count_tokens(text),
                }
            )
            chunk_number += 1

    return chunks
PY
```

2. Crea una prueba unitaria para validar la preservación de metadatos y límites de tokens:

```bash
cat > tests/test_semantic_chunker.py <<'PY'
from pathlib import Path

from src.semantic_chunker import MAX_TOKENS, build_chunks


def test_build_chunks_preserves_required_metadata(tmp_path: Path):
    document = tmp_path / "sample.md"
    document.write_text(
        """# Documento de prueba

## Sección principal

Este contenido describe autenticación mediante token Bearer.
Debe conservar el encabezado de la sección en los metadatos.
""",
        encoding="utf-8",
    )

    chunks = build_chunks(document, tmp_path)

    assert len(chunks) == 1
    chunk = chunks[0]
    assert chunk["document_id"] == "sample"
    assert chunk["source_path"] == "sample.md"
    assert chunk["section_title"] == "Documento de prueba > Sección principal"
    assert chunk["chunk_id"] == "sample-001"
    assert chunk["token_count"] <= MAX_TOKENS
    assert "token Bearer" in chunk["text"]
PY
```

3. Ejecuta la prueba:

```bash
pytest -q tests/test_semantic_chunker.py
```

**Resultado esperado**

La prueba debe finalizar correctamente:

```text
1 passed
```

**Verificación**

Confirma que el código aplica estos principios:

- Primero separa por encabezados Markdown.
- Conserva el título de sección como metadato.
- Divide por párrafos y, si es necesario, por oraciones.
- No utiliza únicamente ventanas de longitud fija.
- Añade solapamiento de hasta 60 tokens cuando se crea un nuevo fragmento.

---

### Paso 3. Implementar la ingesta documental

**Objetivo**

Generar un archivo JSON Lines con los fragmentos semánticos y metadatos requeridos.

**Instrucciones**

1. Crea el script `src/ingest_documents.py`:

```bash
cat > src/ingest_documents.py <<'PY'
from __future__ import annotations

import argparse
import json
from pathlib import Path

from semantic_chunker import build_chunks


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Ingiere documentos Markdown y genera fragmentos semánticos."
    )
    parser.add_argument(
        "--input-dir",
        default="data/documents",
        help="Directorio con documentos Markdown.",
    )
    parser.add_argument(
        "--output-file",
        default="data/processed/chunks.jsonl",
        help="Archivo JSONL de salida.",
    )
    args = parser.parse_args()

    input_dir = Path(args.input_dir)
    output_file = Path(args.output_file)

    if not input_dir.exists():
        raise FileNotFoundError(f"No existe el directorio de entrada: {input_dir}")

    documents = sorted(input_dir.glob("*.md"))
    if not documents:
        raise ValueError(f"No se encontraron archivos Markdown en {input_dir}")

    all_chunks: list[dict] = []
    for document in documents:
        chunks = build_chunks(document, input_dir)
        all_chunks.extend(chunks)
        print(f"Ingerido: {document.name} | chunks: {len(chunks)}")

    output_file.parent.mkdir(parents=True, exist_ok=True)
    with output_file.open("w", encoding="utf-8") as handle:
        for chunk in all_chunks:
            handle.write(json.dumps(chunk, ensure_ascii=False) + "\n")

    print(f"Total de chunks: {len(all_chunks)}")
    print(f"Archivo generado: {output_file}")


if __name__ == "__main__":
    main()
PY
```

2. Ejecuta la ingesta desde la raíz del repositorio:

```bash
python src/ingest_documents.py
```

3. Inspecciona las primeras líneas del archivo generado:

```bash
head -n 3 data/processed/chunks.jsonl
```

4. Valida que todos los fragmentos tengan los campos obligatorios:

```bash
python - <<'PY'
import json
from pathlib import Path

required = {
    "document_id",
    "source_path",
    "section_title",
    "chunk_id",
    "text",
    "token_count",
}

path = Path("data/processed/chunks.jsonl")
chunks = [json.loads(line) for line in path.read_text(encoding="utf-8").splitlines()]

assert chunks, "No se generaron fragmentos."
assert all(required.issubset(chunk) for chunk in chunks), "Faltan campos requeridos."
assert all(chunk["token_count"] <= 450 for chunk in chunks), "Hay chunks mayores de 450 tokens."

print(f"Validación correcta: {len(chunks)} chunks con metadatos completos.")
PY
```

**Resultado esperado**

La ingesta debe informar los documentos procesados y crear:

```text
data/processed/chunks.jsonl
```

Cada línea debe ser un objeto JSON independiente similar al siguiente:

```json
{
  "document_id": "authentication-policy",
  "source_path": "documents/authentication-policy.md",
  "section_title": "Política de autenticación y autorización > Identidad administrada",
  "chunk_id": "authentication-policy-002",
  "text": "Una Azure Container App puede usar identidad administrada...",
  "token_count": 57
}
```

**Verificación**

Ejecuta:

```bash
wc -l data/processed/chunks.jsonl
```

El archivo debe contener al menos un fragmento por sección documental. Confirma además que no contiene claves de API:

```bash
grep -iE "sk-[a-zA-Z0-9]" data/processed/chunks.jsonl && echo "Revisar posible secreto" || echo "Sin patrones de claves detectados"
```

---

### Paso 4. Generar embeddings con `text-embedding-3-small`

**Objetivo**

Generar un embedding para cada fragmento documental y almacenarlo junto con sus metadatos en `data/processed/chunks_with_embeddings.jsonl`.

**Instrucciones**

1. Crea el script `src/generate_embeddings.py`:

```bash
cat > src/generate_embeddings.py <<'PY'
from __future__ import annotations

import argparse
import json
import os
from pathlib import Path

from dotenv import load_dotenv
from openai import OpenAI


EMBEDDING_MODEL = "text-embedding-3-small"
BATCH_SIZE = 64


def read_jsonl(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8") as handle:
        return [json.loads(line) for line in handle if line.strip()]


def write_jsonl(path: Path, records: list[dict]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open("w", encoding="utf-8") as handle:
        for record in records:
            handle.write(json.dumps(record, ensure_ascii=False) + "\n")


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Genera embeddings para fragmentos JSONL."
    )
    parser.add_argument(
        "--input-file",
        default="data/processed/chunks.jsonl",
        help="Archivo JSONL de fragmentos.",
    )
    parser.add_argument(
        "--output-file",
        default="data/processed/chunks_with_embeddings.jsonl",
        help="Archivo JSONL con embeddings.",
    )
    args = parser.parse_args()

    load_dotenv()
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        raise EnvironmentError(
            "OPENAI_API_KEY no está configurada. Defínela en .env o en el entorno."
        )

    input_file = Path(args.input_file)
    output_file = Path(args.output_file)
    chunks = read_jsonl(input_file)

    if not chunks:
        raise ValueError("El archivo de entrada no contiene fragmentos.")

    client = OpenAI(api_key=api_key)
    embedded_chunks: list[dict] = []

    for start in range(0, len(chunks), BATCH_SIZE):
        batch = chunks[start : start + BATCH_SIZE]
        texts = [chunk["text"] for chunk in batch]

        response = client.embeddings.create(
            model=EMBEDDING_MODEL,
            input=texts,
        )

        for chunk, embedding_item in zip(batch, response.data, strict=True):
            record = {
                **chunk,
                "embedding_model": EMBEDDING_MODEL,
                "embedding": embedding_item.embedding,
            }
            embedded_chunks.append(record)

        end = min(start + BATCH_SIZE, len(chunks))
        print(f"Embeddings generados: {end}/{len(chunks)}")

    write_jsonl(output_file, embedded_chunks)
    print(f"Archivo generado: {output_file}")
    print(f"Registros escritos: {len(embedded_chunks)}")


if __name__ == "__main__":
    main()
PY
```

2. Verifica que la variable de entorno esté disponible sin imprimir su valor:

```bash
python - <<'PY'
import os
from dotenv import load_dotenv

load_dotenv()
assert os.getenv("OPENAI_API_KEY"), "OPENAI_API_KEY no está configurada."
print("OPENAI_API_KEY está configurada.")
PY
```

3. Genera los embeddings:

```bash
python src/generate_embeddings.py
```

4. Inspecciona de forma segura la estructura del primer registro, sin imprimir el vector completo:

```bash
python - <<'PY'
import json

with open("data/processed/chunks_with_embeddings.jsonl", encoding="utf-8") as file:
    first = json.loads(next(file))

print("chunk_id:", first["chunk_id"])
print("section_title:", first["section_title"])
print("embedding_model:", first["embedding_model"])
print("dimensión del embedding:", len(first["embedding"]))
print("primeros 5 valores:", first["embedding"][:5])
PY
```

**Resultado esperado**

El script debe indicar el progreso y crear el archivo:

```text
data/processed/chunks_with_embeddings.jsonl
```

El vector generado por `text-embedding-3-small` normalmente tendrá 1536 dimensiones.

**Verificación**

Ejecuta la siguiente validación:

```bash
python - <<'PY'
import json
from pathlib import Path

path = Path("data/processed/chunks_with_embeddings.jsonl")
records = [json.loads(line) for line in path.read_text(encoding="utf-8").splitlines()]

assert records, "No se encontraron registros con embeddings."
assert all(record["embedding_model"] == "text-embedding-3-small" for record in records)
assert all(isinstance(record["embedding"], list) and len(record["embedding"]) > 0 for record in records)
assert all(len(record["embedding"]) == 1536 for record in records)

print(f"Validación correcta: {len(records)} embeddings de 1536 dimensiones.")
PY
```

> **Nota operativa:** el archivo JSONL de embeddings puede crecer rápidamente en escenarios reales. Para este laboratorio se conserva como artefacto local. En un entorno productivo, se usaría normalmente un índice vectorial gestionado o una base de datos con soporte vectorial.

---

### Paso 5. Implementar la recuperación vectorial base

**Objetivo**

Crear un recuperador que genere el embedding de una consulta, calcule similitud coseno localmente y devuelva los fragmentos más relevantes con sus fuentes.

**Instrucciones**

1. Crea el script `src/retrieve_baseline.py`:

```bash
cat > src/retrieve_baseline.py <<'PY'
from __future__ import annotations

import argparse
import json
import math
import os
from pathlib import Path

from dotenv import load_dotenv
from openai import OpenAI


EMBEDDING_MODEL = "text-embedding-3-small"


def cosine_similarity(vector_a: list[float], vector_b: list[float]) -> float:
    dot_product = sum(a * b for a, b in zip(vector_a, vector_b, strict=True))
    norm_a = math.sqrt(sum(a * a for a in vector_a))
    norm_b = math.sqrt(sum(b * b for b in vector_b))

    if norm_a == 0 or norm_b == 0:
        return 0.0

    return dot_product / (norm_a * norm_b)


def load_records(path: Path) -> list[dict]:
    with path.open("r", encoding="utf-8") as handle:
        return [json.loads(line) for line in handle if line.strip()]


def retrieve(
    query: str,
    records: list[dict],
    client: OpenAI,
    top_k: int = 3,
) -> list[dict]:
    response = client.embeddings.create(
        model=EMBEDDING_MODEL,
        input=query,
    )
    query_embedding = response.data[0].embedding

    scored_results = []
    for record in records:
        score = cosine_similarity(query_embedding, record["embedding"])
        scored_results.append(
            {
                "score": round(score, 4),
                "chunk_id": record["chunk_id"],
                "document_id": record["document_id"],
                "source_path": record["source_path"],
                "section_title": record["section_title"],
                "text": record["text"],
            }
        )

    return sorted(scored_results, key=lambda item: item["score"], reverse=True)[:top_k]


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Recupera fragmentos relevantes mediante similitud coseno."
    )
    parser.add_argument("--query", required=True, help="Consulta técnica.")
    parser.add_argument(
        "--input-file",
        default="data/processed/chunks_with_embeddings.jsonl",
        help="Archivo JSONL con embeddings.",
    )
    parser.add_argument("--top-k", type=int, default=3, help="Número de resultados.")
    args = parser.parse_args()

    load_dotenv()
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        raise EnvironmentError("OPENAI_API_KEY no está configurada.")

    records = load_records(Path(args.input_file))
    client = OpenAI(api_key=api_key)

    results = retrieve(
        query=args.query,
        records=records,
        client=client,
        top_k=args.top_k,
    )

    print(json.dumps(
        {
            "query": args.query,
            "results": results,
        },
        ensure_ascii=False,
        indent=2,
    ))


if __name__ == "__main__":
    main()
PY
```

2. Ejecuta una consulta sobre autenticación:

```bash
python src/retrieve_baseline.py \
  --query "¿Cómo debe enviarse el token para autenticar una solicitud a la API?" \
  --top-k 3
```

3. Ejecuta una consulta sobre identidad administrada y RBAC:

```bash
python src/retrieve_baseline.py \
  --query "¿Cómo se deben asignar permisos a una identidad administrada en Container Apps?" \
  --top-k 3
```

4. Ejecuta una consulta sobre el error 429:

```bash
python src/retrieve_baseline.py \
  --query "¿Qué acciones debe realizar el cliente si el proveedor devuelve HTTP 429?" \
  --top-k 3
```

**Resultado esperado**

Cada ejecución debe devolver un objeto JSON con:

- La consulta original.
- Los resultados ordenados de mayor a menor similitud.
- El identificador del fragmento.
- La sección documental.
- La ruta del documento fuente.
- El texto recuperado.
- La puntuación de similitud.

Para la consulta sobre HTTP 429, uno de los primeros resultados debe provenir de:

```text
Runbook de incidencias del agente de soporte > Error 429 del proveedor de modelos
```

**Verificación**

Guarda una consulta de validación en el directorio de reportes:

```bash
python src/retrieve_baseline.py \
  --query "¿Dónde debe configurarse OPENAI_API_KEY durante el despliegue?" \
  --top-k 3 > reports/retrieval_deployment_secret.json
```

Valida que el resultado referencia el runbook de despliegue:

```bash
grep -q "Variables de entorno" reports/retrieval_deployment_secret.json \
  && echo "Recuperación validada" \
  || echo "Revisar ranking o contenido documental"
```

---

### Paso 6. Integrar el `KnowledgeRetrievalSkill`

**Objetivo**

Exponer la recuperación como una habilidad reutilizable por el agente y por el harness automatizado del laboratorio anterior.

**Instrucciones**

1. Crea el archivo de inicialización del paquete si no existe:

```bash
touch src/__init__.py
touch src/skills/__init__.py
```

2. Crea `src/skills/knowledge_retrieval.py`:

```bash
cat > src/skills/knowledge_retrieval.py <<'PY'
from __future__ import annotations

import json
import math
import os
from pathlib import Path

from dotenv import load_dotenv
from openai import OpenAI
from pydantic import BaseModel, Field


EMBEDDING_MODEL = "text-embedding-3-small"
DEFAULT_INDEX_PATH = "data/processed/chunks_with_embeddings.jsonl"


class RetrievedSource(BaseModel):
    chunk_id: str
    document_id: str
    source_path: str
    section_title: str
    score: float
    text: str


class KnowledgeRetrievalResult(BaseModel):
    query: str
    sources: list[RetrievedSource] = Field(default_factory=list)
    context: str


class KnowledgeRetrievalSkill:
    """
    Recupera evidencia documental vectorial para que otra capa genere respuestas.
    Esta habilidad no genera respuestas ni inventa fuentes.
    """

    def __init__(self, index_path: str = DEFAULT_INDEX_PATH) -> None:
        load_dotenv()
        api_key = os.getenv("OPENAI_API_KEY")
        if not api_key:
            raise EnvironmentError("OPENAI_API_KEY no está configurada.")

        self.client = OpenAI(api_key=api_key)
        self.index_path = Path(index_path)
        self.records = self._load_records()

    def _load_records(self) -> list[dict]:
        if not self.index_path.exists():
            raise FileNotFoundError(
                f"No existe el índice de embeddings: {self.index_path}. "
                "Ejecute ingest_documents.py y generate_embeddings.py."
            )

        with self.index_path.open("r", encoding="utf-8") as handle:
            records = [json.loads(line) for line in handle if line.strip()]

        if not records:
            raise ValueError("El índice de embeddings está vacío.")

        return records

    @staticmethod
    def _cosine_similarity(vector_a: list[float], vector_b: list[float]) -> float:
        dot_product = sum(a * b for a, b in zip(vector_a, vector_b, strict=True))
        norm_a = math.sqrt(sum(a * a for a in vector_a))
        norm_b = math.sqrt(sum(b * b for b in vector_b))
        return dot_product / (norm_a * norm_b) if norm_a and norm_b else 0.0

    def retrieve(self, query: str, top_k: int = 3) -> KnowledgeRetrievalResult:
        if not query.strip():
            raise ValueError("La consulta no puede estar vacía.")

        embedding_response = self.client.embeddings.create(
            model=EMBEDDING_MODEL,
            input=query,
        )
        query_embedding = embedding_response.data[0].embedding

        scored_sources = [
            RetrievedSource(
                chunk_id=record["chunk_id"],
                document_id=record["document_id"],
                source_path=record["source_path"],
                section_title=record["section_title"],
                score=round(
                    self._cosine_similarity(query_embedding, record["embedding"]),
                    4,
                ),
                text=record["text"],
            )
            for record in self.records
        ]

        sources = sorted(
            scored_sources,
            key=lambda source: source.score,
            reverse=True,
        )[:top_k]

        context_lines = [
            (
                f"[Fuente: {source.source_path} | "
                f"Sección: {source.section_title} | "
                f"Chunk: {source.chunk_id}]\n"
                f"{source.text}"
            )
            for source in sources
        ]

        return KnowledgeRetrievalResult(
            query=query,
            sources=sources,
            context="\n\n---\n\n".join(context_lines),
        )
PY
```

3. Crea una prueba del contrato público de la habilidad:

```bash
cat > tests/test_knowledge_retrieval_contract.py <<'PY'
from src.skills.knowledge_retrieval import KnowledgeRetrievalResult, RetrievedSource


def test_retrieval_result_exposes_traceable_sources():
    source = RetrievedSource(
        chunk_id="incident-runbook-001",
        document_id="incident-runbook",
        source_path="documents/incident-runbook.md",
        section_title="Runbook de incidencias > Error 429",
        score=0.85,
        text="Implemente reintentos con espera exponencial.",
    )

    result = KnowledgeRetrievalResult(
        query="¿Qué hacer ante HTTP 429?",
        sources=[source],
        context="[Fuente: documents/incident-runbook.md]\nImplemente reintentos.",
    )

    assert result.sources[0].chunk_id == "incident-runbook-001"
    assert "Fuente:" in result.context
    assert result.sources[0].score > 0
PY
```

4. Ejecuta todas las pruebas locales:

```bash
pytest -q
```

5. Prueba la habilidad mediante Python:

```bash
python - <<'PY'
from src.skills.knowledge_retrieval import KnowledgeRetrievalSkill

skill = KnowledgeRetrievalSkill()
result = skill.retrieve(
    "¿Qué se debe hacer si una clave de API pudo quedar expuesta?",
    top_k=2,
)

print("Consulta:", result.query)
print("\nContexto recuperado:\n")
print(result.context)
PY
```

**Resultado esperado**

La salida debe incluir fragmentos relacionados con gestión de secretos, revocación y reemplazo de claves. Cada fuente debe incluir su ruta, sección y `chunk_id`.

**Verificación**

El objeto devuelto por `KnowledgeRetrievalSkill.retrieve()` debe cumplir estas condiciones:

- Tiene la consulta original en `query`.
- Tiene una lista `sources`.
- Cada fuente contiene `chunk_id`, `document_id`, `source_path`, `section_title`, `score` y `text`.
- `context` contiene referencias atribuibles a fuentes.
- La habilidad recupera evidencia, pero no genera una respuesta final del modelo.

> **Relación con GraphRAG:** esta habilidad implementa la búsqueda vectorial de una recuperación híbrida. Un paso posterior puede extraer entidades como `Azure Container Apps`, `RBAC`, `OPENAI_API_KEY` e `identidad administrada`, y conectar sus relaciones para responder preguntas compuestas con evidencia proveniente de múltiples documentos.

---

## Validación y pruebas

Ejecuta la secuencia completa desde la raíz del repositorio:

```bash
source .venv/bin/activate

python src/ingest_documents.py

python src/generate_embeddings.py

pytest -q

python src/retrieve_baseline.py \
  --query "¿Qué debe hacer un cliente cuando recibe HTTP 401?" \
  --top-k 2
```

Crea un reporte de validación reproducible:

```bash
python - <<'PY' > reports/lab-03-00-01-validation.txt
import json
from pathlib import Path

chunks_path = Path("data/processed/chunks.jsonl")
embeddings_path = Path("data/processed/chunks_with_embeddings.jsonl")

chunks = [
    json.loads(line)
    for line in chunks_path.read_text(encoding="utf-8").splitlines()
    if line.strip()
]
records = [
    json.loads(line)
    for line in embeddings_path.read_text(encoding="utf-8").splitlines()
    if line.strip()
]

required_chunk_fields = {
    "document_id",
    "source_path",
    "section_title",
    "chunk_id",
    "text",
    "token_count",
}

assert len(chunks) > 0
assert len(chunks) == len(records)
assert all(required_chunk_fields.issubset(chunk.keys()) for chunk in chunks)
assert all(chunk["token_count"] <= 450 for chunk in chunks)
assert all(record["embedding_model"] == "text-embedding-3-small" for record in records)
assert all(len(record["embedding"]) == 1536 for record in records)

print("VALIDACIÓN CORRECTA")
print(f"Chunks generados: {len(chunks)}")
print(f"Embeddings generados: {len(records)}")
print("Límite de tokens: todos los chunks tienen <= 450 tokens")
print("Modelo de embeddings: text-embedding-3-small")
print("Dimensión de embeddings: 1536")
PY

cat reports/lab-03-00-01-validation.txt
```

El reporte debe indicar `VALIDACIÓN CORRECTA`.

Finalmente, revisa los cambios pendientes y realiza el commit de la práctica:

```bash
git status --short
git add data/documents src tests reports .gitignore
git commit -m "lab-03-00-01"
```

No incluyas en el commit:

- `.env`
- Directorios de caché como `__pycache__/`
- Claves, tokens o credenciales
- Archivos temporales no requeridos

## Solución de problemas

### Problema 1: `OPENAI_API_KEY no está configurada`

**Síntoma**

Al ejecutar `generate_embeddings.py`, `retrieve_baseline.py` o `KnowledgeRetrievalSkill`, aparece un error similar a:

```text
EnvironmentError: OPENAI_API_KEY no está configurada.
```

**Causa**

El archivo `.env` no existe, la variable está escrita con otro nombre, el script se ejecuta fuera de `~/genai-agent-labs` o la clave no está exportada en el entorno.

**Solución**

1. Confirma que estás en el directorio del repositorio:

```bash
cd ~/genai-agent-labs
```

2. Verifica que el archivo exista:

```bash
ls -la .env
```

3. Confirma que contiene exactamente la variable esperada, sin imprimir su valor:

```bash
grep '^OPENAI_API_KEY=' .env | cut -d= -f1
```

4. Vuelve a ejecutar el script desde la raíz del repositorio:

```bash
python src/generate_embeddings.py
```

### Problema 2: La recuperación devuelve resultados poco relacionados con la consulta

**Síntoma**

La consulta sobre un error HTTP, RBAC o secretos no devuelve entre los primeros resultados la sección técnica esperada.

**Causa**

El archivo de embeddings está desactualizado respecto de los documentos, la consulta es demasiado ambigua, el conjunto documental es muy pequeño o los fragmentos no se regeneraron después de modificar el contenido.

**Solución**

1. Regenera los fragmentos y embeddings:

```bash
python src/ingest_documents.py
python src/generate_embeddings.py
```

2. Formula una consulta que incluya términos técnicos distintivos, por ejemplo:

```bash
python src/retrieve_baseline.py \
  --query "reintentos con espera exponencial después de HTTP 429" \
  --top-k 3
```

3. Inspecciona las secciones disponibles:

```bash
python - <<'PY'
import json

with open("data/processed/chunks.jsonl", encoding="utf-8") as file:
    for line in file:
        chunk = json.loads(line)
        print(chunk["chunk_id"], "-", chunk["section_title"])
PY
```

4. En una implementación posterior, mejora el ranking con filtros de metadatos, búsqueda por entidades, reordenamiento o expansión de relaciones GraphRAG.

## Limpieza

Este laboratorio conserva los documentos y el índice local porque son insumos para prácticas posteriores. No elimines `data/documents/` ni `data/processed/chunks_with_embeddings.jsonl` si el instructor indica reutilizarlos.

Para eliminar únicamente cachés de Python y resultados temporales no versionados:

```bash
find src tests -type d -name "__pycache__" -prune -exec rm -rf {} +
rm -f reports/retrieval_deployment_secret.json
```

Confirma que no se versionará el archivo de secretos:

```bash
git check-ignore .env
```

Si necesitas invalidar una clave utilizada durante la práctica por una exposición accidental, revócala desde el panel del proveedor y actualiza `.env` con una nueva clave. Nunca intentes “limpiar” una clave ya confirmada en Git solamente borrando el archivo: debe revocarse.

## Resumen

En esta práctica construiste un pipeline RAG vectorial completo para documentación técnica:

1. Creaste una base documental Markdown con guías, políticas y runbooks.
2. Implementaste chunking semántico basado en secciones, párrafos y oraciones.
3. Conservaste `document_id`, `source_path`, `section_title`, `chunk_id`, texto y conteo de tokens en cada fragmento.
4. Generaste embeddings con `text-embedding-3-small`.
5. Implementaste recuperación por similitud coseno.
6. Expusiste la evidencia mediante `KnowledgeRetrievalSkill` para su consumo por un agente y un harness de evaluación.

La recuperación vectorial responde eficazmente a preguntas próximas al contenido documental, como “¿qué hacer ante HTTP 429?”. Para preguntas compuestas que requieren conectar políticas, sistemas, propietarios y responsabilidades, el siguiente paso es complementar esta base con entidades y relaciones explícitas, aplicando recuperación híbrida y expansión controlada de un grafo.

### Recursos opcionales

- [OpenAI API: Embeddings](https://platform.openai.com/docs/guides/embeddings)
- [Documentación de tiktoken](https://github.com/openai/tiktoken)
- [Microsoft Research: GraphRAG](https://www.microsoft.com/en-us/research/project/graphrag/)
- [Azure AI Search: recuperación aumentada por generación](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview)

---

# 1. Práctica 2. Extender el pipeline mediante GraphRAG para responder preguntas considerando relaciones entre entidades almacenadas en un grafo de conocimiento.

## Metadatos

| Propiedad | Valor |
|---|---|
| Duración | 40 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En este laboratorio extenderás el pipeline RAG construido en el laboratorio `03-00-01` con una capa de conocimiento estructurado. Extraerás entidades técnicas y relaciones explícitas desde los fragmentos documentales ya vectorizados, construirás un grafo portable en JSON y desarrollarás un recuperador híbrido capaz de combinar evidencia textual con recorridos BFS de hasta dos saltos.

El resultado será una evolución de `KnowledgeRetrievalSkill` hacia `GraphKnowledgeRetrievalSkill`, preparada para responder sin inventar relaciones y para entregar las fuentes documentales que justifican cada hecho recuperado.

## Objetivos de aprendizaje

- [ ] Extraer entidades normalizadas de tipo `Service`, `API`, `Component`, `ErrorCode`, `Procedure`, `Configuration` y `Dependency`.
- [ ] Construir relaciones `DEPENDS_ON`, `EXPOSES`, `AUTHENTICATES_WITH`, `RESOLVES`, `DEPLOYED_ON` y `DOCUMENTED_IN`.
- [ ] Persistir un grafo técnico portable en `data/graph/technical_graph.json`.
- [ ] Implementar recuperación híbrida mediante coincidencia de entidades, evidencia textual y expansión BFS de hasta dos saltos.
- [ ] Validar que las respuestas y los hechos recuperados estén fundamentados en fragmentos fuente.

## Prerrequisitos

### Conocimientos

Antes de comenzar, debes comprender:

- El pipeline de fragmentación, embeddings y recuperación semántica implementado en el laboratorio `03-00-01`.
- Estructuras de datos JSON, nodos, aristas, propiedades y recorridos BFS.
- Uso básico de Python, entornos virtuales, `pytest` y Git.
- El harness de evaluación implementado o disponible desde el laboratorio `02-00-04`.

### Acceso y artefactos requeridos

Debes disponer de:

- El repositorio local `~/genai-agent-labs`.
- El entorno virtual compartido en `~/genai-agent-labs/.venv`.
- El archivo de entrada:

  ```text
  ~/genai-agent-labs/data/processed/chunks_with_embeddings.jsonl
  ```

- Una versión funcional de la recuperación baseline del laboratorio `03-00-01`.
- Acceso a Azure OpenAI únicamente si tu implementación baseline genera embeddings de consulta en tiempo de ejecución.
- Acceso al harness existente de evaluación de grounding.

> Este laboratorio no requiere Docker, una base de datos ni un servicio externo de grafos. El grafo se conserva inicialmente como JSON para inspeccionar y validar la lógica de extracción.

## Entorno del laboratorio

### Requisitos de hardware

| Recurso | Mínimo recomendado |
|---|---:|
| Procesador | 4 núcleos |
| Memoria RAM | 16 GB |
| Espacio libre en disco | 10 GB |
| Sistema operativo de referencia | Ubuntu 22.04.4 LTS de 64 bits |

### Software

| Herramienta | Versión de referencia |
|---|---|
| Python | 3.12.x |
| pip | 23.3.2 o superior |
| Git | 2.43.0 |
| Visual Studio Code | 1.86.2 o superior |
| OpenAI Python SDK | 1.35.13 |
| pytest | Instalado en el entorno compartido |

### Preparación inicial

1. Abre una terminal y entra al repositorio obligatorio:

   ```bash
   cd ~/genai-agent-labs
   ```

2. Activa el entorno virtual compartido:

   ```bash
   source .venv/bin/activate
   ```

3. Comprueba la versión de Python:

   ```bash
   python --version
   ```

4. Verifica que existe el artefacto de entrada del laboratorio anterior:

   ```bash
   ls -lh data/processed/chunks_with_embeddings.jsonl
   ```

5. Crea los directorios de trabajo del laboratorio:

   ```bash
   mkdir -p src/graphrag data/graph tests reports
   touch src/graphrag/__init__.py
   ```

**Salida esperada**

Debes observar una versión de Python 3.12.x y el archivo `chunks_with_embeddings.jsonl` debe existir y tener contenido.

**Verificación**

```bash
test -s data/processed/chunks_with_embeddings.jsonl && echo "Entrada disponible"
```

La salida debe ser:

```text
Entrada disponible
```

---

## Procedimiento paso a paso

### Paso 1. Inspeccionar el contrato de los fragmentos de entrada

**Objetivo:** identificar los campos disponibles en el archivo generado por el pipeline baseline y definir el contrato mínimo utilizado por GraphRAG.

**Instrucciones**

1. Muestra las primeras líneas del archivo JSONL:

   ```bash
   head -n 2 data/processed/chunks_with_embeddings.jsonl | python -m json.tool
   ```

2. Si el comando anterior falla porque hay varias líneas JSON independientes, inspecciona la primera línea:

   ```bash
   head -n 1 data/processed/chunks_with_embeddings.jsonl | python -m json.tool
   ```

3. Identifica los campos equivalentes a los siguientes:

   | Campo lógico requerido | Nombres aceptables frecuentes |
   |---|---|
   | Identificador del fragmento | `chunk_id`, `id` |
   | Texto del fragmento | `text`, `content`, `chunk_text` |
   | Vector | `embedding`, `embeddings` |
   | Documento fuente | `source`, `source_file`, `document_id` |
   | Metadatos opcionales | `metadata` |

4. Crea el archivo `src/graphrag/contracts.py`:

   ```python
   from __future__ import annotations

   from typing import Any


   def get_chunk_id(chunk: dict[str, Any]) -> str:
       return str(chunk.get("chunk_id") or chunk.get("id") or "")


   def get_chunk_text(chunk: dict[str, Any]) -> str:
       return str(
           chunk.get("text")
           or chunk.get("content")
           or chunk.get("chunk_text")
           or ""
       )


   def get_chunk_source(chunk: dict[str, Any]) -> str:
       metadata = chunk.get("metadata") or {}
       return str(
           chunk.get("source")
           or chunk.get("source_file")
           or chunk.get("document_id")
           or metadata.get("source")
           or metadata.get("source_file")
           or "fuente_desconocida"
       )
   ```

5. Si tu pipeline usa nombres de campos diferentes, adapta únicamente las funciones anteriores. No modifiques el archivo de datos original.

**Salida esperada**

Debes conocer el identificador, el texto y la fuente de cada fragmento. El contrato de compatibilidad debe aislar las diferencias de formato entre laboratorios.

**Verificación**

Ejecuta este comando:

```bash
python - <<'PY'
import json
from src.graphrag.contracts import get_chunk_id, get_chunk_text, get_chunk_source

with open("data/processed/chunks_with_embeddings.jsonl", encoding="utf-8") as file:
    chunk = json.loads(next(file))

print("chunk_id:", get_chunk_id(chunk))
print("source:", get_chunk_source(chunk))
print("text_preview:", get_chunk_text(chunk)[:120])
PY
```

La salida debe mostrar un identificador no vacío, una fuente y una vista previa de texto.

---

### Paso 2. Implementar la extracción de entidades y relaciones

**Objetivo:** crear un extractor determinista y trazable que identifique entidades técnicas y relaciones explícitas en los chunks documentales.

**Instrucciones**

1. Crea el archivo `src/graphrag/extract_graph_entities.py`.

2. Copia la siguiente implementación:

   ```python
   from __future__ import annotations

   import argparse
   import json
   import re
   import unicodedata
   from collections import defaultdict
   from pathlib import Path
   from typing import Any

   from src.graphrag.contracts import (
       get_chunk_id,
       get_chunk_source,
       get_chunk_text,
   )


   ENTITY_PATTERNS: dict[str, list[str]] = {
       "Service": [
           r"\b(?:servicio|service)\s+[`“\"]?([A-Za-z][A-Za-z0-9._-]{1,80})[`”\"]?",
       ],
       "API": [
           r"\b(?:api|endpoint)\s+[`“\"]?([A-Za-z][A-Za-z0-9._/-]{1,80})[`”\"]?",
       ],
       "Component": [
           r"\b(?:componente|component|m[oó]dulo|module)\s+[`“\"]?([A-Za-z][A-Za-z0-9._-]{1,80})[`”\"]?",
       ],
       "ErrorCode": [
           r"\b((?:ERR|HTTP|E)[-_ ]?\d{3,5})\b",
       ],
       "Procedure": [
           r"\b(?:procedimiento|procedure|runbook)\s+[#:]*[`“\"]?([A-Za-z][A-Za-z0-9._ -]{1,80})[`”\"]?",
       ],
       "Configuration": [
           r"\b(?:configuraci[oó]n|configuration|variable de entorno|setting)\s+[`“\"]?([A-Za-z_][A-Za-z0-9_.-]{1,80})[`”\"]?",
       ],
       "Dependency": [
           r"\b(?:dependencia|dependency|paquete|package|librer[ií]a|library)\s+[`“\"]?([A-Za-z][A-Za-z0-9._-]{1,80})[`”\"]?",
       ],
   }

   RELATION_PATTERNS: list[tuple[str, str]] = [
       (
           "DEPENDS_ON",
           r"(?P<source>[A-Za-z][A-Za-z0-9._/-]{1,80})\s+"
           r"(?:depende de|depends on|requiere)\s+"
           r"(?P<target>[A-Za-z][A-Za-z0-9._/-]{1,80})",
       ),
       (
           "EXPOSES",
           r"(?P<source>[A-Za-z][A-Za-z0-9._/-]{1,80})\s+"
           r"(?:expone|exposes|publica)\s+"
           r"(?P<target>(?:API|api|endpoint)?\s*[A-Za-z][A-Za-z0-9._/-]{1,80})",
       ),
       (
           "AUTHENTICATES_WITH",
           r"(?P<source>[A-Za-z][A-Za-z0-9._/-]{1,80})\s+"
           r"(?:se autentica con|autentica con|authenticates with|usa para autenticaci[oó]n)\s+"
           r"(?P<target>[A-Za-z][A-Za-z0-9._/-]{1,80})",
       ),
       (
           "RESOLVES",
           r"(?P<source>(?:ERR|HTTP|E)[-_ ]?\d{3,5}|[A-Za-z][A-Za-z0-9._ -]{1,80})\s+"
           r"(?:se resuelve con|se corrige con|resolves|fixes)\s+"
           r"(?P<target>[A-Za-z][A-Za-z0-9._ -]{1,80})",
       ),
       (
           "DEPLOYED_ON",
           r"(?P<source>[A-Za-z][A-Za-z0-9._/-]{1,80})\s+"
           r"(?:se despliega en|desplegado en|deployed on|runs on)\s+"
           r"(?P<target>[A-Za-z][A-Za-z0-9._/-]{1,80})",
       ),
       (
           "DOCUMENTED_IN",
           r"(?P<source>[A-Za-z][A-Za-z0-9._/-]{1,80})\s+"
           r"(?:est[aá] documentado en|documented in)\s+"
           r"(?P<target>[A-Za-z][A-Za-z0-9._ -]{1,100})",
       ),
   ]


   def normalize(value: str) -> str:
       normalized = unicodedata.normalize("NFKD", value)
       normalized = "".join(
           char for char in normalized if not unicodedata.combining(char)
       )
       normalized = normalized.lower().strip()
       normalized = re.sub(r"[^a-z0-9]+", "-", normalized)
       return normalized.strip("-")


   def entity_id(entity_type: str, name: str) -> str:
       return f"{entity_type.lower()}:{normalize(name)}"


   def infer_entity_type(name: str, known_entities: dict[str, dict[str, Any]]) -> str:
       normalized_name = normalize(name)

       for entity in known_entities.values():
           if entity["normalized_name"] == normalized_name:
               return entity["type"]

       if re.fullmatch(r"(?:err|http|e)-?\d{3,5}", normalized_name):
           return "ErrorCode"
       if normalized_name.startswith("api-") or "/api/" in normalized_name:
           return "API"
       if "procedure" in normalized_name or "procedimiento" in normalized_name:
           return "Procedure"
       if normalized_name.startswith("config") or normalized_name.isupper():
           return "Configuration"

       return "Component"


   def add_entity(
       entities: dict[str, dict[str, Any]],
       entity_type: str,
       name: str,
       chunk_id: str,
       source: str,
   ) -> str:
       clean_name = re.sub(r"\s+", " ", name).strip(" `\"“”")
       node_id = entity_id(entity_type, clean_name)

       if node_id not in entities:
           entities[node_id] = {
               "id": node_id,
               "type": entity_type,
               "name": clean_name,
               "normalized_name": normalize(clean_name),
               "evidence": [],
           }

       evidence = {
           "chunk_id": chunk_id,
           "source": source,
       }

       if evidence not in entities[node_id]["evidence"]:
           entities[node_id]["evidence"].append(evidence)

       return node_id


   def extract_entities_from_chunk(
       text: str,
       chunk_id: str,
       source: str,
       entities: dict[str, dict[str, Any]],
   ) -> None:
       for entity_type, patterns in ENTITY_PATTERNS.items():
           for pattern in patterns:
               for match in re.finditer(pattern, text, flags=re.IGNORECASE):
                   add_entity(
                       entities,
                       entity_type,
                       match.group(1),
                       chunk_id,
                       source,
                   )


   def get_or_create_relation_entity(
       name: str,
       chunk_id: str,
       source: str,
       entities: dict[str, dict[str, Any]],
   ) -> str:
       entity_type = infer_entity_type(name, entities)
       return add_entity(entities, entity_type, name, chunk_id, source)


   def extract_relations_from_chunk(
       text: str,
       chunk_id: str,
       source: str,
       entities: dict[str, dict[str, Any]],
       edges: dict[tuple[str, str, str], dict[str, Any]],
   ) -> None:
       for relation_type, pattern in RELATION_PATTERNS:
           for match in re.finditer(pattern, text, flags=re.IGNORECASE):
               source_name = match.group("source").strip()
               target_name = match.group("target").strip()

               source_id = get_or_create_relation_entity(
                   source_name, chunk_id, source, entities
               )
               target_id = get_or_create_relation_entity(
                   target_name, chunk_id, source, entities
               )

               edge_key = (source_id, relation_type, target_id)

               if edge_key not in edges:
                   edges[edge_key] = {
                       "id": (
                           f"edge:{normalize(source_id)}:"
                           f"{relation_type.lower()}:"
                           f"{normalize(target_id)}"
                       ),
                       "source": source_id,
                       "relation": relation_type,
                       "target": target_id,
                       "evidence": [],
                   }

               evidence = {
                   "chunk_id": chunk_id,
                   "source": source,
                   "excerpt": text[:500],
               }

               if evidence not in edges[edge_key]["evidence"]:
                   edges[edge_key]["evidence"].append(evidence)


   def load_chunks(input_path: Path) -> list[dict[str, Any]]:
       chunks: list[dict[str, Any]] = []

       with input_path.open(encoding="utf-8") as file:
           for line_number, line in enumerate(file, start=1):
               if not line.strip():
                   continue

               try:
                   chunks.append(json.loads(line))
               except json.JSONDecodeError as error:
                   raise ValueError(
                       f"JSON inválido en la línea {line_number}: {error}"
                   ) from error

       return chunks


   def build_graph(chunks: list[dict[str, Any]]) -> dict[str, Any]:
       entities: dict[str, dict[str, Any]] = {}
       edges: dict[tuple[str, str, str], dict[str, Any]] = {}

       for chunk in chunks:
           chunk_id = get_chunk_id(chunk)
           text = get_chunk_text(chunk)
           source = get_chunk_source(chunk)

           if not chunk_id or not text:
               continue

           extract_entities_from_chunk(text, chunk_id, source, entities)
           extract_relations_from_chunk(
               text,
               chunk_id,
               source,
               entities,
               edges,
           )

       chunks_index = {
           get_chunk_id(chunk): {
               "chunk_id": get_chunk_id(chunk),
               "source": get_chunk_source(chunk),
               "text": get_chunk_text(chunk),
           }
           for chunk in chunks
           if get_chunk_id(chunk)
       }

       nodes_by_type: dict[str, int] = defaultdict(int)
       for node in entities.values():
           nodes_by_type[node["type"]] += 1

       return {
           "schema_version": "1.0",
           "description": "Grafo técnico extraído desde chunks documentales.",
           "statistics": {
               "node_count": len(entities),
               "edge_count": len(edges),
               "nodes_by_type": dict(sorted(nodes_by_type.items())),
               "chunk_count": len(chunks_index),
           },
           "nodes": sorted(entities.values(), key=lambda node: node["id"]),
           "edges": sorted(edges.values(), key=lambda edge: edge["id"]),
           "chunks": chunks_index,
       }


   def main() -> None:
       parser = argparse.ArgumentParser(
           description="Extrae entidades y relaciones para un grafo técnico."
       )
       parser.add_argument(
           "--input",
           default="data/processed/chunks_with_embeddings.jsonl",
       )
       parser.add_argument(
           "--output",
           default="data/graph/technical_graph.json",
       )
       args = parser.parse_args()

       input_path = Path(args.input)
       output_path = Path(args.output)

       if not input_path.exists():
           raise FileNotFoundError(
               f"No existe el archivo de entrada: {input_path}"
           )

       graph = build_graph(load_chunks(input_path))
       output_path.parent.mkdir(parents=True, exist_ok=True)

       with output_path.open("w", encoding="utf-8") as file:
           json.dump(graph, file, ensure_ascii=False, indent=2)

       print(
           "Grafo generado:",
           f"{graph['statistics']['node_count']} nodos,",
           f"{graph['statistics']['edge_count']} aristas.",
       )
       print(f"Archivo: {output_path}")


   if __name__ == "__main__":
       main()
   ```

3. Observa las decisiones de diseño de la implementación:

   - Los identificadores de entidad siguen el patrón `tipo:nombre-normalizado`.
   - Cada entidad y arista conserva referencias a `chunk_id` y `source`.
   - Las relaciones se extraen únicamente cuando hay patrones explícitos en el texto.
   - La relación no se crea si el patrón no está presente; este comportamiento reduce alucinaciones estructurales.
   - Los nodos y aristas se deduplican mediante claves normalizadas.
   - El grafo mantiene un índice ligero de chunks para recuperar evidencia textual sin una base de datos.

4. Ejecuta el extractor:

   ```bash
   python -m src.graphrag.extract_graph_entities \
     --input data/processed/chunks_with_embeddings.jsonl \
     --output data/graph/technical_graph.json
   ```

**Salida esperada**

La salida debe ser similar a:

```text
Grafo generado: 18 nodos, 9 aristas.
Archivo: data/graph/technical_graph.json
```

El número exacto dependerá de tus documentos técnicos.

**Verificación**

Inspecciona las estadísticas y algunas aristas:

```bash
python - <<'PY'
import json

with open("data/graph/technical_graph.json", encoding="utf-8") as file:
    graph = json.load(file)

print(json.dumps(graph["statistics"], ensure_ascii=False, indent=2))
print("\nPrimeras aristas:")
for edge in graph["edges"][:5]:
    print(
        f"- {edge['source']} --{edge['relation']}--> {edge['target']}"
    )
    print(f"  Evidencia: {edge['evidence'][0]['chunk_id']}")
PY
```

Confirma que toda arista posee al menos una entrada en `evidence`.

---

### Paso 3. Revisar y validar el artefacto de grafo

**Objetivo:** comprobar que el archivo JSON representa un grafo trazable, portable y consistente.

**Instrucciones**

1. Valida que el archivo sea JSON correcto:

   ```bash
   python -m json.tool data/graph/technical_graph.json > /dev/null
   ```

2. Muestra los tipos de entidades identificados:

   ```bash
   python - <<'PY'
   import json
   from collections import Counter

   with open("data/graph/technical_graph.json", encoding="utf-8") as file:
       graph = json.load(file)

   counts = Counter(node["type"] for node in graph["nodes"])

   for entity_type, count in sorted(counts.items()):
       print(f"{entity_type}: {count}")
   PY
   ```

3. Muestra las relaciones extraídas:

   ```bash
   python - <<'PY'
   import json
   from collections import Counter

   with open("data/graph/technical_graph.json", encoding="utf-8") as file:
       graph = json.load(file)

   counts = Counter(edge["relation"] for edge in graph["edges"])

   for relation, count in sorted(counts.items()):
       print(f"{relation}: {count}")
   PY
   ```

4. Revisa visualmente una arista completa:

   ```bash
   python - <<'PY'
   import json

   with open("data/graph/technical_graph.json", encoding="utf-8") as file:
       graph = json.load(file)

   if graph["edges"]:
       print(json.dumps(graph["edges"][0], ensure_ascii=False, indent=2))
   else:
       print("No se extrajeron aristas. Revise los patrones y el contenido documental.")
   PY
   ```

**Salida esperada**

Debes observar un documento JSON con la estructura principal:

```json
{
  "schema_version": "1.0",
  "statistics": {},
  "nodes": [],
  "edges": [],
  "chunks": {}
}
```

Cada arista debe contener:

```json
{
  "source": "component:...",
  "relation": "DEPENDS_ON",
  "target": "dependency:...",
  "evidence": [
    {
      "chunk_id": "...",
      "source": "...",
      "excerpt": "..."
    }
  ]
}
```

**Verificación**

Ejecuta la siguiente validación de integridad:

```bash
python - <<'PY'
import json

with open("data/graph/technical_graph.json", encoding="utf-8") as file:
    graph = json.load(file)

node_ids = {node["id"] for node in graph["nodes"]}
errors = []

for edge in graph["edges"]:
    if edge["source"] not in node_ids:
        errors.append(f"Origen inexistente: {edge['source']}")
    if edge["target"] not in node_ids:
        errors.append(f"Destino inexistente: {edge['target']}")
    if not edge.get("evidence"):
        errors.append(f"Arista sin evidencia: {edge['id']}")

if errors:
    print("\n".join(errors))
    raise SystemExit(1)

print("Integridad del grafo validada.")
PY
```

La salida debe ser:

```text
Integridad del grafo validada.
```

---

### Paso 4. Implementar el recuperador GraphRAG con BFS limitado

**Objetivo:** desarrollar un recuperador que detecte entidades candidatas en una pregunta, expanda sus vecinos hasta dos saltos y devuelva hechos con evidencia documental.

**Instrucciones**

1. Crea el archivo `src/graphrag/graph_retriever.py`.

2. Copia la siguiente implementación:

   ```python
   from __future__ import annotations

   import json
   import re
   from collections import defaultdict, deque
   from pathlib import Path
   from typing import Any


   ALLOWED_RELATIONS = {
       "DEPENDS_ON",
       "EXPOSES",
       "AUTHENTICATES_WITH",
       "RESOLVES",
       "DEPLOYED_ON",
       "DOCUMENTED_IN",
   }


   class GraphRetriever:
       """Recuperador híbrido basado en coincidencia de entidades y BFS."""

       def __init__(self, graph_path: str | Path) -> None:
           with Path(graph_path).open(encoding="utf-8") as file:
               self.graph: dict[str, Any] = json.load(file)

           self.nodes = {
               node["id"]: node
               for node in self.graph.get("nodes", [])
           }
           self.edges = self.graph.get("edges", [])
           self.chunks = self.graph.get("chunks", {})

           self.adjacency: dict[str, list[dict[str, Any]]] = defaultdict(list)

           for edge in self.edges:
               if edge["relation"] not in ALLOWED_RELATIONS:
                   continue

               self.adjacency[edge["source"]].append(edge)
               self.adjacency[edge["target"]].append(edge)

       @staticmethod
       def _normalize(value: str) -> str:
           return re.sub(
               r"[^a-z0-9]+",
               " ",
               value.lower(),
           ).strip()

       @staticmethod
       def _tokens(value: str) -> set[str]:
           return {
               token
               for token in re.findall(r"[a-zA-Z0-9_-]{2,}", value.lower())
               if token not in {
                   "que", "como", "para", "con", "del", "las",
                   "los", "una", "uno", "por", "the", "and",
               }
           }

       def identify_candidate_entities(
           self,
           question: str,
           limit: int = 5,
       ) -> list[dict[str, Any]]:
           normalized_question = self._normalize(question)
           question_tokens = self._tokens(question)
           candidates: list[tuple[float, dict[str, Any]]] = []

           for node in self.nodes.values():
               normalized_name = self._normalize(
                   node.get("normalized_name") or node["name"]
               )
               name_tokens = self._tokens(node["name"])

               exact_match = (
                   normalized_name in normalized_question
                   and len(normalized_name) > 1
               )
               overlap = len(question_tokens.intersection(name_tokens))

               score = 0.0
               if exact_match:
                   score += 10.0
               score += overlap

               if score > 0:
                   candidates.append((score, node))

           candidates.sort(
               key=lambda item: (item[0], item[1]["name"]),
               reverse=True,
           )

           return [
               {
                   "id": node["id"],
                   "name": node["name"],
                   "type": node["type"],
                   "score": score,
               }
               for score, node in candidates[:limit]
           ]

       def expand_neighbors(
           self,
           seed_ids: list[str],
           max_hops: int = 2,
           max_edges: int = 20,
       ) -> list[dict[str, Any]]:
           if max_hops < 1:
               return []

           queue = deque((node_id, 0) for node_id in seed_ids)
           visited_nodes = set(seed_ids)
           visited_edges: set[str] = set()
           facts: list[dict[str, Any]] = []

           while queue and len(facts) < max_edges:
               current_node, depth = queue.popleft()

               if depth >= max_hops:
                   continue

               for edge in self.adjacency.get(current_node, []):
                   if edge["id"] in visited_edges:
                       continue

                   visited_edges.add(edge["id"])

                   neighbor = (
                       edge["target"]
                       if edge["source"] == current_node
                       else edge["source"]
                   )

                   source_node = self.nodes[edge["source"]]
                   target_node = self.nodes[edge["target"]]

                   facts.append(
                       {
                           "edge_id": edge["id"],
                           "hop": depth + 1,
                           "source": {
                               "id": source_node["id"],
                               "name": source_node["name"],
                               "type": source_node["type"],
                           },
                           "relation": edge["relation"],
                           "target": {
                               "id": target_node["id"],
                               "name": target_node["name"],
                               "type": target_node["type"],
                           },
                           "evidence": edge["evidence"],
                       }
                   )

                   if neighbor not in visited_nodes:
                       visited_nodes.add(neighbor)
                       queue.append((neighbor, depth + 1))

                   if len(facts) >= max_edges:
                       break

           return facts

       def retrieve_source_chunks(
           self,
           facts: list[dict[str, Any]],
           question: str,
           limit: int = 6,
       ) -> list[dict[str, Any]]:
           referenced_chunk_ids: set[str] = set()

           for fact in facts:
               for evidence in fact["evidence"]:
                   referenced_chunk_ids.add(evidence["chunk_id"])

           question_tokens = self._tokens(question)
           ranked_chunks: list[tuple[int, dict[str, Any]]] = []

           for chunk_id in referenced_chunk_ids:
               chunk = self.chunks.get(chunk_id)

               if not chunk:
                   continue

               score = len(
                   question_tokens.intersection(
                       self._tokens(chunk.get("text", ""))
                   )
               )

               ranked_chunks.append((score, chunk))

           ranked_chunks.sort(
               key=lambda item: (item[0], item[1]["chunk_id"]),
               reverse=True,
           )

           return [
               {
                   "chunk_id": chunk["chunk_id"],
                   "source": chunk["source"],
                   "text": chunk["text"],
               }
               for _, chunk in ranked_chunks[:limit]
           ]

       def retrieve(
           self,
           question: str,
           max_hops: int = 2,
           max_entities: int = 5,
           max_edges: int = 20,
           max_chunks: int = 6,
       ) -> dict[str, Any]:
           if max_hops > 2:
               raise ValueError(
                   "La expansión está limitada a un máximo de dos saltos."
               )

           candidates = self.identify_candidate_entities(
               question,
               limit=max_entities,
           )

           seed_ids = [candidate["id"] for candidate in candidates]
           facts = self.expand_neighbors(
               seed_ids,
               max_hops=max_hops,
               max_edges=max_edges,
           )
           chunks = self.retrieve_source_chunks(
               facts,
               question,
               limit=max_chunks,
           )

           return {
               "question": question,
               "candidate_entities": candidates,
               "facts": facts,
               "source_chunks": chunks,
               "limits": {
                   "max_hops": max_hops,
                   "max_entities": max_entities,
                   "max_edges": max_edges,
                   "max_chunks": max_chunks,
               },
           }


   if __name__ == "__main__":
       import argparse

       parser = argparse.ArgumentParser(
           description="Ejecuta una recuperación GraphRAG sobre el grafo técnico."
       )
       parser.add_argument(
           "--graph",
           default="data/graph/technical_graph.json",
       )
       parser.add_argument("--question", required=True)
       parser.add_argument("--max-hops", type=int, default=2)
       args = parser.parse_args()

       retriever = GraphRetriever(args.graph)
       result = retriever.retrieve(
           args.question,
           max_hops=args.max_hops,
       )

       print(json.dumps(result, ensure_ascii=False, indent=2))
   ```

3. Ejecuta una consulta basada en una entidad que exista en tu corpus. Sustituye `NOMBRE_ENTIDAD` por una entidad mostrada en el paso anterior:

   ```bash
   python -m src.graphrag.graph_retriever \
     --graph data/graph/technical_graph.json \
     --question "¿De qué depende NOMBRE_ENTIDAD y dónde se despliega?" \
     --max-hops 2
   ```

4. Observa las secciones de la respuesta JSON:

   - `candidate_entities`: nodos encontrados en la pregunta.
   - `facts`: tripletas recuperadas y evidencia de cada relación.
   - `source_chunks`: textos fuente asociados a las aristas.
   - `limits`: límites operativos aplicados a la consulta.

**Salida esperada**

La estructura de salida debe contener hechos trazables similares a:

```json
{
  "source": {
    "name": "ServicioPagos",
    "type": "Service"
  },
  "relation": "DEPENDS_ON",
  "target": {
    "name": "Redis",
    "type": "Dependency"
  },
  "evidence": [
    {
      "chunk_id": "chunk-014",
      "source": "arquitectura.md",
      "excerpt": "ServicioPagos depende de Redis para almacenar..."
    }
  ]
}
```

**Verificación**

Comprueba que la recuperación no excede dos saltos:

```bash
python - <<'PY'
from src.graphrag.graph_retriever import GraphRetriever

retriever = GraphRetriever("data/graph/technical_graph.json")
result = retriever.retrieve(
    "¿Qué dependencias tiene el servicio principal?",
    max_hops=2,
)

assert all(fact["hop"] <= 2 for fact in result["facts"])
assert all(fact["evidence"] for fact in result["facts"])

print("BFS limitado y evidencia por hecho: validado.")
PY
```

La salida debe ser:

```text
BFS limitado y evidencia por hecho: validado.
```

---

### Paso 5. Crear `GraphKnowledgeRetrievalSkill`

**Objetivo:** encapsular el recuperador GraphRAG en una skill reutilizable que entregue contexto compacto para el generador y evidencia verificable para el harness.

**Instrucciones**

1. Crea el archivo `src/graphrag/graph_knowledge_retrieval_skill.py`.

2. Copia el siguiente código:

   ```python
   from __future__ import annotations

   from typing import Any

   from src.graphrag.graph_retriever import GraphRetriever


   class GraphKnowledgeRetrievalSkill:
       """Skill de recuperación híbrida basada en grafo y evidencia textual."""

       def __init__(self, graph_path: str) -> None:
           self.retriever = GraphRetriever(graph_path)

       @staticmethod
       def _format_fact(fact: dict[str, Any]) -> str:
           source = fact["source"]["name"]
           relation = fact["relation"]
           target = fact["target"]["name"]

           sources = sorted(
               {
                   evidence["source"]
                   for evidence in fact["evidence"]
               }
           )

           return (
               f"- {source} --{relation}--> {target}. "
               f"Fuentes: {', '.join(sources)}."
           )

       @staticmethod
       def _format_chunk(chunk: dict[str, Any]) -> str:
           compact_text = " ".join(chunk["text"].split())
           return (
               f"- [{chunk['chunk_id']}] {compact_text} "
               f"(Fuente: {chunk['source']})"
           )

       def run(
           self,
           question: str,
           max_hops: int = 2,
       ) -> dict[str, Any]:
           retrieval = self.retriever.retrieve(
               question=question,
               max_hops=max_hops,
           )

           facts = retrieval["facts"]
           chunks = retrieval["source_chunks"]

           evidence_lines = [
               self._format_fact(fact)
               for fact in facts
           ]

           chunk_lines = [
               self._format_chunk(chunk)
               for chunk in chunks
           ]

           if not evidence_lines:
               evidence_lines.append(
                   "- No se recuperaron relaciones explícitas respaldadas "
                   "por el grafo."
               )

           context = "\n".join(
               [
                   "HECHOS DEL GRAFO:",
                   *evidence_lines,
                   "",
                   "FRAGMENTOS DOCUMENTALES:",
                   *chunk_lines,
               ]
           )

           return {
               "question": question,
               "context": context,
               "candidate_entities": retrieval["candidate_entities"],
               "facts": facts,
               "source_chunks": chunks,
               "grounding_rules": [
                   "Usar únicamente los hechos y fragmentos proporcionados.",
                   "No afirmar relaciones ausentes del campo facts.",
                   "Indicar incertidumbre si la evidencia es insuficiente.",
                   "Citar las fuentes documentales utilizadas.",
               ],
           }
   ```

3. Crea un script de prueba manual llamado `src/graphrag/demo_graph_skill.py`:

   ```python
   from __future__ import annotations

   import json

   from src.graphrag.graph_knowledge_retrieval_skill import (
       GraphKnowledgeRetrievalSkill,
   )


   skill = GraphKnowledgeRetrievalSkill(
       "data/graph/technical_graph.json"
   )

   result = skill.run(
       "¿Qué servicio depende de una dependencia y qué procedimiento resuelve sus errores?"
   )

   print(result["context"])
   print("\n--- Evidencia estructurada ---")
   print(
       json.dumps(
           {
               "candidate_entities": result["candidate_entities"],
               "facts": result["facts"],
           },
           ensure_ascii=False,
           indent=2,
       )
   )
   ```

4. Ejecuta la demostración:

   ```bash
   python -m src.graphrag.demo_graph_skill
   ```

5. Integra esta skill en el punto donde el proyecto anterior invocaba `KnowledgeRetrievalSkill`. La integración debe respetar esta separación:

   ```text
   Pregunta
      │
      ▼
   GraphKnowledgeRetrievalSkill
      │
      ├── Hechos del grafo con evidencia
      ├── Chunks fuente
      │
      ▼
   Generador o modelo de lenguaje
      │
      ▼
   Respuesta con fuentes
   ```

6. Usa el siguiente mensaje de sistema para la fase generativa:

   ```text
   Eres un asistente técnico basado en evidencia documental.

   Responde únicamente con base en los HECHOS DEL GRAFO y los FRAGMENTOS
   DOCUMENTALES proporcionados. No inventes entidades, relaciones, causas,
   configuraciones ni procedimientos.

   Si las evidencias no permiten responder completamente, indícalo de forma
   explícita. Cuando realices una conclusión a partir de varias relaciones,
   preséntala como una inferencia respaldada por las evidencias.

   Incluye al final una sección titulada "Fuentes" con los documentos o chunks
   utilizados.
   ```

**Salida esperada**

La skill debe producir un campo `context` con dos secciones:

```text
HECHOS DEL GRAFO:
- ServicioPagos --DEPENDS_ON--> Redis. Fuentes: arquitectura.md.

FRAGMENTOS DOCUMENTALES:
- [chunk-014] ServicioPagos depende de Redis... (Fuente: arquitectura.md)
```

**Verificación**

Ejecuta la siguiente comprobación:

```bash
python - <<'PY'
from src.graphrag.graph_knowledge_retrieval_skill import (
    GraphKnowledgeRetrievalSkill,
)

skill = GraphKnowledgeRetrievalSkill("data/graph/technical_graph.json")
result = skill.run("¿Qué relaciones están documentadas para los servicios?")

assert "HECHOS DEL GRAFO:" in result["context"]
assert "FRAGMENTOS DOCUMENTALES:" in result["context"]
assert isinstance(result["facts"], list)
assert isinstance(result["source_chunks"], list)

print("Contrato de GraphKnowledgeRetrievalSkill validado.")
PY
```

---

### Paso 6. Crear pruebas automatizadas de grounding y recorrido

**Objetivo:** asegurar que el recuperador devuelve relaciones existentes, mantiene evidencia y respeta el límite de expansión.

**Instrucciones**

1. Crea el archivo `tests/test_graph_retriever.py`.

2. Copia el siguiente contenido:

   ```python
   from __future__ import annotations

   import json
   from pathlib import Path

   import pytest

   from src.graphrag.graph_retriever import GraphRetriever


   @pytest.fixture()
   def graph_file(tmp_path: Path) -> Path:
       graph = {
           "schema_version": "1.0",
           "nodes": [
               {
                   "id": "service:atlas",
                   "type": "Service",
                   "name": "Atlas",
                   "normalized_name": "atlas",
                   "evidence": [],
               },
               {
                   "id": "dependency:redis",
                   "type": "Dependency",
                   "name": "Redis",
                   "normalized_name": "redis",
                   "evidence": [],
               },
               {
                   "id": "procedure:reiniciar-cache",
                   "type": "Procedure",
                   "name": "Reiniciar Cache",
                   "normalized_name": "reiniciar-cache",
                   "evidence": [],
               },
           ],
           "edges": [
               {
                   "id": "edge:atlas:depends_on:redis",
                   "source": "service:atlas",
                   "relation": "DEPENDS_ON",
                   "target": "dependency:redis",
                   "evidence": [
                       {
                           "chunk_id": "chunk-001",
                           "source": "arquitectura.md",
                           "excerpt": "Atlas depende de Redis.",
                       }
                   ],
               },
               {
                   "id": "edge:redis:resolves:reiniciar-cache",
                   "source": "dependency:redis",
                   "relation": "RESOLVES",
                   "target": "procedure:reiniciar-cache",
                   "evidence": [
                       {
                           "chunk_id": "chunk-002",
                           "source": "runbook.md",
                           "excerpt": "Redis se resuelve con Reiniciar Cache.",
                       }
                   ],
               },
           ],
           "chunks": {
               "chunk-001": {
                   "chunk_id": "chunk-001",
                   "source": "arquitectura.md",
                   "text": "Atlas depende de Redis.",
               },
               "chunk-002": {
                   "chunk_id": "chunk-002",
                   "source": "runbook.md",
                   "text": "Redis se resuelve con Reiniciar Cache.",
               },
           },
       }

       path = tmp_path / "technical_graph.json"
       path.write_text(
           json.dumps(graph, ensure_ascii=False),
           encoding="utf-8",
       )
       return path


   def test_recupera_relaciones_existentes_con_evidencia(graph_file: Path) -> None:
       retriever = GraphRetriever(graph_file)

       result = retriever.retrieve(
           "¿De qué depende Atlas y cómo se resuelve Redis?",
           max_hops=2,
       )

       relations = {fact["relation"] for fact in result["facts"]}

       assert "DEPENDS_ON" in relations
       assert "RESOLVES" in relations
       assert all(fact["evidence"] for fact in result["facts"])
       assert all(fact["hop"] <= 2 for fact in result["facts"])


   def test_no_inventa_relaciones_ausentes(graph_file: Path) -> None:
       retriever = GraphRetriever(graph_file)

       result = retriever.retrieve(
           "¿Dónde se despliega Atlas?",
           max_hops=2,
       )

       relations = {fact["relation"] for fact in result["facts"]}

       assert "DEPLOYED_ON" not in relations


   def test_rechaza_mas_de_dos_saltos(graph_file: Path) -> None:
       retriever = GraphRetriever(graph_file)

       with pytest.raises(ValueError, match="máximo de dos saltos"):
           retriever.retrieve(
               "¿De qué depende Atlas?",
               max_hops=3,
           )
   ```

3. Ejecuta las pruebas específicas:

   ```bash
   pytest -q tests/test_graph_retriever.py
   ```

4. Si el harness del laboratorio `02-00-04` permite registrar casos de evaluación en JSON, crea `data/graph/evaluation_cases.json`:

   ```json
   [
     {
       "id": "graph-grounding-001",
       "question": "¿De qué depende Atlas y cómo se resuelve Redis?",
       "expected_relations": [
         "DEPENDS_ON",
         "RESOLVES"
       ],
       "forbidden_relations": [
         "DEPLOYED_ON",
         "AUTHENTICATES_WITH"
       ],
       "require_sources": true,
       "max_hops": 2
     },
     {
       "id": "graph-grounding-002",
       "question": "¿Dónde se despliega Atlas?",
       "expected_relations": [],
       "forbidden_relations": [
         "DEPLOYED_ON"
       ],
       "require_sources": true,
       "max_hops": 2
     }
   ]
   ```

5. Adapta el adaptador del harness existente para comprobar estas reglas:

   - Cada relación de la respuesta debe existir en `result["facts"]`.
   - Cada hecho debe contener evidencia.
   - Cada evidencia debe referenciar un `chunk_id` existente.
   - La respuesta generada debe citar al menos una fuente si hay hechos recuperados.
   - Una relación prohibida no puede aparecer en la respuesta ni en los hechos.

**Salida esperada**

Las pruebas deben finalizar correctamente:

```text
3 passed
```

**Verificación**

Ejecuta la suite de pruebas del laboratorio:

```bash
pytest -q tests/test_graph_retriever.py | tee reports/lab-03-00-02-tests.txt
```

Comprueba el reporte:

```bash
cat reports/lab-03-00-02-tests.txt
```

---

### Paso 7. Ejecutar una validación integral con el grafo real

**Objetivo:** validar que el grafo generado desde los documentos reales puede responder una consulta multi-relación sin introducir hechos no respaldados.

**Instrucciones**

1. Identifica una cadena de relaciones real en el archivo del grafo:

   ```bash
   python - <<'PY'
   import json

   with open("data/graph/technical_graph.json", encoding="utf-8") as file:
       graph = json.load(file)

   nodes = {node["id"]: node["name"] for node in graph["nodes"]}

   for edge in graph["edges"]:
       print(
           f"{nodes[edge['source']]} --{edge['relation']}--> "
           f"{nodes[edge['target']]}"
       )
   PY
   ```

2. Formula una pregunta que requiera al menos dos conexiones. Algunos ejemplos, que debes adaptar a tus entidades reales:

   ```text
   ¿Qué dependencia utiliza ServicioPagos y qué procedimiento está asociado a esa dependencia?
   ```

   ```text
   ¿Qué API expone el servicio y con qué configuración se autentica?
   ```

   ```text
   ¿Qué componente se despliega en una plataforma y qué dependencia necesita?
   ```

3. Ejecuta la skill con la pregunta seleccionada:

   ```bash
   python - <<'PY'
   import json
   from src.graphrag.graph_knowledge_retrieval_skill import (
       GraphKnowledgeRetrievalSkill,
   )

   question = "REEMPLAZA ESTA PREGUNTA POR UNA CONSULTA BASADA EN TU GRAFO"

   skill = GraphKnowledgeRetrievalSkill("data/graph/technical_graph.json")
   result = skill.run(question, max_hops=2)

   print(result["context"])

   with open(
       "reports/graph_retrieval_result.json",
       "w",
       encoding="utf-8",
   ) as file:
       json.dump(result, file, ensure_ascii=False, indent=2)
   PY
   ```

4. Revisa el archivo de resultados:

   ```bash
   python -m json.tool reports/graph_retrieval_result.json | less
   ```

5. Verifica manualmente que:

   - Cada hecho del grafo está presente literalmente o de forma inequívoca en el chunk indicado.
   - La respuesta no atribuye una relación a una entidad no conectada por una arista.
   - La expansión no supera dos saltos.
   - Los chunks fuente pertenecen a evidencias de las aristas recuperadas.

**Salida esperada**

Debes obtener un reporte en:

```text
reports/graph_retrieval_result.json
```

El reporte debe contener una lista de hechos conectados y sus documentos fuente.

**Verificación**

Ejecuta esta comprobación final de grounding estructural:

```bash
python - <<'PY'
import json

with open("reports/graph_retrieval_result.json", encoding="utf-8") as file:
    result = json.load(file)

source_chunk_ids = {
    chunk["chunk_id"]
    for chunk in result["source_chunks"]
}

for fact in result["facts"]:
    assert fact["hop"] <= 2, f"Salto inválido: {fact}"
    assert fact["evidence"], f"Hecho sin evidencia: {fact}"
    for evidence in fact["evidence"]:
        assert evidence["chunk_id"] in source_chunk_ids, (
            f"Chunk no recuperado para evidencia: {evidence['chunk_id']}"
        )

print("Grounding estructural validado.")
PY
```

---

## Validación y pruebas

Ejecuta la siguiente secuencia completa desde la raíz del repositorio:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate

python -m src.graphrag.extract_graph_entities \
  --input data/processed/chunks_with_embeddings.jsonl \
  --output data/graph/technical_graph.json

python -m json.tool data/graph/technical_graph.json > /dev/null

pytest -q tests/test_graph_retriever.py

python - <<'PY'
import json

with open("data/graph/technical_graph.json", encoding="utf-8") as file:
    graph = json.load(file)

node_ids = {node["id"] for node in graph["nodes"]}

assert graph["schema_version"] == "1.0"

for edge in graph["edges"]:
    assert edge["source"] in node_ids
    assert edge["target"] in node_ids
    assert edge["relation"] in {
        "DEPENDS_ON",
        "EXPOSES",
        "AUTHENTICATES_WITH",
        "RESOLVES",
        "DEPLOYED_ON",
        "DOCUMENTED_IN",
    }
    assert edge["evidence"]

print("Validación integral del artefacto GraphRAG completada.")
PY
```

### Criterios de aceptación

Considera el laboratorio completado cuando se cumplan todos los criterios siguientes:

| Criterio | Evidencia |
|---|---|
| El grafo se genera correctamente | Existe `data/graph/technical_graph.json` y es JSON válido |
| Las entidades están normalizadas | Los nodos tienen `id`, `type`, `name` y `normalized_name` |
| Las aristas son trazables | Cada arista tiene `evidence` con `chunk_id` y `source` |
| Se respetan los tipos de relación | Solo se usan las seis relaciones definidas en el alcance |
| La expansión está controlada | `max_hops` no puede ser superior a 2 |
| El recuperador entrega texto fuente | `source_chunks` contiene los chunks asociados a los hechos |
| El grounding está probado | Las pruebas automatizadas finalizan sin errores |
| No se inventan relaciones | Una relación inexistente no aparece en los resultados ni en una respuesta generada |

### Control de cambios

Revisa los archivos modificados:

```bash
git status
```

Añade los artefactos relevantes, evitando archivos temporales o secretos:

```bash
git add \
  src/graphrag \
  tests/test_graph_retriever.py \
  data/graph/technical_graph.json \
  data/graph/evaluation_cases.json \
  reports/lab-03-00-02-tests.txt
```

Revisa el diff antes de confirmar:

```bash
git diff --cached
```

Realiza el commit utilizando la convención de mensajes indicada por tu instructor o por la política vigente del repositorio.

> No incluyas claves de Azure OpenAI, archivos `.env`, resultados que contengan secretos ni archivos de entorno virtual en el commit.

## Solución de problemas

### Problema 1. El grafo se genera, pero no contiene aristas

**Síntoma:** el comando de extracción informa nodos, pero `edge_count` es `0`.

**Causa probable:** los patrones de relación son deliberadamente conservadores y requieren frases explícitas, por ejemplo “ServicioPagos depende de Redis”. La documentación puede usar una redacción distinta, tablas, listas o sinónimos no contemplados.

**Solución:**

1. Busca expresiones relacionales reales en los chunks:

   ```bash
   grep -inE "depende de|expone|autentica|despliega|resuelve|documentado" \
     data/processed/chunks_with_embeddings.jsonl | head -n 20
   ```

2. Añade patrones específicos y revisables a `RELATION_PATTERNS` en `extract_graph_entities.py`.
3. Mantén el principio de extracción conservadora: agrega solo patrones que representen relaciones explícitas y que puedan asociarse a un chunk fuente.
4. Regenera el grafo y revisa las nuevas aristas.

### Problema 2. La consulta devuelve entidades candidatas, pero no devuelve chunks fuente

**Síntoma:** `candidate_entities` contiene nodos, pero `source_chunks` está vacío o los hechos no tienen evidencia útil.

**Causa probable:** el grafo fue modificado manualmente, se generó con chunks sin `chunk_id`, o las referencias de evidencia apuntan a identificadores inexistentes en la sección `chunks`.

**Solución:**

1. Verifica que los chunks de entrada tengan identificadores no vacíos:

   ```bash
   python - <<'PY'
   import json
   from src.graphrag.contracts import get_chunk_id

   invalid = 0

   with open("data/processed/chunks_with_embeddings.jsonl", encoding="utf-8") as file:
       for line in file:
           chunk = json.loads(line)
           if not get_chunk_id(chunk):
               invalid += 1

   print(f"Chunks sin identificador: {invalid}")
   PY
   ```

2. Ajusta `get_chunk_id()` en `src/graphrag/contracts.py` si tu pipeline baseline utiliza otro nombre de campo.
3. Regenera `technical_graph.json`; no edites manualmente las referencias de `chunk_id`.
4. Ejecuta nuevamente las pruebas de integridad y la suite `pytest`.

## Limpieza

Este laboratorio no crea recursos de Azure, contenedores ni bases de datos. Conserva los artefactos versionables necesarios para las prácticas posteriores:

```text
data/graph/technical_graph.json
data/graph/evaluation_cases.json
src/graphrag/
tests/test_graph_retriever.py
reports/lab-03-00-02-tests.txt
```

Si creaste archivos temporales de inspección, elimínalos:

```bash
rm -f reports/graph_retrieval_result.json
```

No elimines `data/graph/technical_graph.json`, ya que será el artefacto de entrada para extensiones posteriores hacia una base de datos especializada o una implementación GraphRAG más avanzada.

## Resumen

En este laboratorio construiste una capa GraphRAG sobre el pipeline de recuperación semántica existente. Implementaste extracción normalizada de entidades técnicas, relaciones explícitas con evidencia documental y persistencia en un archivo JSON portable.

También desarrollaste `GraphRetriever` con detección de entidades y recorrido BFS limitado a dos saltos, y encapsulaste el flujo en `GraphKnowledgeRetrievalSkill`. La validación automatizada comprueba que las relaciones recuperadas existan realmente en el grafo, que dispongan de evidencia y que el sistema no exceda los límites de expansión establecidos.

### Recursos opcionales

- [Microsoft Research GraphRAG](https://www.microsoft.com/en-us/research/project/graphrag/)
- [Repositorio oficial Microsoft GraphRAG](https://github.com/microsoft/graphrag)
- [Azure AI Search: recuperación aumentada por generación](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview)
- [Neo4j GraphRAG](https://neo4j.com/developer/genai-ecosystem/graphrag/)

---

# 2. Práctica 3. Implementar persistencia vectorial para almacenar y recuperar contexto mediante búsquedas semánticas.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 40 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica sustituirá la recuperación basada en el archivo local `chunks_with_embeddings.jsonl` por una colección persistente de Qdrant. Cargará embeddings y metadatos documentales en la colección `technical_chunks`, implementará filtros por documento y tipo de entidad, y conectará el recuperador persistente con `GraphKnowledgeRetrievalSkill`.

La representación lógica de relaciones permanecerá en `technical_graph.json`. Por tanto, la solución resultante seguirá el patrón GraphRAG híbrido: Qdrant recupera evidencia textual semántica y el grafo aporta relaciones explícitas entre entidades.

## Objetivos de aprendizaje

- [ ] Desplegar Qdrant 1.9.1 mediante Docker Compose con almacenamiento persistente local.
- [ ] Crear una colección vectorial con embeddings de dimensión 1536 y distancia Cosine.
- [ ] Cargar chunks, embeddings, metadatos y referencias de entidades en Qdrant.
- [ ] Implementar consultas semánticas top-k con filtros por documento y tipo de entidad.
- [ ] Comparar funcionalmente la recuperación persistente contra el baseline JSONL mediante consultas de regresión.

## Prerrequisitos

### Conocimientos requeridos

- Comprensión básica de embeddings, similitud Cosine, chunks y recuperación top-k.
- Comprensión del patrón RAG y de la diferencia entre evidencia textual y relaciones de grafo.
- Conocimiento básico de Docker Compose y archivos JSON/JSONL.
- Haber completado los laboratorios `03-00-01` y `03-00-02`.

### Acceso y artefactos requeridos

Verifique que dispone de los siguientes elementos:

- Repositorio local `~/genai-agent-labs`.
- Entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Archivo `data/chunks_with_embeddings.jsonl` generado en el laboratorio `03-00-01`.
- Archivo `data/technical_graph.json` generado en el laboratorio `03-00-02`.
- Docker Engine 26.1.4 o compatible.
- Docker Compose Plugin 2.27.0 o compatible.
- Credenciales válidas de Azure OpenAI para generar embeddings de consultas.
- Despliegue compatible con `text-embedding-3-small`, cuya dimensión esperada es `1536`.

> **Importante:** esta práctica usa Qdrant únicamente como almacenamiento vectorial. No use una base de datos relacional ni cree una base de datos llamada `genai_agents_db`.

## Entorno del laboratorio

### Recursos de hardware y software

| Recurso | Requisito |
|---|---|
| Memoria RAM | 16 GB mínimo; 32 GB recomendado |
| Espacio libre | 20 GB en SSD recomendado |
| Sistema operativo de referencia | Ubuntu 22.04.4 LTS |
| Python | 3.12.1 |
| Docker Engine | 26.1.4 |
| Docker Compose Plugin | 2.27.0 |
| Qdrant | 1.9.1 |
| Cliente Python de Qdrant | 1.9.1 |
| Modelo de embeddings | `text-embedding-3-small` |
| Tamaño vectorial | 1536 |
| Métrica vectorial | Cosine |

### Preparación inicial

1. Abra una terminal y vaya al directorio obligatorio del curso.

   ```bash
   cd ~/genai-agent-labs
   ```

2. Active el entorno virtual compartido.

   ```bash
   source .venv/bin/activate
   ```

3. Compruebe que los artefactos de los laboratorios previos existen.

   ```bash
   ls -lh data/chunks_with_embeddings.jsonl data/technical_graph.json
   ```

4. Compruebe las versiones de Docker y Docker Compose.

   ```bash
   docker --version
   docker compose version
   ```

5. Instale el cliente de Qdrant en la versión requerida.

   ```bash
   pip install qdrant-client==1.9.1
   ```

6. Cree los directorios de trabajo necesarios.

   ```bash
   mkdir -p src/retrieval src/scripts src/skills config reports data/qdrant_storage
   touch src/__init__.py src/retrieval/__init__.py src/scripts/__init__.py src/skills/__init__.py
   ```

7. Añada los datos persistentes de Qdrant y secretos locales a `.gitignore`.

   ```bash
   cat >> .gitignore <<'EOF'

   # Qdrant y secretos locales
   data/qdrant_storage/
   .env
   reports/qdrant_load_report.json
   reports/retrieval_regression.json
   EOF
   ```

8. Compruebe que su archivo `.env` contiene las variables necesarias. No muestre secretos ni incluya valores reales en archivos versionados.

   ```bash
   grep -E '^(AZURE_OPENAI_ENDPOINT|AZURE_OPENAI_API_KEY|AZURE_OPENAI_API_VERSION|AZURE_OPENAI_EMBEDDING_DEPLOYMENT)=' .env
   ```

   El archivo `.env` debe seguir una estructura equivalente a esta:

   ```dotenv
   AZURE_OPENAI_ENDPOINT=https://<recurso>.openai.azure.com/
   AZURE_OPENAI_API_KEY=<secreto-no-versionado>
   AZURE_OPENAI_API_VERSION=2024-02-15-preview
   AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small
   QDRANT_URL=http://127.0.0.1:6333
   ```

> **Seguridad:** no escriba claves de Azure OpenAI en código Python, archivos JSON, Docker Compose ni commits de Git.

## Procedimiento paso a paso

### Paso 1. Desplegar Qdrant con almacenamiento persistente

**Objetivo:** iniciar una instancia local de Qdrant 1.9.1 y conservar los índices entre reinicios del contenedor.

**Instrucciones:**

1. Cree el archivo `docker-compose.qdrant.yml`.

   ```bash
   cat > docker-compose.qdrant.yml <<'EOF'
   services:
     qdrant:
       image: qdrant/qdrant:v1.9.1
       container_name: genai-agent-qdrant
       restart: unless-stopped
       ports:
         - "127.0.0.1:6333:6333"
         - "127.0.0.1:6334:6334"
       volumes:
         - ./data/qdrant_storage:/qdrant/storage
   EOF
   ```

2. Inicie Qdrant en segundo plano.

   ```bash
   docker compose -f docker-compose.qdrant.yml up -d
   ```

3. Consulte el estado del contenedor.

   ```bash
   docker compose -f docker-compose.qdrant.yml ps
   ```

4. Valide la API REST de Qdrant.

   ```bash
   curl -s http://127.0.0.1:6333/healthz
   ```

5. Consulte la información de la instancia.

   ```bash
   curl -s http://127.0.0.1:6333/ | python -m json.tool
   ```

**Salida esperada:**

El comando de estado debe indicar que el servicio `genai-agent-qdrant` está en ejecución. La consulta de salud debe devolver una respuesta satisfactoria, normalmente:

```json
{"title":"qdrant - vector search engine","version":"1.9.1"}
```

La respuesta exacta puede variar ligeramente según la imagen utilizada.

**Verificación:**

Ejecute:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:6333/collections
```

El resultado esperado es:

```text
200
```

---

### Paso 2. Revisar y normalizar los artefactos de entrada

**Objetivo:** comprobar que los chunks contienen embeddings de dimensión 1536 y que el grafo está disponible como fuente de referencias de entidades.

**Instrucciones:**

1. Inspeccione una línea del archivo JSONL sin imprimir innecesariamente el embedding completo.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   path = Path("data/chunks_with_embeddings.jsonl")
   first = json.loads(path.read_text(encoding="utf-8").splitlines()[0])

   print("Campos:", sorted(first.keys()))
   embedding = first.get("embedding") or first.get("embeddings")
   print("Dimensión del embedding:", len(embedding) if embedding else "no encontrado")
   print("chunk_id:", first.get("chunk_id"))
   print("document_id:", first.get("document_id"))
   print("section_title:", first.get("section_title"))
   PY
   ```

2. Inspeccione la estructura superior del artefacto de grafo.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   graph = json.loads(Path("data/technical_graph.json").read_text(encoding="utf-8"))
   print("Tipo raíz:", type(graph).__name__)

   if isinstance(graph, dict):
       print("Campos raíz:", sorted(graph.keys()))
       for key in ("nodes", "entities", "edges", "relationships", "relations"):
           value = graph.get(key)
           if isinstance(value, list):
               print(f"{key}: {len(value)} elementos")
   PY
   ```

3. Si el archivo JSONL no contiene los campos `entity_refs` o `entity_types`, no modifique los chunks manualmente. El cargador que implementará en el siguiente paso derivará estas referencias desde el grafo cuando estén disponibles.

4. Cree un directorio de configuración para las consultas de regresión.

   ```bash
   mkdir -p config
   ```

**Salida esperada:**

La dimensión del embedding debe ser:

```text
Dimensión del embedding: 1536
```

El grafo debe indicar una estructura con nodos, entidades, aristas o relaciones. Los nombres exactos dependen de la implementación realizada en el laboratorio `03-00-02`.

**Verificación:**

Ejecute el siguiente comando:

```bash
python - <<'PY'
import json
from pathlib import Path

line = Path("data/chunks_with_embeddings.jsonl").read_text(encoding="utf-8").splitlines()[0]
item = json.loads(line)
embedding = item.get("embedding") or item.get("embeddings")

assert embedding is not None, "No se encontró embedding ni embeddings."
assert len(embedding) == 1536, f"Se esperaban 1536 dimensiones y se encontraron {len(embedding)}."
print("Validación de embedding completada.")
PY
```

---

### Paso 3. Implementar el cargador de chunks y embeddings en Qdrant

**Objetivo:** crear la colección `technical_chunks`, almacenar los vectores y preservar como payload los metadatos, identificadores documentales y referencias de entidades.

**Instrucciones:**

1. Cree el archivo `src/scripts/load_qdrant.py`.

   ```bash
   cat > src/scripts/load_qdrant.py <<'PY'
   import json
   import os
   import uuid
   from pathlib import Path
   from typing import Any

   from dotenv import load_dotenv
   from qdrant_client import QdrantClient
   from qdrant_client.models import (
       Distance,
       FieldSchema,
       PointStruct,
       VectorParams,
   )

   COLLECTION_NAME = "technical_chunks"
   VECTOR_SIZE = 1536
   CHUNKS_PATH = Path("data/chunks_with_embeddings.jsonl")
   GRAPH_PATH = Path("data/technical_graph.json")
   REPORT_PATH = Path("reports/qdrant_load_report.json")


   def value_from(record: dict[str, Any], *keys: str, default: Any = None) -> Any:
       for key in keys:
           value = record.get(key)
           if value is not None:
               return value
       return default


   def load_graph_entities() -> dict[str, dict[str, Any]]:
       if not GRAPH_PATH.exists():
           return {}

       graph = json.loads(GRAPH_PATH.read_text(encoding="utf-8"))
       nodes = graph.get("nodes") or graph.get("entities") or []
       entities: dict[str, dict[str, Any]] = {}

       for node in nodes:
           entity_id = str(value_from(node, "id", "entity_id", "name", default=""))
           if entity_id:
               entities[entity_id.lower()] = node

       return entities


   def normalize_entity_refs(record: dict[str, Any], graph_entities: dict[str, dict[str, Any]]) -> list[dict[str, str]]:
       existing = value_from(record, "entity_refs", "entities", default=[])
       refs: list[dict[str, str]] = []

       if isinstance(existing, list):
           for entity in existing:
               if isinstance(entity, str):
                   refs.append({"id": entity, "type": "unknown"})
               elif isinstance(entity, dict):
                   refs.append(
                       {
                           "id": str(value_from(entity, "id", "entity_id", "name", default="unknown")),
                           "type": str(value_from(entity, "type", "entity_type", "label", default="unknown")),
                       }
                   )

       text = str(value_from(record, "text", "content", "chunk_text", default="")).lower()
       for entity_id, entity in graph_entities.items():
           entity_name = str(value_from(entity, "name", "id", "entity_id", default="")).lower()
           if entity_name and entity_name in text:
               candidate = {
                   "id": str(value_from(entity, "id", "entity_id", "name", default=entity_name)),
                   "type": str(value_from(entity, "type", "entity_type", "label", default="unknown")),
               }
               if candidate not in refs:
                   refs.append(candidate)

       return refs


   def normalize_payload(record: dict[str, Any], entity_refs: list[dict[str, str]]) -> dict[str, Any]:
       metadata = value_from(record, "metadata", default={})
       metadata = metadata if isinstance(metadata, dict) else {}

       document_id = str(value_from(record, "document_id", default=metadata.get("document_id", "unknown")))
       chunk_id = str(value_from(record, "chunk_id", "id", default=metadata.get("chunk_id", "unknown")))
       section_title = str(value_from(record, "section_title", default=metadata.get("section_title", "")))
       text = str(value_from(record, "text", "content", "chunk_text", default=""))

       entity_types = sorted(
           {
               ref["type"]
               for ref in entity_refs
               if ref.get("type") and ref["type"] != "unknown"
           }
       )

       return {
           "document_id": document_id,
           "chunk_id": chunk_id,
           "section_title": section_title,
           "text": text,
           "metadata": metadata,
           "entity_refs": entity_refs,
           "entity_types": entity_types,
       }


   def main() -> None:
       load_dotenv()

       qdrant_url = os.getenv("QDRANT_URL", "http://127.0.0.1:6333")
       client = QdrantClient(url=qdrant_url)
       graph_entities = load_graph_entities()

       if not CHUNKS_PATH.exists():
           raise FileNotFoundError(f"No existe el archivo requerido: {CHUNKS_PATH}")

       client.recreate_collection(
           collection_name=COLLECTION_NAME,
           vectors_config=VectorParams(size=VECTOR_SIZE, distance=Distance.COSINE),
       )

       for field_name in ("document_id", "chunk_id", "entity_types"):
           client.create_payload_index(
               collection_name=COLLECTION_NAME,
               field_name=field_name,
               field_schema=FieldSchema.KEYWORD,
           )

       points: list[PointStruct] = []
       invalid_records: list[dict[str, Any]] = []

       for line_number, line in enumerate(CHUNKS_PATH.read_text(encoding="utf-8").splitlines(), start=1):
           if not line.strip():
               continue

           record = json.loads(line)
           embedding = value_from(record, "embedding", "embeddings")
           chunk_id = str(value_from(record, "chunk_id", "id", default=f"line-{line_number}"))

           if not isinstance(embedding, list) or len(embedding) != VECTOR_SIZE:
               invalid_records.append(
                   {
                       "line": line_number,
                       "chunk_id": chunk_id,
                       "embedding_dimensions": len(embedding) if isinstance(embedding, list) else None,
                   }
               )
               continue

           entity_refs = normalize_entity_refs(record, graph_entities)
           payload = normalize_payload(record, entity_refs)
           point_id = str(uuid.uuid5(uuid.NAMESPACE_URL, f"technical_chunks:{chunk_id}"))

           points.append(
               PointStruct(
                   id=point_id,
                   vector=embedding,
                   payload=payload,
               )
           )

       if invalid_records:
           raise ValueError(
               "Se encontraron embeddings inválidos. "
               f"Ejemplos: {invalid_records[:3]}"
           )

       batch_size = 100
       for start in range(0, len(points), batch_size):
           client.upsert(
               collection_name=COLLECTION_NAME,
               points=points[start : start + batch_size],
               wait=True,
           )

       collection = client.get_collection(COLLECTION_NAME)
       REPORT_PATH.parent.mkdir(parents=True, exist_ok=True)

       report = {
           "collection": COLLECTION_NAME,
           "qdrant_url": qdrant_url,
           "vector_size": VECTOR_SIZE,
           "distance": "Cosine",
           "points_loaded": len(points),
           "points_count": collection.points_count,
           "graph_entities_available": len(graph_entities),
       }

       REPORT_PATH.write_text(
           json.dumps(report, indent=2, ensure_ascii=False),
           encoding="utf-8",
       )

       print(json.dumps(report, indent=2, ensure_ascii=False))


   if __name__ == "__main__":
       main()
   PY
   ```

2. Ejecute el cargador.

   ```bash
   python -m src.scripts.load_qdrant
   ```

3. Consulte el informe generado.

   ```bash
   cat reports/qdrant_load_report.json | python -m json.tool
   ```

4. Consulte la colección mediante la API REST.

   ```bash
   curl -s http://127.0.0.1:6333/collections/technical_chunks | python -m json.tool
   ```

**Salida esperada:**

El cargador debe informar una estructura similar a la siguiente:

```json
{
  "collection": "technical_chunks",
  "qdrant_url": "http://127.0.0.1:6333",
  "vector_size": 1536,
  "distance": "Cosine",
  "points_loaded": 42,
  "points_count": 42,
  "graph_entities_available": 12
}
```

El número exacto de puntos depende del corpus creado en los laboratorios anteriores.

**Verificación:**

Compruebe que el número de puntos cargados es igual al número de líneas JSONL válidas:

```bash
python - <<'PY'
import json
from pathlib import Path

report = json.loads(Path("reports/qdrant_load_report.json").read_text(encoding="utf-8"))
lines = [
    line for line in Path("data/chunks_with_embeddings.jsonl")
    .read_text(encoding="utf-8")
    .splitlines()
    if line.strip()
]

assert report["points_loaded"] == len(lines), (
    f"Qdrant cargó {report['points_loaded']} puntos y el JSONL contiene {len(lines)} líneas."
)
print("La colección contiene todos los chunks esperados.")
PY
```

---

### Paso 4. Implementar el recuperador vectorial persistente

**Objetivo:** implementar búsquedas semánticas top-k sobre Qdrant con filtros por `document_id` y por tipos de entidad, devolviendo contexto textual citable.

**Instrucciones:**

1. Cree el archivo `src/retrieval/persistent_vector_retriever.py`.

   ```bash
   cat > src/retrieval/persistent_vector_retriever.py <<'PY'
   import os
   from dataclasses import dataclass
   from typing import Any

   from dotenv import load_dotenv
   from openai import AzureOpenAI
   from qdrant_client import QdrantClient
   from qdrant_client.models import FieldCondition, Filter, MatchAny, MatchValue


   @dataclass
   class RetrievedChunk:
       chunk_id: str
       document_id: str
       section_title: str
       text: str
       score: float
       entity_refs: list[dict[str, str]]

       @property
       def citation(self) -> str:
           section = f", sección: {self.section_title}" if self.section_title else ""
           return f"[document_id: {self.document_id}, chunk_id: {self.chunk_id}{section}]"

       def as_context(self) -> str:
           return f"{self.citation}\n{self.text}"


   class PersistentVectorRetriever:
       def __init__(
           self,
           collection_name: str = "technical_chunks",
           qdrant_url: str | None = None,
       ) -> None:
           load_dotenv()

           self.collection_name = collection_name
           self.qdrant_url = qdrant_url or os.getenv(
               "QDRANT_URL",
               "http://127.0.0.1:6333",
           )

           self.client = QdrantClient(url=self.qdrant_url)
           self.embedding_client = AzureOpenAI(
               azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
               api_key=os.environ["AZURE_OPENAI_API_KEY"],
               api_version=os.getenv(
                   "AZURE_OPENAI_API_VERSION",
                   "2024-02-15-preview",
               ),
           )
           self.embedding_deployment = os.environ[
               "AZURE_OPENAI_EMBEDDING_DEPLOYMENT"
           ]

       def embed_query(self, query: str) -> list[float]:
           response = self.embedding_client.embeddings.create(
               model=self.embedding_deployment,
               input=query,
           )
           embedding = response.data[0].embedding

           if len(embedding) != 1536:
               raise ValueError(
                   f"Se esperaban embeddings de 1536 dimensiones; se recibieron {len(embedding)}."
               )

           return embedding

       @staticmethod
       def build_filter(
           document_id: str | None = None,
           entity_type: str | None = None,
       ) -> Filter | None:
           conditions = []

           if document_id:
               conditions.append(
                   FieldCondition(
                       key="document_id",
                       match=MatchValue(value=document_id),
                   )
               )

           if entity_type:
               conditions.append(
                   FieldCondition(
                       key="entity_types",
                       match=MatchAny(any=[entity_type]),
                   )
               )

           return Filter(must=conditions) if conditions else None

       def retrieve(
           self,
           query: str,
           top_k: int = 4,
           document_id: str | None = None,
           entity_type: str | None = None,
       ) -> list[RetrievedChunk]:
           query_vector = self.embed_query(query)
           query_filter = self.build_filter(
               document_id=document_id,
               entity_type=entity_type,
           )

           results = self.client.search(
               collection_name=self.collection_name,
               query_vector=query_vector,
               query_filter=query_filter,
               limit=top_k,
               with_payload=True,
           )

           chunks = []
           for result in results:
               payload: dict[str, Any] = result.payload or {}
               chunks.append(
                   RetrievedChunk(
                       chunk_id=str(payload.get("chunk_id", result.id)),
                       document_id=str(payload.get("document_id", "unknown")),
                       section_title=str(payload.get("section_title", "")),
                       text=str(payload.get("text", "")),
                       score=float(result.score),
                       entity_refs=payload.get("entity_refs", []),
                   )
               )

           return chunks

       def retrieve_citable_context(
           self,
           query: str,
           top_k: int = 4,
           document_id: str | None = None,
           entity_type: str | None = None,
       ) -> str:
           chunks = self.retrieve(
               query=query,
               top_k=top_k,
               document_id=document_id,
               entity_type=entity_type,
           )

           if not chunks:
               return "No se recuperó evidencia textual con los filtros solicitados."

           return "\n\n---\n\n".join(chunk.as_context() for chunk in chunks)
   PY
   ```

2. Ejecute una búsqueda semántica sin filtros.

   ```bash
   python - <<'PY'
   from src.retrieval.persistent_vector_retriever import PersistentVectorRetriever

   retriever = PersistentVectorRetriever()
   results = retriever.retrieve(
       query="¿Qué sistema procesa facturas y qué área lo administra?",
       top_k=3,
   )

   for index, result in enumerate(results, start=1):
       print(f"\nResultado {index}")
       print("Puntuación:", round(result.score, 4))
       print("Cita:", result.citation)
       print("Entidades:", result.entity_refs)
       print("Texto:", result.text[:300])
   PY
   ```

3. Genere contexto citable para una respuesta posterior de un modelo.

   ```bash
   python - <<'PY'
   from src.retrieval.persistent_vector_retriever import PersistentVectorRetriever

   retriever = PersistentVectorRetriever()
   context = retriever.retrieve_citable_context(
       query="¿Qué política aplica a las facturas?",
       top_k=3,
   )
   print(context)
   PY
   ```

4. Identifique un `document_id` válido a partir de los resultados y pruebe un filtro documental.

   ```bash
   python - <<'PY'
   from src.retrieval.persistent_vector_retriever import PersistentVectorRetriever

   retriever = PersistentVectorRetriever()
   initial = retriever.retrieve("retención de facturas", top_k=1)

   if initial:
       document_id = initial[0].document_id
       print("Probando document_id:", document_id)
       filtered = retriever.retrieve(
           query="retención de facturas",
           top_k=3,
           document_id=document_id,
       )

       for item in filtered:
           assert item.document_id == document_id
           print(item.citation, round(item.score, 4))
   else:
       print("No se recuperaron resultados; revise la colección y las credenciales.")
   PY
   ```

**Salida esperada:**

Cada resultado debe incluir:

- Una puntuación de similitud Cosine.
- El `chunk_id`.
- El `document_id`.
- El título de sección, si existe.
- El texto del chunk.
- Las referencias de entidades preservadas en el payload.
- Una cita trazable.

Ejemplo de cita:

```text
[document_id: Politica_RET-01.pdf, chunk_id: Politica_RET-01_003, sección: Retención]
```

**Verificación:**

Ejecute una consulta y confirme que todos los resultados contienen una cita no vacía:

```bash
python - <<'PY'
from src.retrieval.persistent_vector_retriever import PersistentVectorRetriever

retriever = PersistentVectorRetriever()
results = retriever.retrieve("sistemas gestionados por Finanzas", top_k=3)

assert results, "No se recuperaron resultados."
assert all(item.chunk_id and item.document_id for item in results)
assert all(item.citation.startswith("[document_id:") for item in results)

print(f"Recuperación persistente correcta: {len(results)} chunks citables.")
PY
```

---

### Paso 5. Aplicar filtros por tipo de entidad y validar payloads

**Objetivo:** demostrar que Qdrant permite restringir la evidencia recuperada mediante metadatos estructurados, sin alterar la representación lógica del grafo.

**Instrucciones:**

1. Recupere algunos puntos para identificar los tipos de entidad disponibles.

   ```bash
   python - <<'PY'
   from qdrant_client import QdrantClient

   client = QdrantClient(url="http://127.0.0.1:6333")
   points, _ = client.scroll(
       collection_name="technical_chunks",
       limit=100,
       with_payload=True,
       with_vectors=False,
   )

   entity_types = sorted(
       {
           entity_type
           for point in points
           for entity_type in point.payload.get("entity_types", [])
       }
   )

   print("Tipos de entidad disponibles:")
   for entity_type in entity_types:
       print("-", entity_type)
   PY
   ```

2. Seleccione uno de los tipos mostrados. Por ejemplo, si el corpus contiene entidades con tipo `Sistema`, ejecute:

   ```bash
   python - <<'PY'
   from src.retrieval.persistent_vector_retriever import PersistentVectorRetriever

   retriever = PersistentVectorRetriever()
   results = retriever.retrieve(
       query="activos y sistemas relacionados con facturas",
       top_k=5,
       entity_type="Sistema",
   )

   for result in results:
       print(result.citation)
       print("Tipos asociados:", sorted({e.get("type", "unknown") for e in result.entity_refs}))
       print()
   PY
   ```

3. Si su grafo utiliza otro valor, por ejemplo `POLICY`, `SYSTEM`, `AREA` o `PRODUCT`, reemplace `"Sistema"` por el tipo real mostrado en el primer comando.

4. Consulte los índices de payload creados para la colección.

   ```bash
   curl -s http://127.0.0.1:6333/collections/technical_chunks | python -m json.tool
   ```

**Salida esperada:**

El filtro por tipo de entidad debe devolver exclusivamente chunks cuyo payload tenga el valor solicitado dentro de `entity_types`.

**Verificación:**

Sustituya `Sistema` por un tipo existente y ejecute:

```bash
python - <<'PY'
from src.retrieval.persistent_vector_retriever import PersistentVectorRetriever

ENTITY_TYPE = "Sistema"

retriever = PersistentVectorRetriever()
results = retriever.retrieve(
    query="información técnica relacionada",
    top_k=10,
    entity_type=ENTITY_TYPE,
)

assert results, f"No se recuperaron resultados para entity_type={ENTITY_TYPE}."

for result in results:
    types = {ref.get("type") for ref in result.entity_refs}
    assert ENTITY_TYPE in types, (
        f"El chunk {result.chunk_id} no contiene el tipo esperado. Tipos: {types}"
    )

print("Filtro por tipo de entidad validado.")
PY
```

---

### Paso 6. Integrar Qdrant en `GraphKnowledgeRetrievalSkill`

**Objetivo:** sustituir la recuperación textual desde JSONL por Qdrant, manteniendo `technical_graph.json` como representación de relaciones para expansión controlada del grafo.

**Instrucciones:**

1. Abra la implementación existente de `GraphKnowledgeRetrievalSkill` creada en el laboratorio `03-00-02`.

   ```bash
   sed -n '1,240p' src/skills/graph_knowledge_retrieval_skill.py
   ```

2. Sustituya la dependencia del recuperador JSONL por `PersistentVectorRetriever`. Si el archivo no existe, cree una implementación mínima compatible con esta práctica.

   ```bash
   cat > src/skills/graph_knowledge_retrieval_skill.py <<'PY'
   import json
   from pathlib import Path
   from typing import Any

   from src.retrieval.persistent_vector_retriever import PersistentVectorRetriever


   class GraphKnowledgeRetrievalSkill:
       """
       Recupera evidencia textual desde Qdrant y relaciones explícitas desde
       technical_graph.json. El grafo continúa siendo la capa lógica de relaciones.
       """

       def __init__(
           self,
           graph_path: str = "data/technical_graph.json",
           retriever: PersistentVectorRetriever | None = None,
       ) -> None:
           self.graph_path = Path(graph_path)
           self.retriever = retriever or PersistentVectorRetriever()
           self.graph = json.loads(self.graph_path.read_text(encoding="utf-8"))

       def _relationships(self) -> list[dict[str, Any]]:
           if isinstance(self.graph, dict):
               return (
                   self.graph.get("relationships")
                   or self.graph.get("relations")
                   or self.graph.get("edges")
                   or []
               )
           return []

       def find_related_evidence(
           self,
           entity_name: str,
           max_hops: int = 1,
       ) -> list[dict[str, Any]]:
           """
           Expansión controlada a una profundidad máxima. Adapte los nombres
           source/target si el artefacto del laboratorio 03-00-02 usa otros campos.
           """
           if max_hops < 1:
               return []

           normalized = entity_name.lower()
           evidence = []

           for relation in self._relationships():
               source = str(
                   relation.get("source")
                   or relation.get("from")
                   or relation.get("origin")
                   or ""
               )
               target = str(
                   relation.get("target")
                   or relation.get("to")
                   or relation.get("destination")
                   or ""
               )

               if normalized in {source.lower(), target.lower()}:
                   evidence.append(
                       {
                           "source": source,
                           "relation": relation.get("relation")
                           or relation.get("type")
                           or relation.get("label")
                           or "related_to",
                           "target": target,
                           "source_document": relation.get("source_document")
                           or relation.get("source")
                           or "technical_graph.json",
                       }
                   )

           return evidence

       def retrieve(
           self,
           question: str,
           top_k: int = 4,
           entity_name: str | None = None,
           document_id: str | None = None,
           entity_type: str | None = None,
       ) -> dict[str, Any]:
           textual_chunks = self.retriever.retrieve(
               query=question,
               top_k=top_k,
               document_id=document_id,
               entity_type=entity_type,
           )

           graph_evidence = (
               self.find_related_evidence(entity_name, max_hops=1)
               if entity_name
               else []
           )

           textual_context = "\n\n---\n\n".join(
               chunk.as_context() for chunk in textual_chunks
           )

           graph_context = "\n".join(
               (
                   f"- {item['source']} —{item['relation']}→ {item['target']} "
                   f"(fuente: {item['source_document']})"
               )
               for item in graph_evidence
           )

           return {
               "question": question,
               "textual_evidence": [
                   {
                       "chunk_id": chunk.chunk_id,
                       "document_id": chunk.document_id,
                       "section_title": chunk.section_title,
                       "score": chunk.score,
                       "citation": chunk.citation,
                       "text": chunk.text,
                       "entity_refs": chunk.entity_refs,
                   }
                   for chunk in textual_chunks
               ],
               "graph_evidence": graph_evidence,
               "citable_context": (
                   "EVIDENCIA TEXTUAL\n"
                   f"{textual_context or 'Sin evidencia textual recuperada.'}\n\n"
                   "EVIDENCIA DE GRAFO\n"
                   f"{graph_context or 'Sin relaciones recuperadas.'}"
               ),
           }
   PY
   ```

3. Ejecute una recuperación híbrida. Sustituya `Atlas` por una entidad existente en su corpus si fuera necesario.

   ```bash
   python - <<'PY'
   import json
   from src.skills.graph_knowledge_retrieval_skill import GraphKnowledgeRetrievalSkill

   skill = GraphKnowledgeRetrievalSkill()

   result = skill.retrieve(
       question="¿Qué área administra el sistema que procesa facturas?",
       top_k=3,
       entity_name="Atlas",
   )

   print(json.dumps(result, indent=2, ensure_ascii=False))
   PY
   ```

4. Confirme que la evidencia textual procede de Qdrant y que la evidencia relacional procede del archivo de grafo.

**Salida esperada:**

La salida debe contener dos grupos de evidencia:

```text
EVIDENCIA TEXTUAL
[document_id: ..., chunk_id: ...]
...

EVIDENCIA DE GRAFO
- Atlas —procesa→ Facturas (...)
- Atlas —gestionado_por→ Finanzas (...)
```

La evidencia textual debe tener puntuaciones vectoriales y citas. La evidencia del grafo debe mantener relaciones explícitas y trazables.

**Verificación:**

Ejecute:

```bash
python - <<'PY'
from src.skills.graph_knowledge_retrieval_skill import GraphKnowledgeRetrievalSkill

skill = GraphKnowledgeRetrievalSkill()
result = skill.retrieve(
    question="¿Qué sistemas están relacionados con la retención de facturas?",
    top_k=3,
)

assert "textual_evidence" in result
assert "graph_evidence" in result
assert "citable_context" in result
assert result["textual_evidence"], "No se obtuvo evidencia textual desde Qdrant."

for evidence in result["textual_evidence"]:
    assert evidence["citation"]
    assert evidence["chunk_id"]
    assert evidence["document_id"]

print("GraphKnowledgeRetrievalSkill usa evidencia textual citable desde Qdrant.")
PY
```

---

### Paso 7. Ejecutar regresión contra el recuperador JSONL baseline

**Objetivo:** medir consistencia funcional, fragmentos recuperados, latencia y puntuación del harness al comparar Qdrant con la recuperación local basada en JSONL.

**Instrucciones:**

1. Cree el conjunto fijo de consultas de regresión en `config/regression_queries.json`.

   ```bash
   cat > config/regression_queries.json <<'EOF'
   [
     {
       "id": "retention-policy",
       "query": "¿Qué política aplica a la retención de facturas?",
       "top_k": 3
     },
     {
       "id": "system-owner",
       "query": "¿Qué área administra el sistema que procesa facturas?",
       "top_k": 3
     },
     {
       "id": "technical-relations",
       "query": "¿Qué relaciones existen entre sistemas, políticas y documentos técnicos?",
       "top_k": 3
     }
   ]
   EOF
   ```

2. Cree el script de regresión `src/scripts/run_retrieval_regression.py`.

   ```bash
   cat > src/scripts/run_retrieval_regression.py <<'PY'
   import json
   import math
   import time
   from pathlib import Path
   from typing import Any

   from src.retrieval.persistent_vector_retriever import PersistentVectorRetriever

   CHUNKS_PATH = Path("data/chunks_with_embeddings.jsonl")
   QUERIES_PATH = Path("config/regression_queries.json")
   REPORT_PATH = Path("reports/retrieval_regression.json")


   def get_value(record: dict[str, Any], *keys: str, default: Any = None) -> Any:
       for key in keys:
           if record.get(key) is not None:
               return record[key]
       return default


   def cosine_similarity(a: list[float], b: list[float]) -> float:
       dot = sum(x * y for x, y in zip(a, b))
       norm_a = math.sqrt(sum(x * x for x in a))
       norm_b = math.sqrt(sum(y * y for y in b))

       if norm_a == 0 or norm_b == 0:
           return 0.0

       return dot / (norm_a * norm_b)


   def load_baseline() -> list[dict[str, Any]]:
       records = []
       for line in CHUNKS_PATH.read_text(encoding="utf-8").splitlines():
           if not line.strip():
               continue

           item = json.loads(line)
           embedding = get_value(item, "embedding", "embeddings")
           records.append(
               {
                   "chunk_id": str(get_value(item, "chunk_id", "id", default="unknown")),
                   "document_id": str(get_value(item, "document_id", default="unknown")),
                   "embedding": embedding,
               }
           )
       return records


   def baseline_search(
       records: list[dict[str, Any]],
       query_embedding: list[float],
       top_k: int,
   ) -> list[dict[str, Any]]:
       scored = [
           {
               **record,
               "score": cosine_similarity(query_embedding, record["embedding"]),
           }
           for record in records
       ]
       return sorted(scored, key=lambda item: item["score"], reverse=True)[:top_k]


   def main() -> None:
       retriever = PersistentVectorRetriever()
       baseline_records = load_baseline()
       queries = json.loads(QUERIES_PATH.read_text(encoding="utf-8"))

       results = []

       for item in queries:
           query = item["query"]
           top_k = item.get("top_k", 3)

           query_embedding = retriever.embed_query(query)

           start_baseline = time.perf_counter()
           baseline = baseline_search(baseline_records, query_embedding, top_k)
           baseline_latency_ms = (time.perf_counter() - start_baseline) * 1000

           start_qdrant = time.perf_counter()
           qdrant = retriever.client.search(
               collection_name=retriever.collection_name,
               query_vector=query_embedding,
               limit=top_k,
               with_payload=True,
           )
           qdrant_latency_ms = (time.perf_counter() - start_qdrant) * 1000

           baseline_ids = [item["chunk_id"] for item in baseline]
           qdrant_ids = [
               str(point.payload.get("chunk_id", point.id))
               for point in qdrant
           ]

           overlap = len(set(baseline_ids) & set(qdrant_ids))
           overlap_at_k = overlap / top_k if top_k else 0.0

           # El harness considera consistente una recuperación con al menos
           # 80 % de coincidencia de chunks entre baseline y Qdrant.
           harness_score = 1.0 if overlap_at_k >= 0.80 else overlap_at_k

           results.append(
               {
                   "id": item["id"],
                   "query": query,
                   "top_k": top_k,
                   "baseline_latency_ms": round(baseline_latency_ms, 3),
                   "qdrant_latency_ms": round(qdrant_latency_ms, 3),
                   "baseline_chunk_ids": baseline_ids,
                   "qdrant_chunk_ids": qdrant_ids,
                   "overlap_at_k": round(overlap_at_k, 3),
                   "harness_score": round(harness_score, 3),
               }
           )

       average_harness_score = sum(
           item["harness_score"] for item in results
       ) / len(results)

       report = {
           "collection": retriever.collection_name,
           "baseline": "JSONL con similitud Cosine en memoria",
           "persistent_retriever": "Qdrant 1.9.1 con distancia Cosine",
           "queries": results,
           "average_harness_score": round(average_harness_score, 3),
       }

       REPORT_PATH.parent.mkdir(parents=True, exist_ok=True)
       REPORT_PATH.write_text(
           json.dumps(report, indent=2, ensure_ascii=False),
           encoding="utf-8",
       )

       print(json.dumps(report, indent=2, ensure_ascii=False))


   if __name__ == "__main__":
       main()
   PY
   ```

3. Ejecute la regresión.

   ```bash
   python -m src.scripts.run_retrieval_regression
   ```

4. Revise el informe persistido.

   ```bash
   cat reports/retrieval_regression.json | python -m json.tool
   ```

5. Interprete los resultados:

   - `baseline_chunk_ids`: chunks recuperados desde JSONL en memoria.
   - `qdrant_chunk_ids`: chunks recuperados desde Qdrant.
   - `overlap_at_k`: porcentaje de IDs comunes entre ambos recuperadores.
   - `baseline_latency_ms`: latencia de cálculo local, excluyendo la generación del embedding.
   - `qdrant_latency_ms`: latencia de búsqueda vectorial persistente, excluyendo la generación del embedding.
   - `harness_score`: puntuación funcional de consistencia.

**Salida esperada:**

El informe debe contener una entrada por cada consulta. En un corpus pequeño y con la misma métrica Cosine, es razonable esperar una coincidencia alta, normalmente `0.8` o superior.

Ejemplo resumido:

```json
{
  "id": "retention-policy",
  "top_k": 3,
  "baseline_latency_ms": 1.242,
  "qdrant_latency_ms": 8.915,
  "overlap_at_k": 1.0,
  "harness_score": 1.0
}
```

La latencia de Qdrant puede ser superior en un corpus pequeño debido a la comunicación HTTP local. Su valor principal en este laboratorio es la persistencia, filtrado, escalabilidad y administración de payloads.

**Verificación:**

Ejecute la validación automatizada del informe:

```bash
python - <<'PY'
import json
from pathlib import Path

report = json.loads(Path("reports/retrieval_regression.json").read_text(encoding="utf-8"))

assert report["queries"], "No existen resultados de regresión."
assert report["average_harness_score"] >= 0.80, (
    f"Puntuación insuficiente: {report['average_harness_score']}. "
    "Revise dimensión, métrica, colección y carga de puntos."
)

for query in report["queries"]:
    assert query["baseline_chunk_ids"], f"Baseline sin resultados: {query['id']}"
    assert query["qdrant_chunk_ids"], f"Qdrant sin resultados: {query['id']}"
    assert query["qdrant_latency_ms"] >= 0

print("Regresión aprobada.")
print("Puntuación media del harness:", report["average_harness_score"])
PY
```

## Validación y pruebas

Ejecute esta secuencia final desde `~/genai-agent-labs` con el entorno virtual activo:

```bash
source .venv/bin/activate

docker compose -f docker-compose.qdrant.yml ps

curl -s http://127.0.0.1:6333/collections/technical_chunks | python -m json.tool

python -m src.scripts.load_qdrant

python -m src.scripts.run_retrieval_regression
```

La práctica se considera completada si se cumplen todos los criterios siguientes:

| Criterio | Evidencia esperada |
|---|---|
| Qdrant está operativo | `docker compose ... ps` indica servicio en ejecución |
| Colección correcta | Existe `technical_chunks` con vectores de tamaño 1536 y distancia Cosine |
| Persistencia habilitada | Existe contenido en `data/qdrant_storage/` |
| Carga correcta | `points_loaded` coincide con los chunks válidos del JSONL |
| Payload preservado | Los resultados contienen `document_id`, `chunk_id`, `section_title`, `entity_refs` y `entity_types` |
| Búsqueda semántica | `PersistentVectorRetriever.retrieve()` devuelve resultados ordenados por puntuación |
| Filtros funcionales | Los filtros por documento y tipo de entidad restringen la evidencia recuperada |
| Integración GraphRAG | `GraphKnowledgeRetrievalSkill` combina evidencia textual de Qdrant y relaciones de `technical_graph.json` |
| Regresión aprobada | `average_harness_score` es igual o superior a `0.80` |
| Seguridad | `.env` y `data/qdrant_storage/` no se incluyen en Git |

Compruebe que no agregará secretos ni datos persistentes al commit:

```bash
git status --short
```

La salida no debe incluir:

```text
.env
data/qdrant_storage/
reports/qdrant_load_report.json
reports/retrieval_regression.json
```

Realice el commit correspondiente a esta práctica según la convención global indicada por el instructor:

```bash
git add docker-compose.qdrant.yml \
  .gitignore \
  config/regression_queries.json \
  src/retrieval/persistent_vector_retriever.py \
  src/scripts/load_qdrant.py \
  src/scripts/run_retrieval_regression.py \
  src/skills/graph_knowledge_retrieval_skill.py

git commit -m "lab-02-00-03"
```

> Si su instructor ha definido un identificador de commit específico para el bloque 03, use la convención indicada por él. No modifique commits previos.

## Solución de problemas

### Problema 1. Qdrant no inicia o el puerto 6333 no responde

**Síntomas:**

```bash
curl -s http://127.0.0.1:6333/collections
```

No devuelve respuesta, devuelve código `000`, o Docker muestra un contenedor detenido.

**Causa probable:**

Docker Engine no está iniciado, el puerto `6333` está ocupado, o existe un contenedor previo con una configuración incompatible.

**Solución:**

1. Revise el estado y los logs:

   ```bash
   docker compose -f docker-compose.qdrant.yml ps
   docker compose -f docker-compose.qdrant.yml logs qdrant
   ```

2. Identifique el proceso que ocupa el puerto:

   ```bash
   sudo lsof -i :6333
   ```

3. Detenga la instancia anterior de Qdrant y reinicie el servicio del laboratorio:

   ```bash
   docker compose -f docker-compose.qdrant.yml down
   docker compose -f docker-compose.qdrant.yml up -d
   ```

4. Valide nuevamente:

   ```bash
   curl -s http://127.0.0.1:6333/healthz
   ```

### Problema 2. La carga falla con dimensión de embedding distinta de 1536

**Síntomas:**

El script `load_qdrant.py` muestra un error similar a:

```text
Se encontraron embeddings inválidos
```

o el recuperador indica:

```text
Se esperaban embeddings de 1536 dimensiones
```

**Causa probable:**

El archivo `chunks_with_embeddings.jsonl` fue generado con un modelo de embeddings diferente, con un parámetro de dimensión distinto, o la variable `AZURE_OPENAI_EMBEDDING_DEPLOYMENT` apunta a un despliegue que no corresponde a `text-embedding-3-small`.

**Solución:**

1. Compruebe la dimensión de los embeddings almacenados:

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   item = json.loads(
       Path("data/chunks_with_embeddings.jsonl")
       .read_text(encoding="utf-8")
       .splitlines()[0]
   )
   embedding = item.get("embedding") or item.get("embeddings")
   print("Dimensión:", len(embedding))
   PY
   ```

2. Verifique la variable de entorno:

   ```bash
   grep '^AZURE_OPENAI_EMBEDDING_DEPLOYMENT=' .env
   ```

3. Regrese al laboratorio `03-00-01`, regenere los embeddings usando el despliegue correcto de `text-embedding-3-small` y confirme que todos tienen dimensión 1536.

4. Elimine y reconstruya la colección:

   ```bash
   python -m src.scripts.load_qdrant
   ```

## Limpieza

Al finalizar la sesión, detenga Qdrant para liberar memoria y puertos, conservando los datos persistentes para la siguiente práctica:

```bash
docker compose -f docker-compose.qdrant.yml down
```

Compruebe que los datos persistentes siguen disponibles:

```bash
ls -lah data/qdrant_storage
```

Si necesita reiniciar completamente el laboratorio y eliminar la colección, detenga el servicio y borre el almacenamiento local:

```bash
docker compose -f docker-compose.qdrant.yml down
rm -rf data/qdrant_storage
```

> **Advertencia:** el último comando elimina los índices y payloads persistidos. Será necesario volver a ejecutar `python -m src.scripts.load_qdrant` antes de realizar nuevas consultas.

## Resumen

En esta práctica desplegó Qdrant como capa de persistencia vectorial para el corpus técnico. La colección `technical_chunks` almacena embeddings de 1536 dimensiones con distancia Cosine, junto con payloads trazables que incluyen documentos, secciones, chunks y referencias de entidades.

También implementó un recuperador persistente capaz de realizar búsquedas top-k y filtros por metadatos. Finalmente, integró Qdrant en `GraphKnowledgeRetrievalSkill`, manteniendo `technical_graph.json` como fuente lógica de relaciones explícitas. Esta separación permite que la recuperación híbrida combine evidencia textual semántica persistente con expansión controlada de relaciones de grafo.

### Recursos opcionales

- [Documentación de Qdrant 1.9](https://qdrant.tech/documentation/)
- [Documentación del cliente Python de Qdrant](https://python-client.qdrant.tech/)
- [Documentación de embeddings de Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/concepts/understand-embeddings)
- [Documentación de GraphRAG de Microsoft](https://microsoft.github.io/graphrag/)

---

# 2. Práctica 4. Implementar una solución GraphRAG utilizando FalkorDB y comparar su arquitectura, capacidades y casos de uso con plataformas como Weaviate y Neo4j para seleccionar la alternativa más adecuada según distintos escenarios.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 35 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Visión general

En esta práctica construirás un flujo GraphRAG híbrido que combina la recuperación semántica de fragmentos técnicos almacenados en Qdrant con la expansión controlada de relaciones persistidas en FalkorDB. El flujo recuperará los cinco fragmentos más relevantes para una pregunta, extraerá sus entidades asociadas, recorrerá el grafo hasta dos saltos y generará una respuesta fundamentada con citas documentales y caminos relacionales.

También desplegarás perfiles Docker Compose independientes para FalkorDB, Neo4j y Weaviate. Finalmente, compararás las tres plataformas mediante una matriz de decisión centrada en arquitectura, consultas, búsqueda vectorial, operación, escalabilidad y escenarios de uso.

## Objetivos de aprendizaje

- [ ] Persistir el archivo `technical_graph.json` en FalkorDB bajo el grafo `technical_knowledge_graph`.
- [ ] Crear índices de entidades por identificador y nombre, y ejecutar consultas Cypher parametrizadas.
- [ ] Implementar un recuperador híbrido que combine Qdrant, FalkorDB y Azure OpenAI.
- [ ] Construir respuestas GraphRAG con evidencia documental, relaciones recuperadas y límites explícitos de expansión.
- [ ] Comparar FalkorDB, Neo4j y Weaviate para seleccionar la plataforma adecuada según distintos escenarios técnicos.

## Prerrequisitos

### Conocimientos requeridos

Antes de comenzar, debes conocer:

- Recuperación semántica basada en embeddings y búsqueda vectorial.
- Estructura del corpus técnico y de la colección `technical_chunks` creada en el laboratorio 03-00-03.
- Estructura del archivo `data/technical_graph.json` generado en el laboratorio 03-00-02.
- Operaciones básicas de Docker Compose.
- Conceptos básicos de Cypher: `MATCH`, `MERGE`, `CREATE INDEX`, relaciones y parámetros.
- Uso de variables de entorno para credenciales de Azure OpenAI.

### Acceso y recursos requeridos

Debes disponer de:

- Repositorio local en `~/genai-agent-labs`.
- Entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Qdrant disponible y con la colección `technical_chunks` cargada.
- Docker Engine y Docker Compose funcionales.
- Al menos 16 GB de RAM disponibles para Docker Compose.
- Acceso a Azure OpenAI con:
  - Despliegue compatible con chat.
  - Despliegue compatible con embeddings.
- Credenciales de Azure OpenAI almacenadas exclusivamente en variables de entorno o en un archivo `.env` no versionado.
- Harness de evaluación creado en el laboratorio 02-00-04.

> **Importante:** FalkorDB, Neo4j y Weaviate se iniciarán mediante perfiles independientes. No es necesario ejecutar las tres plataformas simultáneamente; hacerlo puede consumir memoria innecesariamente.

## Entorno del laboratorio

### Recursos de software

| Componente | Versión objetivo | Uso |
|---|---:|---|
| FalkorDB | 4.6.0 | Persistencia y consulta del grafo técnico |
| Cliente Python FalkorDB | 1.0.10 | Ejecución de consultas Cypher desde Python |
| Qdrant | 1.9.1 | Recuperación vectorial de fragmentos técnicos |
| Neo4j Community Edition | 5.20.0 | Comparación de motor de grafos |
| Weaviate | 1.25.4 | Comparación de base vectorial orientada a objetos |
| Python | 3.12.1 | Implementación del flujo GraphRAG |
| Azure OpenAI SDK | Compatible con el repositorio | Embeddings y generación de respuestas |

### Preparación inicial

1. Abre una terminal y entra al directorio obligatorio del curso.

   ```bash
   cd ~/genai-agent-labs
   ```

2. Activa el entorno virtual compartido.

   ```bash
   source .venv/bin/activate
   ```

3. Verifica que estás en el repositorio correcto.

   ```bash
   git status
   pwd
   ```

4. Instala las dependencias necesarias para esta práctica.

   ```bash
   pip install \
     falkordb==1.0.10 \
     qdrant-client==1.9.1 \
     openai \
     python-dotenv==1.0.1 \
     requests
   ```

5. Crea la estructura de trabajo del laboratorio.

   ```bash
   mkdir -p src/graphrag
   mkdir -p config
   mkdir -p reports
   mkdir -p data
   ```

6. Verifica que los artefactos de los laboratorios previos existen.

   ```bash
   test -f data/technical_graph.json && echo "technical_graph.json disponible"
   find data -maxdepth 2 -type f | sort | sed -n '1,40p'
   ```

7. Comprueba que Qdrant responde. Ajusta el puerto si tu instancia previa utiliza una dirección diferente.

   ```bash
   curl -s http://127.0.0.1:6333/collections/technical_chunks | python -m json.tool | sed -n '1,80p'
   ```

## Procedimiento paso a paso

### Paso 1. Inspeccionar los artefactos de entrada y definir el contrato de datos

**Objetivo:** Confirmar que el grafo técnico y la colección vectorial contienen los campos requeridos para conectar fragmentos, entidades, relaciones y fuentes documentales.

**Instrucciones:**

1. Inspecciona la estructura principal del archivo de grafo.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   path = Path("data/technical_graph.json")
   graph = json.loads(path.read_text(encoding="utf-8"))

   print("Claves principales:", list(graph.keys()))
   for key, value in graph.items():
       if isinstance(value, list):
           print(f"{key}: {len(value)} elementos")
           if value:
               print("Ejemplo:", json.dumps(value[0], ensure_ascii=False, indent=2))
   PY
   ```

2. Identifica los nombres de las colecciones de nodos y relaciones. Esta práctica admite las convenciones más habituales:

   ```json
   {
     "entities": [...],
     "relationships": [...]
   }
   ```

   o bien:

   ```json
   {
     "nodes": [...],
     "edges": [...]
   }
   ```

3. Verifica que cada entidad tenga, como mínimo, un identificador estable y un nombre.

   Ejemplo esperado:

   ```json
   {
     "id": "system-atlas",
     "name": "Atlas",
     "type": "System",
     "source": "Inventario_Sistemas.xlsx, fila Atlas"
   }
   ```

4. Verifica que cada relación tenga origen, destino y tipo.

   Ejemplo esperado:

   ```json
   {
     "source": "system-atlas",
     "target": "area-finanzas",
     "type": "GESTIONADO_POR",
     "source_document": "Inventario_Sistemas.xlsx, fila Atlas"
   }
   ```

5. Inspecciona un punto de Qdrant para comprobar que sus metadatos incluyen referencias a entidades. El recuperador soportará los campos `entity_ids`, `entities` o `entity_id`.

   ```bash
   curl -s \
     -X POST http://127.0.0.1:6333/collections/technical_chunks/points/scroll \
     -H "Content-Type: application/json" \
     -d '{"limit": 1, "with_payload": true, "with_vector": false}' \
     | python -m json.tool
   ```

6. Si los puntos de Qdrant no contienen referencias a entidades, añade el mapeo durante la carga del corpus o conserva una tabla JSON de correspondencia entre `chunk_id` y `entity_ids`. No inventes entidades durante la recuperación: el flujo GraphRAG debe usar evidencia producida por las prácticas anteriores.

**Resultado esperado:**

- El archivo `data/technical_graph.json` contiene nodos y relaciones.
- La colección `technical_chunks` existe en Qdrant.
- Los fragmentos vectoriales pueden asociarse a una o más entidades técnicas.
- Las entidades y relaciones incluyen información de fuente cuando está disponible.

**Verificación:**

Ejecuta:

```bash
python - <<'PY'
import json
graph = json.load(open("data/technical_graph.json", encoding="utf-8"))
entities = graph.get("entities", graph.get("nodes", []))
relations = graph.get("relationships", graph.get("edges", []))
print(f"Entidades o nodos: {len(entities)}")
print(f"Relaciones o aristas: {len(relations)}")
assert entities, "No se encontraron entidades o nodos."
assert relations, "No se encontraron relaciones o aristas."
PY
```

---

### Paso 2. Crear perfiles Docker Compose para FalkorDB, Neo4j y Weaviate

**Objetivo:** Definir un entorno reproducible con perfiles independientes para comparar las tres plataformas sin mezclar sus servicios ni exponer contraseñas en el repositorio.

**Instrucciones:**

1. Crea un archivo de variables locales no versionado.

   ```bash
   cat > .env <<'EOF'
   NEO4J_PASSWORD=ChangeThisLocalPassword_2026!
   AZURE_OPENAI_ENDPOINT=https://REEMPLAZAR.openai.azure.com/
   AZURE_OPENAI_API_KEY=REEMPLAZAR
   AZURE_OPENAI_API_VERSION=2024-10-21
   AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o-mini
   AZURE_OPENAI_EMBEDDING_DEPLOYMENT=REEMPLAZAR_EMBEDDINGS
   QDRANT_URL=http://127.0.0.1:6333
   FALKORDB_HOST=127.0.0.1
   FALKORDB_PORT=6379
   FALKORDB_GRAPH=technical_knowledge_graph
   EOF
   ```

2. Protege el archivo `.env` para evitar que sea incluido en Git.

   ```bash
   grep -qxF ".env" .gitignore || echo ".env" >> .gitignore
   ```

3. Crea `docker-compose.yml`.

   ```bash
   cat > docker-compose.yml <<'EOF'
   services:
     falkordb:
       image: falkordb/falkordb:4.6.0
       profiles: ["falkor"]
       container_name: genai-falkordb
       ports:
         - "6379:6379"
       volumes:
         - falkordb_data:/data
       healthcheck:
         test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
         interval: 10s
         timeout: 5s
         retries: 10

     neo4j:
       image: neo4j:5.20.0-community
       profiles: ["neo4j"]
       container_name: genai-neo4j
       ports:
         - "7474:7474"
         - "7687:7687"
       environment:
         NEO4J_AUTH: neo4j/${NEO4J_PASSWORD}
         NEO4J_PLUGINS: '["apoc"]'
       volumes:
         - neo4j_data:/data
         - neo4j_logs:/logs

     weaviate:
       image: semitechnologies/weaviate:1.25.4
       profiles: ["weaviate"]
       container_name: genai-weaviate
       ports:
         - "8080:8080"
       environment:
         QUERY_DEFAULTS_LIMIT: 25
         AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: "true"
         PERSISTENCE_DATA_PATH: "/var/lib/weaviate"
         DEFAULT_VECTORIZER_MODULE: "none"
         CLUSTER_HOSTNAME: "node1"
       volumes:
         - weaviate_data:/var/lib/weaviate

   volumes:
     falkordb_data:
     neo4j_data:
     neo4j_logs:
     weaviate_data:
   EOF
   ```

4. Valida la sintaxis del archivo Compose.

   ```bash
   docker compose config >/dev/null && echo "docker-compose.yml válido"
   ```

5. Inicia únicamente FalkorDB.

   ```bash
   docker compose --profile falkor up -d
   ```

6. Espera a que el servicio esté disponible.

   ```bash
   docker compose ps
   docker exec genai-falkordb redis-cli ping
   ```

**Resultado esperado:**

- El contenedor `genai-falkordb` está en estado `running`.
- El comando `redis-cli ping` devuelve `PONG`.
- Los perfiles `neo4j` y `weaviate` están definidos, pero no necesariamente iniciados.

**Verificación:**

```bash
docker compose --profile falkor ps
docker logs --tail 30 genai-falkordb
```

Debes observar un servicio FalkorDB disponible en `127.0.0.1:6379`.

---

### Paso 3. Cargar el grafo técnico e indexar las entidades en FalkorDB

**Objetivo:** Importar `technical_graph.json` en el grafo persistente `technical_knowledge_graph`, crear índices y conservar propiedades de trazabilidad.

**Instrucciones:**

1. Crea el cargador `src/graphrag/load_falkordb_graph.py`.

   ```bash
   cat > src/graphrag/load_falkordb_graph.py <<'PY'
   import json
   import os
   import re
   from pathlib import Path

   from dotenv import load_dotenv
   from falkordb import FalkorDB

   load_dotenv()

   GRAPH_FILE = Path("data/technical_graph.json")
   GRAPH_NAME = os.getenv("FALKORDB_GRAPH", "technical_knowledge_graph")
   HOST = os.getenv("FALKORDB_HOST", "127.0.0.1")
   PORT = int(os.getenv("FALKORDB_PORT", "6379"))


   def safe_relation_type(value: str) -> str:
       normalized = re.sub(r"[^A-Za-z0-9_]", "_", str(value).upper())
       return normalized or "RELATED_TO"


   def normalize_entities(document: dict) -> list[dict]:
       raw_entities = document.get("entities", document.get("nodes", []))
       entities = []

       for item in raw_entities:
           entity_id = item.get("id") or item.get("entity_id")
           if not entity_id:
               continue

           properties = dict(item)
           properties["id"] = str(entity_id)
           properties["name"] = str(
               item.get("name")
               or item.get("label")
               or item.get("title")
               or entity_id
           )
           properties["type"] = str(
               item.get("type")
               or item.get("entity_type")
               or "Entity"
           )
           entities.append(properties)

       return entities


   def normalize_relationships(document: dict) -> list[dict]:
       raw_relations = document.get(
           "relationships",
           document.get("relations", document.get("edges", [])),
       )
       relationships = []

       for item in raw_relations:
           source = item.get("source") or item.get("source_id") or item.get("from")
           target = item.get("target") or item.get("target_id") or item.get("to")
           relation_type = (
               item.get("type")
               or item.get("relation")
               or item.get("relationship")
               or "RELATED_TO"
           )

           if not source or not target:
               continue

           properties = dict(item)
           for key in ("source", "source_id", "from", "target", "target_id", "to"):
               properties.pop(key, None)

           properties["relation_type"] = safe_relation_type(relation_type)
           relationships.append(
               {
                   "source": str(source),
                   "target": str(target),
                   "relation_type": safe_relation_type(relation_type),
                   "properties": properties,
               }
           )

       return relationships


   def main() -> None:
       if not GRAPH_FILE.exists():
           raise FileNotFoundError(f"No existe {GRAPH_FILE}")

       document = json.loads(GRAPH_FILE.read_text(encoding="utf-8"))
       entities = normalize_entities(document)
       relationships = normalize_relationships(document)

       db = FalkorDB(host=HOST, port=PORT)
       graph = db.select_graph(GRAPH_NAME)

       graph.query("MATCH (n) DETACH DELETE n")

       try:
           graph.query("CREATE INDEX FOR (n:Entity) ON (n.id)")
           graph.query("CREATE INDEX FOR (n:Entity) ON (n.name)")
       except Exception as error:
           print(f"Índices existentes o sintaxis alternativa requerida: {error}")

       for entity in entities:
           graph.query(
               """
               MERGE (n:Entity {id: $id})
               SET n += $properties
               """,
               {
                   "id": entity["id"],
                   "properties": entity,
               },
           )

       for relationship in relationships:
           relation_type = relationship["relation_type"]
           graph.query(
               f"""
               MATCH (source:Entity {{id: $source}})
               MATCH (target:Entity {{id: $target}})
               MERGE (source)-[r:{relation_type}]->(target)
               SET r += $properties
               """,
               {
                   "source": relationship["source"],
                   "target": relationship["target"],
                   "properties": relationship["properties"],
               },
           )

       count_nodes = graph.query("MATCH (n:Entity) RETURN count(n)")
       count_edges = graph.query("MATCH ()-[r]->() RETURN count(r)")

       print(f"Grafo cargado: {GRAPH_NAME}")
       print(f"Entidades procesadas: {len(entities)}")
       print(f"Relaciones procesadas: {len(relationships)}")
       print(f"Resultado de nodos: {count_nodes.result_set}")
       print(f"Resultado de relaciones: {count_edges.result_set}")


   if __name__ == "__main__":
       main()
   PY
   ```

2. Ejecuta el cargador.

   ```bash
   python src/graphrag/load_falkordb_graph.py
   ```

3. Ejecuta una consulta Cypher desde Python para inspeccionar entidades.

   ```bash
   python - <<'PY'
   import os
   from dotenv import load_dotenv
   from falkordb import FalkorDB

   load_dotenv()
   db = FalkorDB(
       host=os.getenv("FALKORDB_HOST", "127.0.0.1"),
       port=int(os.getenv("FALKORDB_PORT", "6379")),
   )
   graph = db.select_graph(os.getenv("FALKORDB_GRAPH", "technical_knowledge_graph"))

   result = graph.query("""
       MATCH (n:Entity)
       RETURN n.id, n.name, n.type
       LIMIT 10
   """)
   for row in result.result_set:
       print(row)
   PY
   ```

4. Ejecuta una consulta de expansión de dos saltos. Sustituye `Atlas` por una entidad existente si tu corpus usa otros nombres.

   ```bash
   python - <<'PY'
   import os
   from dotenv import load_dotenv
   from falkordb import FalkorDB

   load_dotenv()
   db = FalkorDB(
       host=os.getenv("FALKORDB_HOST", "127.0.0.1"),
       port=int(os.getenv("FALKORDB_PORT", "6379")),
   )
   graph = db.select_graph(os.getenv("FALKORDB_GRAPH", "technical_knowledge_graph"))

   result = graph.query("""
       MATCH p=(seed:Entity)-[*1..2]-(related:Entity)
       WHERE toLower(seed.name) = toLower($name)
       RETURN
           [node IN nodes(p) | node.name] AS path_nodes,
           [rel IN relationships(p) | type(rel)] AS path_relations
       LIMIT 20
   """, {"name": "Atlas"})

   for row in result.result_set:
       print(row)
   PY
   ```

**Resultado esperado:**

- FalkorDB contiene el grafo `technical_knowledge_graph`.
- Las entidades se almacenan con la etiqueta `Entity`.
- Las relaciones se almacenan con tipos Cypher normalizados, por ejemplo `PROCESA`, `GESTIONADO_POR` o `APLICA_A`.
- Las consultas devuelven caminos de uno o dos saltos.

**Verificación:**

```bash
python - <<'PY'
import os
from dotenv import load_dotenv
from falkordb import FalkorDB

load_dotenv()
db = FalkorDB(
    host=os.getenv("FALKORDB_HOST", "127.0.0.1"),
    port=int(os.getenv("FALKORDB_PORT", "6379")),
)
graph = db.select_graph(os.getenv("FALKORDB_GRAPH", "technical_knowledge_graph"))
result = graph.query("MATCH (n:Entity) RETURN count(n)")
print("Número de entidades en FalkorDB:", result.result_set[0][0])
assert result.result_set[0][0] > 0
PY
```

---

### Paso 4. Implementar el recuperador relacional de FalkorDB

**Objetivo:** Crear un componente reutilizable que reciba identificadores de entidades, ejecute una expansión parametrizada y devuelva caminos trazables del subgrafo.

**Instrucciones:**

1. Crea `src/graphrag/falkordb_graph_retriever.py`.

   ```bash
   cat > src/graphrag/falkordb_graph_retriever.py <<'PY'
   import os
   from typing import Any

   from dotenv import load_dotenv
   from falkordb import FalkorDB

   load_dotenv()


   class FalkorDBGraphRetriever:
       def __init__(self) -> None:
           self.graph_name = os.getenv(
               "FALKORDB_GRAPH",
               "technical_knowledge_graph",
           )
           self.db = FalkorDB(
               host=os.getenv("FALKORDB_HOST", "127.0.0.1"),
               port=int(os.getenv("FALKORDB_PORT", "6379")),
           )
           self.graph = self.db.select_graph(self.graph_name)

       def find_entities_by_name(self, text: str, limit: int = 10) -> list[list[Any]]:
           result = self.graph.query(
               """
               MATCH (n:Entity)
               WHERE toLower(n.name) CONTAINS toLower($text)
               RETURN n.id, n.name, n.type, n.source
               LIMIT $limit
               """,
               {"text": text, "limit": limit},
           )
           return result.result_set

       def expand_entities(
           self,
           entity_ids: list[str],
           max_hops: int = 2,
           limit: int = 30,
       ) -> list[dict[str, Any]]:
           if not entity_ids:
               return []

           max_hops = min(max(max_hops, 1), 2)

           query = f"""
           MATCH p=(seed:Entity)-[*1..{max_hops}]-(related:Entity)
           WHERE seed.id IN $entity_ids
           RETURN
               [node IN nodes(p) | {{
                   id: node.id,
                   name: node.name,
                   type: node.type,
                   source: node.source
               }}] AS nodes,
               [rel IN relationships(p) | {{
                   type: type(rel),
                   source: rel.source,
                   source_document: rel.source_document
               }}] AS relationships
           LIMIT $limit
           """

           result = self.graph.query(
               query,
               {
                   "entity_ids": entity_ids,
                   "limit": limit,
               },
           )

           paths = []
           for row in result.result_set:
               paths.append(
                   {
                       "nodes": row[0],
                       "relationships": row[1],
                   }
               )
           return paths
   PY
   ```

2. Prueba el recuperador con identificadores reales extraídos de tu JSON.

   ```bash
   python - <<'PY'
   import json
   from src.graphrag.falkordb_graph_retriever import FalkorDBGraphRetriever

   graph_data = json.load(open("data/technical_graph.json", encoding="utf-8"))
   entities = graph_data.get("entities", graph_data.get("nodes", []))
   sample_id = entities[0].get("id") or entities[0].get("entity_id")

   retriever = FalkorDBGraphRetriever()
   paths = retriever.expand_entities([sample_id], max_hops=2, limit=10)

   print(f"Entidad inicial: {sample_id}")
   print(f"Caminos recuperados: {len(paths)}")
   for path in paths[:3]:
       print(path)
   PY
   ```

3. Confirma que la consulta usa parámetros para valores externos, especialmente `entity_ids` y `limit`. El único elemento interpolado es `max_hops`, que se limita explícitamente a valores entre uno y dos y no procede de texto libre.

4. Conserva el límite de dos saltos. Una expansión no controlada puede incorporar relaciones irrelevantes, elevar el coste de generación y reducir la precisión de las respuestas.

**Resultado esperado:**

- El archivo `falkordb_graph_retriever.py` expone una clase reutilizable.
- El recuperador devuelve caminos compuestos por nodos y relaciones.
- La expansión se limita a uno o dos saltos.
- La entrada externa se pasa mediante parámetros Cypher.

**Verificación:**

```bash
python - <<'PY'
from src.graphrag.falkordb_graph_retriever import FalkorDBGraphRetriever

retriever = FalkorDBGraphRetriever()
print(retriever.find_entities_by_name("Atlas", limit=5))
PY
```

Si no existe `Atlas`, utiliza otro nombre presente en `technical_graph.json`.

---

### Paso 5. Construir el flujo híbrido GraphRAG con Qdrant, FalkorDB y Azure OpenAI

**Objetivo:** Recuperar evidencia semántica top-5 desde Qdrant, expandir entidades relacionadas en FalkorDB y generar una respuesta con citas documentales.

**Instrucciones:**

1. Crea `src/graphrag/hybrid_graphrag.py`.

   ```bash
   cat > src/graphrag/hybrid_graphrag.py <<'PY'
   import json
   import os
   import sys
   from typing import Any

   from dotenv import load_dotenv
   from openai import AzureOpenAI
   from qdrant_client import QdrantClient

   from src.graphrag.falkordb_graph_retriever import FalkorDBGraphRetriever

   load_dotenv()

   COLLECTION_NAME = "technical_chunks"


   def azure_client() -> AzureOpenAI:
       return AzureOpenAI(
           azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
           api_key=os.environ["AZURE_OPENAI_API_KEY"],
           api_version=os.getenv("AZURE_OPENAI_API_VERSION", "2024-10-21"),
       )


   def embed_question(question: str) -> list[float]:
       client = azure_client()
       response = client.embeddings.create(
           model=os.environ["AZURE_OPENAI_EMBEDDING_DEPLOYMENT"],
           input=question,
       )
       return response.data[0].embedding


   def extract_entity_ids(payload: dict[str, Any]) -> list[str]:
       entity_ids = payload.get("entity_ids", [])

       if isinstance(entity_ids, str):
           return [entity_ids]

       if isinstance(entity_ids, list):
           return [str(item) for item in entity_ids]

       entity_id = payload.get("entity_id")
       if entity_id:
           return [str(entity_id)]

       entities = payload.get("entities", [])
       if isinstance(entities, list):
           return [
               str(item.get("id", item))
               for item in entities
               if isinstance(item, (str, dict))
           ]

       return []


   def vector_retrieve(question: str, limit: int = 5) -> list[dict[str, Any]]:
       qdrant = QdrantClient(url=os.getenv("QDRANT_URL", "http://127.0.0.1:6333"))
       vector = embed_question(question)

       results = qdrant.search(
           collection_name=COLLECTION_NAME,
           query_vector=vector,
           limit=limit,
           with_payload=True,
       )

       chunks = []
       for point in results:
           payload = point.payload or {}
           chunks.append(
               {
                   "chunk_id": str(point.id),
                   "score": point.score,
                   "text": payload.get("text", payload.get("content", "")),
                   "source": payload.get(
                       "source",
                       payload.get("source_document", "Fuente no disponible"),
                   ),
                   "entity_ids": extract_entity_ids(payload),
               }
           )
       return chunks


   def format_paths(paths: list[dict[str, Any]]) -> list[str]:
       evidence = []

       for path in paths:
           nodes = path.get("nodes", [])
           relationships = path.get("relationships", [])

           if len(nodes) < 2 or not relationships:
               continue

           parts = []
           for index, relation in enumerate(relationships):
               source_name = nodes[index].get("name", nodes[index].get("id"))
               target_name = nodes[index + 1].get("name", nodes[index + 1].get("id"))
               relation_type = relation.get("type", "RELATED_TO")
               source = (
                   relation.get("source_document")
                   or relation.get("source")
                   or nodes[index].get("source")
                   or "Fuente de relación no disponible"
               )
               parts.append(
                   f"{source_name} --{relation_type}--> {target_name} "
                   f"[Fuente: {source}]"
               )

           evidence.append(" | ".join(parts))

       return sorted(set(evidence))


   def build_context(chunks: list[dict[str, Any]], paths: list[dict[str, Any]]) -> str:
       chunk_lines = []
       for chunk in chunks:
           chunk_lines.append(
               f"- Fragmento {chunk['chunk_id']} "
               f"(similitud={chunk['score']:.4f}, fuente={chunk['source']}): "
               f"{chunk['text']}"
           )

       path_lines = format_paths(paths)

       return (
           "EVIDENCIA DOCUMENTAL RECUPERADA:\n"
           + "\n".join(chunk_lines)
           + "\n\nEVIDENCIA RELACIONAL RECUPERADA:\n"
           + ("\n".join(f"- {line}" for line in path_lines) or "- Sin caminos relacionados.")
       )


   def answer_question(question: str) -> dict[str, Any]:
       chunks = vector_retrieve(question, limit=5)
       entity_ids = sorted(
           {
               entity_id
               for chunk in chunks
               for entity_id in chunk["entity_ids"]
           }
       )

       graph_retriever = FalkorDBGraphRetriever()
       paths = graph_retriever.expand_entities(
           entity_ids=entity_ids,
           max_hops=2,
           limit=30,
       )

       context = build_context(chunks, paths)

       completion = azure_client().chat.completions.create(
           model=os.environ["AZURE_OPENAI_CHAT_DEPLOYMENT"],
           temperature=0,
           messages=[
               {
                   "role": "system",
                   "content": (
                       "Eres un asistente técnico basado en evidencia. "
                       "Responde únicamente con la evidencia suministrada. "
                       "Distingue entre hechos explícitos e inferencias basadas "
                       "en caminos relacionales. Si la evidencia es insuficiente, "
                       "indícalo claramente. Incluye una sección final titulada "
                       "'Fuentes' con las referencias documentales utilizadas."
                   ),
               },
               {
                   "role": "user",
                   "content": f"Pregunta:\n{question}\n\n{context}",
               },
           ],
       )

       return {
           "question": question,
           "answer": completion.choices[0].message.content,
           "vector_chunks": chunks,
           "seed_entity_ids": entity_ids,
           "graph_paths": paths,
           "context": context,
       }


   if __name__ == "__main__":
       if len(sys.argv) < 2:
           raise SystemExit('Uso: python -m src.graphrag.hybrid_graphrag "pregunta"')

       result = answer_question(sys.argv[1])
       print(json.dumps(result, ensure_ascii=False, indent=2))
   PY
   ```

2. Ejecuta una pregunta de una relación. Adapta la pregunta al contenido real de tu corpus.

   ```bash
   python -m src.graphrag.hybrid_graphrag \
     "¿Qué sistema procesa facturas?"
   ```

3. Ejecuta una pregunta que requiera varias relaciones.

   ```bash
   python -m src.graphrag.hybrid_graphrag \
     "¿Qué área es responsable operativa de cumplir la política de retención de facturas?"
   ```

4. Guarda una salida de evidencia para revisión.

   ```bash
   python -m src.graphrag.hybrid_graphrag \
     "¿Qué área es responsable operativa de cumplir la política de retención de facturas?" \
     > reports/graphrag_evidence.json
   ```

5. Revisa las secciones `vector_chunks`, `seed_entity_ids`, `graph_paths` y `answer`.

   ```bash
   python -m json.tool reports/graphrag_evidence.json | sed -n '1,220p'
   ```

**Resultado esperado:**

- Qdrant devuelve cinco fragmentos semánticamente relevantes.
- Los metadatos de los fragmentos permiten identificar entidades semilla.
- FalkorDB expande el contexto hasta dos saltos.
- La respuesta final incluye fuentes y no afirma hechos que no estén respaldados por fragmentos o caminos recuperados.
- Cuando la conclusión depende de un camino, se expresa como inferencia fundamentada.

**Verificación:**

Confirma los siguientes criterios en `reports/graphrag_evidence.json`:

```bash
python - <<'PY'
import json

result = json.load(open("reports/graphrag_evidence.json", encoding="utf-8"))

assert len(result["vector_chunks"]) <= 5
assert "Fuentes" in result["answer"]
assert "context" in result
print("Fragmentos vectoriales:", len(result["vector_chunks"]))
print("Entidades semilla:", len(result["seed_entity_ids"]))
print("Caminos de grafo:", len(result["graph_paths"]))
print("Validación estructural correcta.")
PY
```

---

### Paso 6. Cargar una muestra comparativa en Neo4j y Weaviate

**Objetivo:** Realizar una carga representativa para contrastar el modelo de grafos de FalkorDB y Neo4j frente al enfoque vectorial orientado a objetos de Weaviate.

**Instrucciones:**

1. Detén FalkorDB temporalmente si el equipo tiene memoria limitada.

   ```bash
   docker compose --profile falkor stop
   ```

2. Inicia Neo4j.

   ```bash
   docker compose --profile neo4j up -d
   docker compose --profile neo4j ps
   ```

3. Espera hasta que Neo4j esté disponible.

   ```bash
   until curl -s http://127.0.0.1:7474 >/dev/null; do
     echo "Esperando a Neo4j..."
     sleep 5
   done
   echo "Neo4j disponible en http://127.0.0.1:7474"
   ```

4. Abre la interfaz Neo4j Browser en `http://127.0.0.1:7474` e inicia sesión con:

   - Usuario: `neo4j`
   - Contraseña: el valor de `NEO4J_PASSWORD` en `.env`

5. Ejecuta en Neo4j Browser una carga manual representativa usando datos equivalentes a los del corpus.

   ```cypher
   CREATE CONSTRAINT entity_id IF NOT EXISTS
   FOR (n:Entity) REQUIRE n.id IS UNIQUE;

   MERGE (atlas:Entity {
     id: "system-atlas",
     name: "Atlas",
     type: "System",
     source: "Inventario_Sistemas.xlsx, fila Atlas"
   })
   MERGE (facturas:Entity {
     id: "asset-facturas",
     name: "Facturas",
     type: "InformationAsset",
     source: "Politica_RET-01.pdf, sección 3"
   })
   MERGE (finanzas:Entity {
     id: "area-finanzas",
     name: "Finanzas",
     type: "BusinessArea",
     source: "Inventario_Sistemas.xlsx, fila Atlas"
   })
   MERGE (retencion:Entity {
     id: "policy-ret-01",
     name: "RET-01",
     type: "Policy",
     source: "Politica_RET-01.pdf, sección 3"
   })
   MERGE (atlas)-[:PROCESA {
     source_document: "Inventario_Sistemas.xlsx, fila Atlas"
   }]->(facturas)
   MERGE (atlas)-[:GESTIONADO_POR {
     source_document: "Inventario_Sistemas.xlsx, fila Atlas"
   }]->(finanzas)
   MERGE (retencion)-[:APLICA_A {
     source_document: "Politica_RET-01.pdf, sección 3"
   }]->(facturas);
   ```

6. Ejecuta una consulta equivalente a la usada en FalkorDB.

   ```cypher
   MATCH p=(policy:Entity {name: "RET-01"})-[:APLICA_A]->(asset)<-[:PROCESA]-(system)-[:GESTIONADO_POR]->(area)
   RETURN policy.name, asset.name, system.name, area.name;
   ```

7. Detén Neo4j antes de iniciar Weaviate si necesitas liberar memoria.

   ```bash
   docker compose --profile neo4j stop
   ```

8. Inicia Weaviate.

   ```bash
   docker compose --profile weaviate up -d
   curl -s http://127.0.0.1:8080/v1/.well-known/ready
   ```

9. Crea una clase vectorial para una muestra de fragmentos técnicos. Weaviate no generará embeddings automáticamente porque el vectorizador predeterminado se ha establecido en `none`.

   ```bash
   curl -s \
     -X POST http://127.0.0.1:8080/v1/schema \
     -H "Content-Type: application/json" \
     -d '{
       "class": "TechnicalChunk",
       "description": "Fragmentos técnicos para comparación vectorial",
       "vectorizer": "none",
       "properties": [
         {"name": "chunkId", "dataType": ["text"]},
         {"name": "content", "dataType": ["text"]},
         {"name": "source", "dataType": ["text"]},
         {"name": "entityIds", "dataType": ["text[]"]}
       ]
     }' | python -m json.tool
   ```

10. Inserta una muestra representativa. Para una comparación real, sustituye el vector de ejemplo por embeddings generados con el mismo modelo usado para Qdrant.

   ```bash
   curl -s \
     -X POST http://127.0.0.1:8080/v1/objects \
     -H "Content-Type: application/json" \
     -d '{
       "class": "TechnicalChunk",
       "properties": {
         "chunkId": "sample-ret-01",
         "content": "La política RET-01 establece requisitos de retención para facturas.",
         "source": "Politica_RET-01.pdf, sección 3",
         "entityIds": ["policy-ret-01", "asset-facturas"]
       },
       "vector": [0.01, 0.02, 0.03, 0.04]
     }' | python -m json.tool
   ```

11. Consulta la muestra insertada.

   ```bash
   curl -s \
     -X POST http://127.0.0.1:8080/v1/graphql \
     -H "Content-Type: application/json" \
     -d '{
       "query": "{ Get { TechnicalChunk { chunkId content source entityIds } } }"
     }' | python -m json.tool
   ```

**Resultado esperado:**

- Neo4j ejecuta el mismo patrón Cypher relacional que FalkorDB.
- Weaviate almacena objetos y vectores, pero la expansión relacional no es su capacidad principal.
- La comparación evidencia que un índice vectorial por sí solo no reemplaza consultas relacionales de múltiples saltos.

**Verificación:**

| Plataforma | Verificación |
|---|---|
| FalkorDB | `MATCH` de entidades y caminos de hasta dos saltos devuelve resultados. |
| Neo4j | La consulta `RET-01 → Facturas ← Atlas → Finanzas` devuelve un camino. |
| Weaviate | La clase `TechnicalChunk` devuelve el objeto insertado mediante GraphQL. |

---

### Paso 7. Completar la matriz de decisión tecnológica

**Objetivo:** Seleccionar la plataforma adecuada para distintos escenarios, evitando asumir que una única base de datos es óptima para todos los tipos de recuperación.

**Instrucciones:**

1. Crea el archivo `reports/matriz_decision_graphrag.md`.

   ```bash
   cat > reports/matriz_decision_graphrag.md <<'EOF'
   # Matriz de decisión: FalkorDB, Neo4j y Weaviate

   | Criterio | FalkorDB 4.6.0 | Neo4j Community 5.20.0 | Weaviate 1.25.4 |
   |---|---|---|---|
   | Modelo de datos | Grafo de propiedades sobre arquitectura compatible con Redis. | Grafo de propiedades maduro con Cypher. | Objetos, propiedades y vectores; referencias entre objetos disponibles, pero no orientado a recorridos complejos. |
   | Consultas relacionales | Cypher, recorridos y patrones de grafo eficientes para subgrafos acotados. | Cypher muy expresivo, herramientas avanzadas y amplio ecosistema de grafos. | GraphQL y búsqueda vectorial; menos adecuado para razonamiento por caminos complejos. |
   | Búsqueda vectorial | Puede complementar el grafo, pero en esta práctica Qdrant asume la recuperación vectorial principal. | Capacidades vectoriales disponibles según edición y configuración, aunque requiere evaluación operativa específica. | Capacidad vectorial nativa y modelo orientado a recuperación semántica. |
   | Operación | Ligero para escenarios que ya usan Redis/FalkorDB y requieren grafos de baja latencia. | Mayor madurez operativa, administración, modelado y observabilidad especializada. | Operación enfocada en índices vectoriales, esquemas de objetos y APIs HTTP/GraphQL. |
   | Infraestructura | Un servicio de grafo; apropiado para despliegues compactos. | Requiere planificación de memoria, almacenamiento y operación de base de datos de grafos. | Requiere planificación de vectores, dimensiones, memoria e indexación. |
   | Latencia esperada | Baja para vecindarios y patrones de grafo acotados. | Buena para patrones complejos; depende del diseño, índices y tamaño del grafo. | Buena para recuperación vectorial; no sustituye recorridos de grafo profundos. |
   | Escalabilidad | Adecuada para grafos operativos y aplicaciones con necesidad de rapidez. | Adecuada para dominios de grafo complejos y gobierno de datos más exigente. | Adecuada para recuperación semántica a gran escala. |
   | Mejor escenario | GraphRAG híbrido con Qdrant y expansión relacional de uno o dos saltos. | Gestión de conocimiento empresarial compleja, análisis de fraude, dependencias y consultas avanzadas. | RAG vectorial, descubrimiento semántico y recuperación de documentos o productos. |
   EOF
   ```

2. Añade una recomendación explícita al final del archivo.

   ```bash
   cat >> reports/matriz_decision_graphrag.md <<'EOF'

   ## Recomendación por escenario

   1. **Asistente técnico con Qdrant ya operativo y necesidad de conectar sistemas, políticas, activos y áreas:** usar **FalkorDB + Qdrant**. Qdrant recupera evidencia semántica y FalkorDB aporta expansión relacional de baja latencia.

   2. **Grafo corporativo con consultas complejas, análisis de impacto, reglas de gobierno y equipos especializados en administración de grafos:** evaluar **Neo4j**. Su madurez de ecosistema y expresividad son especialmente valiosas cuando el grafo es un activo central de la organización.

   3. **Buscador semántico de documentos, catálogo de productos o recuperación basada principalmente en similitud vectorial:** usar **Weaviate** o mantener **Qdrant**, según los requisitos de API, operación y esquema. No se recomienda como sustituto directo de un motor de grafos cuando se necesitan recorridos relacionales de varios saltos.

   4. **Decisión para esta práctica:** seleccionar **FalkorDB + Qdrant** porque el objetivo es combinar top-5 vectorial con expansión controlada de hasta dos saltos, manteniendo separado el almacén vectorial del almacén relacional.
   EOF
   ```

3. Revisa la matriz.

   ```bash
   cat reports/matriz_decision_graphrag.md
   ```

**Resultado esperado:**

- La matriz diferencia claramente recuperación vectorial, navegación relacional y operación.
- La recomendación reconoce que FalkorDB, Neo4j y Weaviate resuelven necesidades distintas.
- La alternativa seleccionada para el laboratorio es FalkorDB + Qdrant.

**Verificación:**

```bash
grep -E "FalkorDB|Neo4j|Weaviate|FalkorDB \+ Qdrant" reports/matriz_decision_graphrag.md
```

## Validación y pruebas

Ejecuta las siguientes validaciones antes de finalizar.

1. Verifica la sintaxis de los módulos Python.

   ```bash
   python -m py_compile \
     src/graphrag/load_falkordb_graph.py \
     src/graphrag/falkordb_graph_retriever.py \
     src/graphrag/hybrid_graphrag.py
   ```

2. Confirma que FalkorDB contiene entidades y relaciones.

   ```bash
   docker compose --profile falkor up -d

   python - <<'PY'
   import os
   from dotenv import load_dotenv
   from falkordb import FalkorDB

   load_dotenv()
   graph = FalkorDB(
       host=os.getenv("FALKORDB_HOST", "127.0.0.1"),
       port=int(os.getenv("FALKORDB_PORT", "6379")),
   ).select_graph(os.getenv("FALKORDB_GRAPH", "technical_knowledge_graph"))

   nodes = graph.query("MATCH (n:Entity) RETURN count(n)").result_set[0][0]
   edges = graph.query("MATCH ()-[r]->() RETURN count(r)").result_set[0][0]

   print(f"Nodos: {nodes}")
   print(f"Relaciones: {edges}")
   assert nodes > 0
   assert edges > 0
   PY
   ```

3. Ejecuta una prueba GraphRAG de una relación y conserva el resultado.

   ```bash
   python -m src.graphrag.hybrid_graphrag \
     "¿Qué sistema procesa facturas?" \
     > reports/graphrag_single_relation.json
   ```

4. Ejecuta una prueba GraphRAG de varias relaciones y conserva el resultado.

   ```bash
   python -m src.graphrag.hybrid_graphrag \
     "¿Qué área gestiona el sistema que procesa facturas sujetas a retención?" \
     > reports/graphrag_multi_relation.json
   ```

5. Ejecuta el harness disponible en el repositorio. Consulta primero sus opciones para respetar el contrato creado en el laboratorio 02-00-04.

   ```bash
   python -m src.evaluation.harness --help
   ```

   Si el harness admite suites, ejecuta la suite GraphRAG:

   ```bash
   python -m src.evaluation.harness --suite graphrag
   ```

6. Verifica manualmente las respuestas del harness y de los archivos JSON con estos criterios:

   | Criterio | Resultado esperado |
   |---|---|
   | Recuperación vectorial | Se recuperan como máximo cinco fragmentos. |
   | Expansión del grafo | No supera dos saltos. |
   | Trazabilidad | La respuesta incluye fuentes documentales. |
   | Pregunta de una relación | La respuesta identifica correctamente una relación explícita. |
   | Pregunta de varias relaciones | La respuesta conecta entidades mediante un camino recuperado. |
   | Seguridad | No hay claves de Azure OpenAI en código, imágenes ni archivos versionados. |
   | Incertidumbre | Si falta evidencia, la respuesta lo declara en lugar de inventar conclusiones. |

7. Revisa el estado de Git y realiza el commit de la práctica conforme a la convención indicada por el instructor.

   ```bash
   git status
   git add \
     docker-compose.yml \
     .gitignore \
     src/graphrag \
     reports/matriz_decision_graphrag.md
   git commit -m "lab-03-00-04"
   ```

> No agregues `.env`, credenciales, volúmenes Docker ni archivos temporales de gran tamaño al commit.

## Solución de problemas

### Problema 1. FalkorDB no carga el grafo o aparece un error al crear índices

**Síntomas:**

- El cargador muestra un error de conexión a `127.0.0.1:6379`.
- `redis-cli ping` no devuelve `PONG`.
- La sentencia `CREATE INDEX FOR (n:Entity) ON (n.id)` informa que el índice ya existe o que la sintaxis no es compatible.

**Causa:**

El contenedor FalkorDB no se ha iniciado correctamente, el puerto 6379 está ocupado por otra instancia Redis, o el motor detecta que el índice ya existe después de ejecutar el cargador más de una vez.

**Corrección:**

1. Comprueba el estado y los registros del contenedor.

   ```bash
   docker compose --profile falkor ps
   docker logs genai-falkordb --tail 100
   ```

2. Verifica el puerto local.

   ```bash
   ss -ltnp | grep 6379 || true
   ```

3. Reinicia FalkorDB si no hay otro servicio que deba conservar ese puerto.

   ```bash
   docker compose --profile falkor down
   docker compose --profile falkor up -d
   ```

4. Si el error se refiere únicamente a un índice existente, continúa con la carga. El script ya captura ese caso; lo importante es que los nodos y relaciones se inserten correctamente.

---

### Problema 2. El flujo GraphRAG recupera fragmentos, pero no encuentra caminos en FalkorDB

**Síntomas:**

- `vector_chunks` contiene resultados.
- `seed_entity_ids` está vacío o contiene valores que no existen en FalkorDB.
- `graph_paths` es una lista vacía.
- La respuesta sólo usa evidencia vectorial y no incluye relaciones.

**Causa:**

Los identificadores de entidad almacenados en los metadatos de Qdrant no coinciden con los identificadores de `technical_graph.json`, o los campos de metadatos utilizan nombres distintos de `entity_ids`, `entities` o `entity_id`.

**Corrección:**

1. Inspecciona un punto de Qdrant.

   ```bash
   curl -s \
     -X POST http://127.0.0.1:6333/collections/technical_chunks/points/scroll \
     -H "Content-Type: application/json" \
     -d '{"limit": 3, "with_payload": true, "with_vector": false}' \
     | python -m json.tool
   ```

2. Inspecciona los identificadores de las entidades cargadas.

   ```bash
   python - <<'PY'
   import json
   graph = json.load(open("data/technical_graph.json", encoding="utf-8"))
   entities = graph.get("entities", graph.get("nodes", []))
   for entity in entities[:20]:
       print(entity.get("id") or entity.get("entity_id"), entity.get("name"))
   PY
   ```

3. Corrige la función `extract_entity_ids()` para usar el nombre real del campo de tus metadatos, o vuelve a cargar Qdrant con los identificadores canónicos del grafo.

4. Ejecuta nuevamente el cargador de FalkorDB y el flujo híbrido.

   ```bash
   python src/graphrag/load_falkordb_graph.py
   python -m src.graphrag.hybrid_graphrag \
     "¿Qué área gestiona el sistema que procesa facturas?"
   ```

## Limpieza

1. Detén los perfiles Docker Compose iniciados durante la práctica.

   ```bash
   docker compose --profile falkor down
   docker compose --profile neo4j down
   docker compose --profile weaviate down
   ```

2. Si necesitas liberar espacio de forma definitiva y no necesitas conservar los datos comparativos, elimina los volúmenes creados por esta práctica.

   ```bash
   docker compose --profile falkor down -v
   docker compose --profile neo4j down -v
   docker compose --profile weaviate down -v
   ```

3. Conserva los archivos de código y los informes en el repositorio:

   ```bash
   ls -la src/graphrag
   ls -la reports
   ```

4. Elimina únicamente resultados temporales si no se requieren como evidencia de evaluación.

   ```bash
   rm -f reports/graphrag_single_relation.json
   rm -f reports/graphrag_multi_relation.json
   ```

5. Nunca elimines `data/technical_graph.json` ni la colección Qdrant `technical_chunks` si serán reutilizados en prácticas posteriores.

## Resumen

En esta práctica implementaste una arquitectura GraphRAG híbrida con responsabilidades separadas:

1. **Qdrant** recupera los cinco fragmentos más cercanos semánticamente a la pregunta.
2. Los metadatos de los fragmentos determinan las **entidades semilla**.
3. **FalkorDB** expande el subgrafo técnico mediante consultas Cypher parametrizadas y un máximo de dos saltos.
4. **Azure OpenAI** genera una respuesta restringida a la evidencia recuperada, con fuentes documentales y distinción entre hechos e inferencias.
5. **Neo4j** se utilizó como referencia para consultas de grafos maduras y complejas.
6. **Weaviate** se evaluó como alternativa centrada en objetos y recuperación vectorial.

La decisión para este escenario es usar **FalkorDB + Qdrant** cuando se necesita combinar recuperación semántica existente con navegación relacional acotada, de baja latencia y trazable. Neo4j resulta más apropiado cuando el grafo es un activo corporativo central con requisitos avanzados de consulta y operación. Weaviate es adecuado para escenarios dominados por búsqueda vectorial, pero no sustituye por sí solo un motor de grafos para razonamiento relacional de varios saltos.

### Recursos opcionales

- [Documentación de FalkorDB](https://docs.falkordb.com/)
- [Documentación de Cypher en Neo4j](https://neo4j.com/docs/cypher-manual/current/)
- [Documentación de Weaviate](https://weaviate.io/developers/weaviate)
- [Documentación de Qdrant](https://qdrant.tech/documentation/)
- [Microsoft Research GraphRAG](https://www.microsoft.com/en-us/research/project/graphrag/)
