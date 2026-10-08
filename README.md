# DocMind RAG — Regras de Monopoly (CKP02)

FIAP · Prompt Engineering and Artificial Intelligence · CC 2026 · Prof. Jorge Luiz Gomes
Checkpoint 02 — Módulo 2

## Integrantes

| Nome | RM |
|---|---|
| Julia Johanson Peniche Dias Da Silva | 572220 |
| Lucas Bomfim Leite | 570420 |
| Eduardo Barcelos De Carvalho Braziliano | 573274 |

## O que é

Pipeline RAG completo (`load → split → embed → store → retrieve → generate`) sobre as regras oficiais de Monopoly. A resposta sempre cita o arquivo e a página de origem dos trechos usados.

- **Embeddings:** `nomic-embed-text` (Ollama)
- **Modelo de chat:** `gemma4:cloud` via Ollama Cloud, `temperature=0`
- **Vector store:** ChromaDB local (`PersistentClient`), uma coleção por configuração de chunking: `monopoly_regras_cs256`, `monopoly_regras_cs512`, `monopoly_regras_cs1024`
- **Chunking:** `RecursiveCharacterTextSplitter` com `separators=["\n\n", "\n", ". ", " ", ""]` e `chunk_overlap` de 12% do `chunk_size`; comparados `chunk_size` 256, 512 e 1024
- **Avaliação:** RAGAS (`faithfulness` e `answer_relevancy`) com 7 perguntas por configuração (6 com resposta na base + 1 fora da base, para testar a recusa)
- **Diferenciais:** filtro por metadata (`where`), reranking com cross-encoder (`cross-encoder/ms-marco-MiniLM-L-6-v2`) e interface Gradio com fonte citada
- **Função modular para o CKP03:** `buscar(consulta)` (e a `@tool` `buscar_regras_monopoly`)

## Como executar

1. Abra o notebook no Google Colab.
2. Cadastre o Secret `OLLAMA_API_KEY` (ícone de chave do Colab). Nunca escreva a chave no código.
3. Rode **Runtime → Run all**. Quando a Seção 2 pedir, envie os documentos da base (PDF, TXT ou MD) para a pasta `data/`.

## Como adicionar novos documentos à base

1. Copie o arquivo (PDF, TXT ou MD) para a pasta `data/` (ou envie pelo upload da Seção 2).
2. Na Seção 2, adicione uma entrada em `DOCS_META` cuja **chave seja o nome exato do arquivo**, com os campos `titulo`, `tipo`, `categoria`, `data`, `idioma` e `url` (link de origem):

   ```python
   "doc6.pdf": {
       "titulo": "Regras — Monopoly Deal", "tipo": "manual",
       "categoria": "variantes", "data": "2020", "idioma": "pt",
       "url": "https://link-de-origem/doc6.pdf"},
   ```

   Arquivos que não estiverem em `DOCS_META` recebem uma metadata padrão e geram um aviso, e a validação exige `url` em todos os documentos.
3. Rode novamente as Seções 2 a 4 (recarrega, divide e reindexa; as coleções antigas são recriadas, sem duplicar chunks).
4. Rode a Seção 9 para reavaliar o RAGAS com a nova base e confira o `chunk_size` vencedor.

## Base de conhecimento (documentos reais, com fonte)

| Arquivo | Documento | Categoria | Fonte |
|---|---|---|---|
| doc1.pdf | Regras — Monopoly Junior | variantes | https://instructions.hasbro.com/api/download/A6984_pt-br_jogo-monopoly-junior.pdf |
| doc2.pdf | Regras — Monopoly Banco Eletrônico | variantes | https://instructions.hasbro.com/api/download/A7444_pt-br_jogo-monopoly-banco-eletronico.pdf |
| doc3.pdf | Regras — Monopoly Grab and Go | variantes | https://instructions.hasbro.com/api/download/B1002_pt-br_jogo-monopoly-grab-go-game.pdf |
| doc4.pdf | Regras oficiais — Monopoly Clássico | regras_gerais | https://instructions.hasbro.com/api/download/00009_pt-br_monopoly.pdf |
| doc5.pdf | Regras — Monopoly Fortnite | variantes | https://instructions.hasbro.com/api/download/E6603_pt-br_jogo-monopoly-fortnite.pdf |

Total: 5 documentos, 22 páginas. Todos os manuais são do site oficial de instruções da Hasbro.

## Resultados RAGAS (média das 7 perguntas)

| chunk_size | n_chunks | faithfulness | answer_relevancy | faithfulness ≥ 0,7 |
|---:|---:|---:|---:|:---:|
| 256 | 315 | 0,390 | 0,692 | não |
| 512 | 160 | 0,643 | 0,508 | não |
| **1024** | 85 | **0,833** | 0,611 | **sim** |

**Configuração vencedora: `chunk_size = 1024`**, com a maior faithfulness média e a única acima da meta de 0,7. Os resultados por pergunta estão em `resultados_ragas.csv`.

**Leitura dos números:** chunks maiores mantêm no mesmo trecho as regras que têm várias condições (hipoteca, construção, leilão), o que ajuda o modelo a responder só com o que está no contexto. Com 256 caracteres, as regras ficam fragmentadas e a faithfulness cai para 0,39. A pior pergunta na configuração vencedora foi "Quantas casas são necessárias antes de construir um hotel?" (faithfulness 0,33); os trechos recuperados trazem a regra espalhada em mais de um documento, o que aponta para um problema de retrieval e não de prompt.

## Estrutura do notebook

1. Instalação e configuração dos modelos
2. Base de conhecimento (upload + `DOCS_META`)
3. Split — estratégias de chunking
4. Embed + Store — ChromaDB
5. Retrieve — `buscar()` com filtro de metadata e reranking
6. Generate — resposta com citação da fonte
7. Diferencial: metadata filtering
8. Diferencial: reranking (com × sem cross-encoder)
9. Avaliação RAGAS
10. Conclusão do chunking vencedor
11. Função final para o CKP03 (`@tool`)
12. Diferencial: interface Gradio
13. Geração deste README
