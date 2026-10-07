# Demonstração — Hash_ID

Autora: Shirley Maciel Dias  
Projeto individual — Insper Sec  
Repositório: https://github.com/shirleydias/projeto-sec

## 1. Projeto, resultados e conclusões

### Objetivo

O Hash_ID é uma ferramenta de terminal em Python que sugere algoritmos compatíveis com uma entrada, analisando prefixos, comprimento e conjunto de caracteres. Cada candidato tem nome, confiança e justificativa. A ferramenta não descriptografa hashes nem recupera senhas.

O trabalho estende a base didática do projeto Hash_ID do repositório [gitinspersec/Projetos](https://github.com/gitinspersec/Projetos). Os arquivos e créditos originais foram preservados. As extensões foram desenvolvidas com apoio de IA para explicações, implementação e revisão.

### Desafios implementados

| Desafio | Implementação | Demonstração |
| --- | --- | --- |
| 1.1 — Prefixo | Regra `$sha1$` para SHA-1 crypt | Identificação por prefixo com confiança `high` |
| 1.2 — Comprimento hexadecimal | Regra de 24 caracteres para candidatos BLAKE2b-96 e BLAKE2s-96 | Lista dos dois candidatos possíveis |
| 1.3 — JSON | Opção `--json`, compatível com `--top` | Saída estruturada, incluindo `[]` para entrada desconhecida |
| 2.1 — Arquivo/stdin | Opção `--file` e leitura por pipe | Processamento de várias linhas; linhas vazias ignoradas |
| 2.2 — Hashcat | Campo opcional `hashcat_mode`, coluna na tabela e dica de comando | Modos cadastrados: MD5, SHA-1, SHA-256 e bcrypt |
| 2.3 — Outras entradas | Reconhecimento de prefixos HTTP/HTTPS como URL | URL identificada sem modo de hashcat |

### Ambiente e execução

Desenvolvimento local no Windows, com PowerShell, Python 3.14.7, uv e just. Todos os exemplos usam entradas de demonstração no próprio ambiente, sem acessar sistemas de terceiros. A classificação de URL não faz requisições de rede.

Comando de instalação das dependências, executado na pasta do projeto:

```powershell
uv sync --locked --all-extras --python 3.14
```

Comandos para reprodução dos exemplos:

```powershell
# Identificação por comprimento: MD5 é um candidato, não uma certeza.
just run -- 5f4dcc3b5aa765d61d8327deb882cf99

# Desafio 1.1: identificação pelo prefixo.
uv run hashid '$sha1$40000$jtNX3nZ2$hBNaIXkt4wBI2o5rsi8KejSjNqIq'

# Desafio 1.2: 24 caracteres hexadecimais = 96 bits.
uv run hashid 13e1441308dfcaac20adf1f6

# Desafios 1.3 e 2.2: JSON com modo 0 para o candidato MD5.
uv run hashid --json --top 1 5f4dcc3b5aa765d61d8327deb882cf99

# Desafio 2.1: criação de um arquivo de demonstração e leitura.
@('5f4dcc3b5aa765d61d8327deb882cf99', '13e1441308dfcaac20adf1f6') | Set-Content -Encoding utf8 hashes-demo.txt
uv run hashid --file hashes-demo.txt --json --top 1
Get-Content .\hashes-demo.txt | uv run hashid --json --top 1

# Desafio 2.3: URL, sem dica de quebra.
uv run hashid 'https://example.com/caminho?item=1'
```

Hashes contendo `$` ficam entre aspas simples para impedir que o PowerShell interprete partes da entrada como variáveis.

No JSON de uma entrada individual, o resultado é uma lista de candidatos. Para arquivo ou stdin, o resultado é uma lista de registros com `input` e `candidates`, preservando a associação entre cada entrada e sua análise.

### Validação

- A suíte local apresentou **63 testes passando** no Windows com Python 3.14.7.
- O Ruff local apresentou **All checks passed!** após as correções de formatação.
- Pylint e Mypy foram bloqueados pelo Controle de Aplicativo do Windows. A proteção foi mantida e a validação completa foi executada em Linux no GitHub Actions.
- A execução do workflow foi conferida com status verde na interface do GitHub. O workflow configurado executa `just test`, `just lint` e uma demonstração de `just run`, com Python 3.14. A receita de lint inclui Ruff, Pylint e `mypy --strict`.

**Evidência:** [execução da validação no GitHub Actions](https://github.com/shirleydias/projeto-sec/actions/runs/37544690645/job/112545754421).

O repositório é privado; os avaliadores precisam ter acesso para consultar os arquivos e os logs.

Os códigos de saída foram conferidos no PowerShell com `$LASTEXITCODE`:

| Código | Significado | Caso validado |
| --- | --- | --- |
| `0` | Entrada reconhecida | Hash de demonstração com candidato MD5 |
| `1` | Nenhum candidato encontrado | `isso nao e um hash`, com JSON `[]` |
| `2` | Uso ou entrada inválida | `--top 0` |

Uma URL reconhecida também retorna `0`: esse código indica reconhecimento, não que a entrada necessariamente seja um hash. Em lotes, a ferramenta retorna `1` se alguma linha não tiver candidatos, sem deixar de analisar as demais.

### Decisões técnicas

1. **Separar identificação de entrada e saída.** `_identify_by_format` aplica as regras. `identify` acrescenta o modo conhecido do hashcat usando `dataclasses.replace`. A leitura de arquivos, o terminal e o JSON ficam em outras funções. Isso permite testar a identificação sem depender do terminal.
2. **Representar resultados com uma dataclass imutável.** Cada candidato possui campos explícitos. `asdict` permite incluir esses campos no JSON. O modo desconhecido é `None`; MD5 usa `0`, portanto a verificação correta é `is not None`.
3. **Adaptar a receita `run` para Python.** A receita original dependia de Bash e `cygpath` no Windows. A versão adaptada usa Python para encaminhar os argumentos e preservar o código de saída do programa.

### Limitações

- O formato não prova qual algoritmo gerou uma entrada. MD5, NTLM e MD4, por exemplo, podem aparecer como 32 caracteres hexadecimais.
- Os níveis `high`, `medium` e `low` expressam a força das regras e a prioridade dos candidatos; não são probabilidades estatísticas medidas.
- Reconhecer um prefixo não valida integralmente o hash. Da mesma forma, a regra de URL verifica apenas o início HTTP/HTTPS, sem validar o endereço completo.
- BLAKE2 permite configurar o tamanho de saída. Os candidatos BLAKE2b-96 e BLAKE2s-96 são possibilidades para 24 caracteres hexadecimais, não as únicas origens possíveis. Cada caractere hexadecimal representa 4 bits: 24 × 4 = 96.
- O dicionário de modos do hashcat contém apenas quatro algoritmos. Um `-` na tabela significa ausência de mapeamento nesta ferramenta, não ausência de suporte no hashcat.
- As heurísticas existentes para JWT e Base64 são aproximadas. Base64 é uma codificação e pode, inclusive, representar bytes de um hash.
- A entrada em lote é carregada na memória; arquivos muito grandes exigiriam processamento incremental.

### Conclusão e próximos passos

As extensões atendem aos seis desafios do MVP e mantêm os testes dos formatos existentes. O projeto mostra como regras simples de formato produzem candidatos úteis, mas também evidencia a necessidade de comunicar incerteza.

Como evolução, seria possível ampliar o catálogo de modos, validar os formatos com mais rigor e processar arquivos grandes linha a linha. Os desafios dos níveis 3 a 5 não fazem parte desta entrega.

## 2. Vídeo de demonstração

Link do vídeo: ainda não disponibilizado.
