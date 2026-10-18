# Guia de Padronização e Instruções para Agentes e Contribuidores

Este documento reúne todas as diretrizes técnicas, decisões de arquitetura e scripts automatizados desenvolvidos para organizar, catalogar e exibir novos lotes de livros técnicos no repositório **EscarizDePobre**, garantindo fidelidade visual e consistência na renderização (especialmente no Markdown Preview do VS Code, GitHub e navegadores baseados em Chromium).

---

## 1. Visão Geral da Estrutura

Ao adicionar uma nova coleção ou lote pessoal de livros (ex: `Livros-guilherme/`):
1. **Pasta dedicada**: Mantenha os arquivos PDF organizados em uma pasta própria no root do projeto.
2. **Catálogo local (`README.md`)**: Crie um arquivo `README.md` dentro da pasta com a listagem agrupada por tópicos/categorias.
3. **Capas em SVG (`.github/img/`)**: Extraia a primeira página de cada PDF como capa em alta resolução e empacote-a em formato SVG compatível com o padrão do repositório.
4. **Atualização do `README.md` principal**:
   - Inclua a nova seção no sumário inicial com âncora HTML.
   - Crie a seção temática com as tabelas padronizadas (`LIVRO | NOME | AUTOR | STATUS | IDIOMA`).
5. **Caminhos relativos**: Utilize sempre caminhos relativos locais (`.github/img/...` e `Livros-.../...`) codificados via URL encoding (`%20` para espaços, etc.).

---

## 2. Requisito Crítico: Estrutura do SVG das Capas

### ⚠️ O Problema de Renderização no VS Code / Chromium
Quando uma imagem SVG é embutida em Markdown através de sintaxe de imagem (`![capa](caminho.svg)`):
- O renderizador Chromium executa em modo **SVG-as-image** restrito por questões de segurança (CSP).
- Elementos `<image href="data:image/jpeg;base64,...">` diretos **falham e exibem ícone de imagem quebrada**.
- O renderizador exige o namespace `xmlns:xlink="http://www.w3.org/1999/xlink"` e o padrão de preenchimento por `<pattern>` adotado pelo Figma.

### ✅ O Formato SVG Obrigatório
Todo SVG de capa gerado deve seguir rigorosamente a estrutura abaixo:

```xml
<svg width="65" height="92" viewBox="0 0 65 92" fill="none" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
  <rect width="65" height="92" fill="url(#pattern0)"/>
  <defs>
    <pattern id="pattern0" patternContentUnits="objectBoundingBox" width="1" height="1">
      <use xlink:href="#image0_cov" transform="scale(0.00285714 0.002)"/>
    </pattern>
    <image id="image0_cov" width="350" height="500" xlink:href="data:image/jpeg;base64,{BASE64_JPEG_DATA}"/>
  </defs>
</svg>
```

**Parâmetros de proporção:**
- Dimensões do viewBox: `65` de largura por `92` de altura (proporção clássica de livro).
- Resolução da imagem base64: `350x500` pixels.
- Fatores da matriz de escala do pattern:
  - `scale_x = 1 / 350 = 0.00285714`
  - `scale_y = 1 / 500 = 0.002`

---

## 3. Script Automatizado para Extração e Geração de Capas

A ferramenta nativa do Linux `pdftoppm` (presente no pacote `poppler-utils`) é utilizada para extrair a primeira página do PDF com alta performance e sem dependências pesadas de bibliotecas de terceiros.

### Script Python (`extract_covers.py`):
Execute o script abaixo a partir da raiz do repositório para processar qualquer diretório de livros:

