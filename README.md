# TCC - Estudo de Caso: Tratamento Estatístico e Visualização

Este repositório contém a análise estatística e scripts para o processamento das métricas coletadas (k6, PMD, ArchUnit) durante a execução do TCC. O gerenciamento de dependências e ambientes virtuais é feito com a ferramenta [uv](https://github.com/astral-sh/uv).

## Pré-requisitos

- [uv](https://github.com/astral-sh/uv) instalado no sistema.

### Instalando e fixando o Python com o uv
O próprio `uv` pode gerenciar e baixar as instalações do Python para você. Para instalar a versão 3.12.13 e fixá-la como a versão padrão apenas para este projeto, execute:

```bash
# Baixa e instala a versão específica do Python no sistema
uv python install 3.12.13

# Fixa a versão no diretório atual (isso cria/atualiza o arquivo .python-version)
uv python pin 3.12.13
```

## Configuração do Ambiente Virtual

Todo o gerenciamento de dependências é baseado no arquivo `pyproject.toml` e no `uv.lock`. Siga os passos abaixo para preparar o seu ambiente local:

### 1. Sincronizar as Dependências
Para criar o ambiente virtual (caso não exista) e instalar/sincronizar todas as bibliotecas necessárias com as versões exatas definidas no projeto, execute:
```bash
uv sync
```
Isso fará o download e instalação de pacotes como `numpy`, `pandas`, `scipy`, `matplotlib`, `seaborn` e `openpyxl` diretamente na pasta `.venv` do projeto.

## Utilizando Jupyter Notebook

O projeto já inclui a biblioteca `ipykernel` instalada. Para que o seu editor (como o VS Code, JupyterLab, etc.) reconheça o ambiente virtual do `uv` de forma nativa como um Kernel Jupyter, execute o seguinte comando:

### Registrar o Kernel Jupyter
```bash
uv run ipython kernel install --user --name=tcc-estudo-de-caso
```
Isso criará um kernel chamado **tcc-estudo-de-caso** que você pode selecionar ao abrir o notebook `analise_estatistica_tcc.ipynb`. Ele utilizará as bibliotecas isoladas do ambiente local.

### Ativando o ambiente e selecionando o Interpretador/Kernel (VS Code / Antigravity)

Caso a sua IDE não reconheça automaticamente as dependências ou o kernel, siga os passos:

1. **Ative o ambiente virtual no terminal:**
   ```bash
   source .venv/bin/activate
   ```
2. **Selecione o interpretador Python na IDE:**
   - Abra a Paleta de Comandos (`Ctrl+Shift+P` ou `Cmd+Shift+P`).
   - Selecione `Python: Select Interpreter`.
   - Na lista, procure por algo como `Python 3.12.13 ('.venv': venv)`.
   - Se não aparecer na lista, clique em `Enter interpreter path...` (Inserir caminho do interpretador) e digite `./.venv/bin/python`.
3. Ao abrir o notebook `.ipynb`, certifique-se de escolher o kernel **tcc-estudo-de-caso** no canto superior direito.

### Abrir o Jupyter Lab / Notebook via Terminal (Opcional)

> **Recomendação VS Code:** É altamente recomendado utilizar o próprio editor para visualizar e executar os notebooks (`.ipynb`). Para a melhor experiência, sugere-se a instalação das seguintes extensões:
> - **Jupyter** (`ms-toolsai.jupyter`): Extensão oficial da Microsoft para suporte completo a notebooks.
> - **Office File Preview** (`takashi-uchida.office-file-preview`): Excelente para visualizar rapidamente as tabelas consolidadas geradas em `.xlsx` sem precisar sair da IDE.

Caso prefira abrir o Jupyter clássico ou o Lab diretamente pelo seu navegador via terminal, você pode rodá-lo utilizando o comando abaixo (que garante a execução através do ambiente virtual):
```bash
uv run --with jupyter jupyter lab
```
*ou para o Notebook clássico:*
```bash
uv run --with jupyter jupyter notebook
```

## Adicionando novas dependências

Para adicionar um novo pacote ao projeto, sempre utilize o `uv` em vez do `pip` tradicional:
```bash
uv add nome-do-pacote
```
Isso adicionará a dependência automaticamente no `pyproject.toml` e atualizará o `uv.lock`.
