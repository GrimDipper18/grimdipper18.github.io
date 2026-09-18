import io
import os
import zipfile

# 1. To extract files directly from a binary zip string/bytes:
def extract_zip_bytes(zip_bytes: bytes, extract_to: str = "./playhub_extracted"):
    """Extracts a zip file represented as raw bytes into a directory."""
    with zipfile.ZipFile(io.BytesIO(zip_bytes)) as z:
        z.extractall(extract_to)
        print(f"Extracted {len(z.namelist())} files to {extract_to}")


# 2. To build a standard PlayHub server directory using Python:
def generate_playhub_structure(base_path: str = "./playhub"):
    """Creates the basic directory layout and starter files for the project."""
    files = {
        "package.json": """{
  "name": "playhub",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.2"
  }
}""",
        "server.js": """const express = require('express');
const path = require('path');
const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.static(path.join(__dirname, 'public')));

app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));""",
        "public/index.html": """<!DOCTYPE html>
<html>
<head><title>PlayHub</title></head>
<body>
  <h1>PlayHub Games</h1>
  <ul>
    <li><a href="games/pong.html">Pong</a></li>
    <li><a href="games/snake.html">Snake</a></li>
    <li><a href="games/tetris.html">Tetris</a></li>
  </ul>
</body>
</html>""",
    }

    for relative_path, content in files.items():
        full_path = os.path.join(base_path, relative_path)
        os.makedirs(os.path.dirname(full_path), exist_ok=True)
        with open(full_path, "w", encoding="utf-8") as f:
            f.write(content)

    print(f"Project structure initialized at {base_path}")


if __name__ == "__main__":
    generate_playhub_structure()
