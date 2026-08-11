# Embedded Source Schema (ESS) Specification
**Version 1.1**  
*A Universal XMP-based Standard for Embedding Generative Source Payloads in Digital Assets*
---
## 1. Abstract
The **Embedded Source Schema (ESS)** defines an open, vendor-neutral metadata standard for embedding generative source definitions—such as diagram markup (`mermaid`, `plantuml`, `excalidraw`), CAD scripts (`openscad`), vector definitions, and structured prompts—directly into rendered visual assets (PNG, SVG, PDF, JPEG).
By building upon the **ISO 16684-1 Extensible Metadata Platform (XMP)** and aligning payload format identifiers with standard Markdown code block tags, ESS provides a seamless round-trip pipeline: source code rendered by any gateway or engine remains embedded within the image, allowing any tool to extract, decode, and recreate the original source code without proprietary plugins or external databases.
---
## 2. Motivation & Design Goals
### 2.1 The Problem
Modern documentation pipelines rely heavily on text-to-diagram and text-to-asset generators. Once rendered into binary image formats (PNG, SVG, PDF), the connection between the image and its source code is lost. Proprietary workarounds exist (e.g., Draw.io's custom PNG `zTXt` chunks, PlantUML's comment fields), but they lack interoperability, break standard parsers, or require specialized tools to read.
### 2.2 Design Principles
1. **Zero Proprietary Parsers**: Built on standard XMP, allowing existing metadata extraction utilities (e.g., `exiftool`, Adobe XMP SDK) to read payloads out of the box.
2. **Markdown Ecosystem Parity**: Format identifiers match the exact language tags used in Markdown triple-backtick code blocks (e.g., ```` ```mermaid ````).
3. **Transport Efficiency**: Payloads default to `gzip+base64` encoding, matching modern web application transport layers and preventing XML escaping issues.
4. **Format Agnostic**: Applies uniformly to raster images (PNG, JPEG), vector graphics (SVG), and printable documents (PDF).
---
## 3. Specification
### 3.1 Namespace & Metadata Identification
* **Namespace URI:** `http://ns.embedded-source.org/1.0/`
* **Preferred Prefix:** `ess`
* **Data Model:** RDF/XML (XMP ISO 16684-1 compliant)
### 3.2 Property Dictionary

| Property | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `ess:Format` | Text | **Required** | The Markdown language identifier specifying the syntax/engine (e.g., `mermaid`, `plantuml`, `excalidraw`, `diagramsnet`, `ditaa`, `openscad`). |
| `ess:Payload` | Text | **Required** | The source definition string, encoded according to `ess:Encoding`. |
| `ess:Encoding` | Choice | Optional | Encoding format of `ess:Payload`. Permitted values: `gzip+base64` (**default**), `deflate+base64`, `raw`. |
| `ess:Tool` | Text | Optional | Name and version of the software generator or gateway (e.g., `kroki/1.3.0`, `mermaid-cli/10.0`). |
| `ess:URI` | URI | Optional | A URL pointing to the canonical source repository, permalink, or editor state. |
| `ess:Timestamp` | Date | Optional | ISO-8601 UTC timestamp of generation (e.g., `2026-08-11T12:00:00Z`). |

### 3.3 Format Identifiers (`ess:Format`)
To ensure complete interoperability with Markdown editors (GitHub, Obsidian, Docusaurus, VS Code), `ess:Format` **MUST** use lowercase Markdown code block identifiers:
* **Diagramming**: `mermaid`, `plantuml`, `excalidraw`, `diagramsnet` / `drawio`, `ditaa`, `graphviz` / `dot`, `c4plantuml`, `structurizr`, `nomnoml`, `umlet`, `svgbob`, `pikchr`, `erd`, `dbml`, `wavedrom`, `bytefield`
* **Data Visualization**: `vega`, `vegalite`
* **CAD / 3D**: `openscad`, `asymptote`
* **Generative**: `prompt`, `typst`, `latex`
---
## 4. XMP Packet Structure
An ESS-compliant asset contains a standard XMP packet. Below is a complete RDF/XML packet embedding a Mermaid diagram source payload:
```xml
<?xpacket begin="﻿" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/" x:xmptk="ESS Metadata Engine v1.1">
  <rdf:RDF xmlns:rdf="[http://www.w3.org/1999/02/22-rdf-syntax-ns#](http://www.w3.org/1999/02/22-rdf-syntax-ns#)">
    <rdf:Description rdf:about=""
        xmlns:ess="[http://ns.embedded-source.org/1.0/](http://ns.embedded-source.org/1.0/)">
      <ess:Format>mermaid</ess:Format>
      <ess:Payload>H4sIAAAAAAAA/8vMLcgvKlGwVVDQ0AASaZl5KQrRBkZcEK6hEZYIUAkAQj881jMAAAA=</ess:Payload>
      <ess:Encoding>gzip+base64</ess:Encoding>
      <ess:Tool>kroki/1.3.0</ess:Tool>
      <ess:Timestamp>2026-08-11T12:00:00Z</ess:Timestamp>
    </rdf:Description>
  </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>


## 5. Extraction & Round-Trip Workflows
CLI Query & Restoration (exiftool)
Inspect asset metadata format:  
```bash
exiftool -ess:Format architecture.png
# Output: Format : mermaid
```

Extract and decompress source markup directly back to disk:  

```bash
exiftool -b -ess:Payload architecture.png | base64 -d | gunzip > architecture.mmd
```

### Automated Markdown Restoration
Implementations can parse assets and recreate original code blocks programmatically:  
```python
import subprocess, base64, gzip

def extract_ess_to_markdown(image_path):
    fmt = subprocess.check_output(["exiftool", "-s3", "-ess:Format", image_path]).decode().strip()
    payload_b64 = subprocess.check_output(["exiftool", "-b", "-ess:Payload", image_path])
    
    raw_source = gzip.decompress(base64.b64decode(payload_b64)).decode("utf-8")
    return f"```{fmt}\n{raw_source}\n```"
```

## 6. Reference Implementation: Kroki Gateway Integration

To integrate ESS into Kroki without modifying individual rendering container engines, intercept the output stream within the Java/Vert.x Gateway layer.  
Architecture Interception Point
Request Entry: io.kroki.server.service.DiagramHandler receives the request and extracts the source payload and diagram type.  
Asynchronous Closure: Intercept the returned Vert.x Buffer inside diagramService.convert().onSuccess(...) before it reaches DiagramResponse:  

```java
// Interception pattern inside DiagramHandler.java
diagramService.convert(source, format, options)
    .onSuccess(renderedBuffer -> {
        // Construct the XMP packet using captured GET/POST URL payload
        String xmp = EssMetadataBuilder.build(diagramType, source, "gzip+base64");
        
        // Inject metadata into the byte buffer
        Buffer finalBuffer = FormatInjector.inject(renderedBuffer, format, xmp);
        
        // Return response
        DiagramResponse.end(response, source, format, finalBuffer, options);
    });
```

3. Advantages:
Zero Container Changes: Requires no edits to Mermaid, PlantUML, or Excalidraw microservices.  
Zero Re-encoding Overhead: Leverages Kroki's existing GET request gzip+base64 path parameters directly for ess:Payload.  

## 7. Security, Governance & Roadmap
Security Considerations
Decompression Limits: Consumers MUST enforce maximum decompressed buffer limits on gzip+base64 streams to mitigate zip bomb risks.  
Sanitization: Extracted source text is raw markup; rendering applications MUST sanitize payloads prior to execution.  

