# my-first-repo

```python
import os
import sys
import base64
import zlib

START_MARKER = "<<<FILE_START>>>\n"
END_MARKER = "<<<FILE_END>>>\n"
PATH_PREFIX = "PATH: "

IGNORE_DIRS = {
    ".git",
    "__pycache__",
    "node_modules",
    "venv",
    "env",
    ".idea",
    ".vscode"
}

IGNORE_EXTS = {
    ".png", ".jpg", ".jpeg", ".gif", ".ico",
    ".pdf", ".exe", ".dll", ".so", ".pyc",
    ".zip", ".tar", ".gz"
}


def is_binary(filepath):
    """Detect binary files."""
    try:
        with open(filepath, "rb") as f:
            chunk = f.read(1024)
            if b"\0" in chunk:
                return True
    except:
        return True
    return False


# ========================================
# PACK REPO
# ========================================

def repo_to_txt(repo_path, output_file):

    raw_data = []

    for root, dirs, files in os.walk(repo_path):

        dirs[:] = [d for d in dirs if d not in IGNORE_DIRS]

        for file in files:

            ext = os.path.splitext(file)[1].lower()
            if ext in IGNORE_EXTS:
                continue

            filepath = os.path.join(root, file)
            rel_path = os.path.relpath(filepath, repo_path)

            if is_binary(filepath):
                print(f"Skipping binary: {rel_path}")
                continue

            try:

                raw_data.append(START_MARKER)
                raw_data.append(f"{PATH_PREFIX}{rel_path}\n")

                with open(filepath, "r", encoding="utf-8", errors="ignore") as infile:
                    raw_data.append(infile.read())

                raw_data.append(END_MARKER)

                print(f"Packed: {rel_path}")

            except Exception as e:
                print(f"Error reading {rel_path}: {e}")

    raw_text = "".join(raw_data)

    compressed = zlib.compress(raw_text.encode("utf-8"), level=9)

    encoded = base64.b64encode(compressed).decode("utf-8")

    with open(output_file, "w", encoding="utf-8") as f:
        f.write(encoded)

    print(f"\nRepository packed into compressed base64 TXT: {output_file}")


# ========================================
# SAFE PATH HANDLING
# ========================================

def safe_join(base_dir, rel_path):

    full_path = os.path.abspath(os.path.join(base_dir, rel_path))
    base_dir = os.path.abspath(base_dir)

    if not full_path.startswith(base_dir):
        raise Exception("Path traversal detected")

    return full_path


# ========================================
# UNPACK REPO
# ========================================

def txt_to_repo(txt_file, output_dir):

    os.makedirs(output_dir, exist_ok=True)

    with open(txt_file, "r", encoding="utf-8") as f:
        encoded = f.read()

    compressed = base64.b64decode(encoded)

    raw_text = zlib.decompress(compressed).decode("utf-8")

    lines = raw_text.splitlines(keepends=True)

    outfile = None
    current_file = None

    for line in lines:

        if line == START_MARKER:
            continue

        if line.startswith(PATH_PREFIX):

            rel_path = line.replace(PATH_PREFIX, "").strip()

            full_path = safe_join(output_dir, rel_path)

            os.makedirs(os.path.dirname(full_path), exist_ok=True)

            outfile = open(full_path, "w", encoding="utf-8")

            current_file = full_path

            print(f"Creating: {rel_path}")

            continue

        if line == END_MARKER:

            if outfile:
                outfile.close()
                outfile = None
                current_file = None

            continue

        if outfile:
            outfile.write(line)

    print(f"\nRepository unpacked into: {output_dir}")


# ========================================
# CLI
# ========================================

def main():

    if len(sys.argv) < 2:
        print("Usage:")
        print("  python script.py pack <repo_path> <output_file>")
        print("  python script.py unpack <txt_file> <output_dir>")
        return

    mode = sys.argv[1]

    if mode == "pack":

        repo = sys.argv[2] if len(sys.argv) > 2 else "."
        output = sys.argv[3] if len(sys.argv) > 3 else "repo_dump.txt"

        repo_to_txt(repo, output)

    elif mode == "unpack":

        txt_file = sys.argv[2] if len(sys.argv) > 2 else "repo_dump.txt"
        output_dir = sys.argv[3] if len(sys.argv) > 3 else "restored_repo"

        txt_to_repo(txt_file, output_dir)

    else:
        print("Unknown command")


if __name__ == "__main__":
    main()
```