```python
import os
import glob
import subprocess
import tempfile
import base64
import urllib.parse
import re

# Diretório de entrada dos PDFs e saída das capas
PDF_DIR = "Livros-guilherme"
IMG_DIR = ".github/img"
os.makedirs(IMG_DIR, exist_ok=True)

# Lista de SVGs já existentes no projeto original que não devem ser sobrescritos
KEEP_ORIGINAL = {
    "Arquitetura Limpa - O Guia do Artesão para Estrutura e Design de Software.svg",
    "Padrões de Projetos - Soluções Reutilizáveis de Software Orientados a Objetos.svg",
    "Refatoração - Aperfeiçoando o Design de Códigos Existentes.svg"
}

pdfs = glob.glob(os.path.join(PDF_DIR, "*.pdf"))

for pdf_path in sorted(pdfs):
    base_name = os.path.basename(pdf_path)
    
    # Define o nome amigável do SVG (ou use o nome mapeado no README)
    svg_name = os.path.splitext(base_name)[0] + ".svg"
    
    if svg_name in KEEP_ORIGINAL:
        print(f"[SKIP] Mantendo original: {svg_name}")
        continue

    svg_path = os.path.join(IMG_DIR, svg_name)

    with tempfile.TemporaryDirectory() as tmpdir:
        prefix = os.path.join(tmpdir, "cover")
        # Extrai a página 1 escalada para 350x500 em JPEG
        cmd = [
            "pdftoppm", "-jpeg", "-r", "150",
            "-scale-to-x", "350", "-scale-to-y", "500",
            "-f", "1", "-l", "1",
            pdf_path, prefix
        ]
        res = subprocess.run(cmd, capture_output=True)
        if res.returncode != 0:
            print(f"[ERROR] Falha ao extrair {pdf_path}: {res.stderr.decode()[:100]}")
            continue

        jpgs = glob.glob(os.path.join(tmpdir, "*.jpg"))
        if not jpgs:
            continue

        with open(jpgs[0], "rb") as jf:
            b64_data = base64.b64encode(jf.read()).decode("ascii")

        svg_content = f'''<svg width="65" height="92" viewBox="0 0 65 92" fill="none" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
<rect width="65" height="92" fill="url(#pattern0)"/>
<defs>
<pattern id="pattern0" patternContentUnits="objectBoundingBox" width="1" height="1">
<use xlink:href="#image0_cov" transform="scale(0.00285714 0.002)"/>
</pattern>
<image id="image0_cov" width="350" height="500" xlink:href="data:image/jpeg;base64,{b64_data}"/>
</defs>
</svg>
'''
        with open(svg_path, "w", encoding="utf-8") as out:
            out.write(svg_content)
        print(f"[OK] Gerado com sucesso: {svg_path}")
```

---

## 4. Padrão da Tabela no `README.md`

Toda listagem de livros na página principal deve seguir o formato de tabela Markdown:

```markdown
| LIVRO | NOME | AUTOR | STATUS | IDIOMA |
| - | - | - | - | - |
| [![capa do livro](.github/img/Nome%20do%20Livro.svg)](Livros-guilherme/Nome%20do%20Livro.pdf) | [Nome do Livro](Livros-guilherme/Nome%20do%20Livro.pdf) | Nome do Autor | ![STATUS](https://user-images.githubusercontent.com/98864503/232176474-b32dd53d-caeb-4053-9189-d8d07171d5d7.png) | ![IDIOMA](https://user-images.githubusercontent.com/98864503/232176386-56cfbf3a-27b5-40e6-bf86-892f8645cbc3.png) |
```

### Badges Padrão:
- **Status (Livro no Repositório)**:
  `![STATUS](https://user-images.githubusercontent.com/98864503/232176474-b32dd53d-caeb-4053-9189-d8d07171d5d7.png)`
- **Idioma Português (Brasil)**:
  `![IDIOMA](https://user-images.githubusercontent.com/98864503/232176253-313aac7b-b0c8-49f3-b234-10eeba6a2077.png)`
- **Idioma Inglês (EUA)**:
  `![IDIOMA](https://user-images.githubusercontent.com/98864503/232176386-56cfbf3a-27b5-40e6-bf86-892f8645cbc3.png)`

---

## 5. Cuidados com Links Relativos e Cache

1. **Evitar links remotos do GitHub (`https://github.com/.../blob/...`)**:
   - Sempre utilize links relativos como `.github/img/Nome.svg`. Links remotos podem quebrar quando o repositório for clonado para outro usuário/organização ou renomeado.
2. **URL Encoding**:
   - Nomes de arquivos com caracteres de espaço, parênteses ou vírgula precisam ser codificados nas URLs do Markdown (`%20` para espaço, `%28` e `%29` para parênteses, `%2C` para vírgulas).
3. **Cache de visualização no VS Code**:
   - Caso o Markdown Preview continue mostrando ícones quebrados após gerar os SVGs, feche a aba de preview e reabra com `Ctrl + Shift + V` para forçar o recarregamento do cache webview do VS Code.

---

## 6. Checklist Rápido para Próximas Adições

- [ ] Criar pasta com os PDFs (ex: `Livros-<nome>/`).
- [ ] Criar `Livros-<nome>/README.md` com a categorização dos títulos.
- [ ] Verificar se os livros já existem no acervo original para reaproveitar a capa se aplicável.
- [ ] Executar script de extração com `pdftoppm` gerando SVGs com a estrutura `<pattern>` + `xlink:href`.
- [ ] Inserir nova âncora no Sumário do `README.md`.
- [ ] Adicionar tabelas organizadas por áreas de conhecimento no `README.md`.
- [ ] Validar a integridade de todos os links com script de checagem.
- [ ] Atualizar instruções ou documentação se novos tipos de mídia/badges forem introduzidos.
