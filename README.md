PDF Compressor — Spring Boot Application
A production-ready Spring Boot service that compresses large PDF files (e.g. 50 MB → ~6 MB) using a four-pass compression pipeline while preserving full readability.

Prerequisites
Tool	Version
Java JDK	17 or later
Apache Maven	3.8+
Quick Start
1. Clone and Build
git clone https://github.com/your-org/pdf-compressor.git
cd pdf-compressor
mvn clean package -DskipTests
2. Run as HTTP Server
java -jar target/pdf-compressor-1.0.0.jar
# Server starts at http://localhost:8080
3. Run as Command-Line Tool (single file)
java -jar target/pdf-compressor-1.0.0.jar /path/to/large.pdf /path/to/output.pdf
The application detects two positional arguments, runs the compression, prints a summary table, then exits.

HTTP API
POST /api/compress — Download compressed PDF
curl -X POST http://localhost:8080/api/compress \
     -F "file=@large.pdf" \
     -o compressed.pdf
Response headers:

X-Original-Size-Bytes — input file size
X-Compressed-Size-Bytes — output file size
X-Compression-Ratio — percentage saved (e.g. 88.0)
Content-Disposition — attachment; filename="large_compressed.pdf"
POST /api/compress/info — Get compression stats as JSON
curl -X POST http://localhost:8080/api/compress/info \
     -F "file=@large.pdf"
Example response:

{
  "outputPath": "/tmp/pdf-compressor-http/abc123_compressed.pdf",
  "originalSizeBytes": 52428800,
  "compressedSizeBytes": 6291456,
  "compressionRatioPercent": 88.0,
  "pageCount": 42,
  "imagesResampled": 18,
  "originalSizeHuman": "50.0 MB",
  "compressedSizeHuman": "6.0 MB"
}
Configuration
All properties can be overridden in application.properties or as environment variables.

Property	Default	Description
pdf.compression.image-dpi	150	Target DPI for downsampled images
pdf.compression.image-quality	0.65	JPEG quality factor (0.1–1.0)
pdf.compression.max-image-width	1600	Max image width in pixels
pdf.compression.max-image-height	1600	Max image height in pixels
pdf.compression.temp-dir	${java.io.tmpdir}/pdf-compressor	Working directory for temp files
spring.servlet.multipart.max-file-size	100MB	Max HTTP upload size
server.port	8080	HTTP server port
Example: high-quality mode

pdf.compression.image-quality=0.85
pdf.compression.max-image-width=2400
pdf.compression.max-image-height=2400
Example: aggressive compression

pdf.compression.image-quality=0.50
pdf.compression.max-image-width=1024
pdf.compression.max-image-height=1024
Running Tests
# All tests
mvn test

# Unit tests only
mvn test -pl . -Dtest="PdfCompressionServiceTest"

# Integration tests only
mvn test -Dtest="CompressionIntegrationTest"

# Controller slice tests
mvn test -Dtest="PdfCompressionControllerTest"

Test Coverage Summary
Test Class	Tests	What's Validated
PdfCompressionServiceTest	16	Valid/invalid inputs, size reduction, page count, directory creation, CompressionResult maths
PdfCompressionControllerTest	8	HTTP 200/400/500 codes, JSON bodies, response headers
CompressionIntegrationTest	8	Real PDFBox pipeline, structural integrity, page count preservation, idempotency, quality tiers
Compression Pipeline
Input PDF
    │
    ▼  Pass 1 — Image Downsampling
    │  • Extract each embedded image XObject
    │  • Resize to max 1600×1600 px (proportional)
    │  • Re-encode as JPEG @ 65% quality via Thumbnailator
    │  • Replace XObject in-place
    │
    ▼  Pass 2 — Stream Compression
    │  • Iterate all COSStream objects
    │  • Re-write with FLATE (zlib) encoding
    │  • Skip streams already FLATE-compressed
    │
    ▼  Pass 3 — Metadata Pruning
    │  • Remove XMP metadata stream (can be 100s of KB)
    │  • Clear document-info dictionary (keep Title + Author)
    │
    ▼  Pass 4 — Object-Stream Save
       • PDFBox full-rewrite mode (no appended updates)
       • Cross-reference stream replaces classic xref table
       • Removes all object duplication from incremental saves

Output PDF
Why This Achieves ~88% Reduction
For a typical 50 MB PDF, the size breakdown is roughly:

~80% high-resolution image data
~15% content streams (text, paths)
~5% overhead (xref, metadata, objects)
Pass 1 targets the dominant 80%. At 1600 px max and 65% JPEG quality, images that were stored as uncompressed bitmaps or 300 DPI PNGs shrink dramatically while remaining legible on screen.

Error Handling
Exception	HTTP Code	Trigger
InvalidPdfException	400	Non-PDF file, missing file, empty file, encrypted PDF
CompressionException	500	I/O failures, unexpected processing errors
All errors return a JSON body:

{ "code": "INVALID_PDF", "message": "Input file not found: /tmp/missing.pdf" }
Project Structure
pdf-compressor/
├── pom.xml
├── README.md
└── src/
    ├── main/
    │   ├── java/com/pdfcompressor/
    │   │   ├── PdfCompressorApplication.java      # Spring Boot entry point
    │   │   ├── config/
    │   │   │   └── CompressionProperties.java     # Typed config (image DPI, quality, etc.)
    │   │   ├── controller/
    │   │   │   └── PdfCompressionController.java  # REST endpoints
    │   │   ├── exception/
    │   │   │   ├── CompressionException.java
    │   │   │   └── InvalidPdfException.java
    │   │   ├── model/
    │   │   │   └── CompressionResult.java         # Compression metrics DTO
    │   │   └── service/
    │   │       ├── CliCompressionRunner.java      # CLI mode runner
    │   │       └── PdfCompressionService.java     # Core compression engine ⬅ start here
    │   └── resources/
    │       └── application.properties
    └── test/
        └── java/com/pdfcompressor/
            ├── controller/
            │   └── PdfCompressionControllerTest.java
            ├── integration/
            │   └── CompressionIntegrationTest.java
            └── service/
                └── PdfCompressionServiceTest.java
Limitations & Notes
Encrypted PDFs are rejected; decrypt first with qpdf --decrypt.
Vector graphics (SVG/PostScript paths) are not affected—they compress well via FLATE.
Password-protected files require the password to be supplied before compression.
The JPEG re-encoding in Pass 1 is lossy. For print-quality workflows, raise image-quality to 0.85+.
Fonts are not subset in this version (PDFBox font subsetting has limited API support); a Ghostscript post-processing step can achieve further reduction if needed.
Optional Ghostscript Post-Processing
For maximum compression, pipe the output through Ghostscript:

gs -sDEVICE=pdfwrite -dCompatibilityLevel=1.5 \
   -dPDFSETTINGS=/ebook \
   -dNOPAUSE -dQUIET -dBATCH \
   -sOutputFile=final.pdf \
   compressed.pdf
/ebook targets 150 DPI and matches the default settings in this application. /screen (72 DPI) achieves maximum compression; /printer (300 DPI) is for print workflows.
