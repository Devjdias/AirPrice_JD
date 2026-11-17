# ✈️ AirPrice Scraper

O AirPrice Scraper é um sistema em Python para coleta automatizada de preços de passagens da LATAM, com interface gráfica, exportação local em Excel e envio opcional para Google Sheets. O código é distribuído no repositório `AirPrice_JD`.

O projeto apoia análises acadêmicas de preços por antecedência de compra. Cada consulta busca uma passagem de ida, para um adulto, em classe econômica, e registra o menor preço válido entre os cards carregados na página.

## ✨ Funcionalidades

- Coleta manual com seleção de rotas, data do voo, repetições e modo de privacidade.
- Agendador com ciclos de coleta, janelas de antecedência, intervalo e variação aleatória do intervalo, chamada de jitter.
- Script de coleta padronizada para as janelas de 7, 15, 30 e 90 dias.
- Exportação `.xlsx` por experimento e histórico de arquivos na interface.
- Envio automático e reenvio de arquivos históricos ao Google Sheets.
- Registro operacional das chamadas ao scraper em `LOGS/tentativas_coleta.csv`.
- Capturas de tela e HTML compactado opcionais para diagnóstico.
- Perfis limpos, perfil persistente, modo incógnito e proxy opcional.

## 🧩 Pré-requisitos

- Python 3.10 ou superior, com `tkinter` disponível.
- Google Chrome instalado.
- Sessão gráfica disponível para abrir o navegador e, quando utilizada, a interface.
- Acesso à internet, inclusive para o gerenciamento do ChromeDriver pelo Selenium.
- Conta de serviço do Google e planilha compartilhada com essa conta, caso utilize Google Sheets.

A opção **Ocultar janela do Chrome** inicia um Chrome comum minimizado e posicionado fora da tela, com `headless=True` no código. Essa execução também utiliza uma sessão gráfica. A forma como a janela aparece depende do sistema operacional.

## 🚀 Instalação

Baixe o código pelo GitHub, usando `Code > Download ZIP`, ou clone o repositório.

```powershell
git clone https://github.com/Devjdias/AirPrice_JD.git
cd AirPrice_JD
```

Execute os comandos a partir da raiz do projeto. No Windows com PowerShell, crie e ative um ambiente virtual e instale as dependências fixadas em `config/requirements.txt`.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r config\requirements.txt
```

Se a política do PowerShell impedir a ativação, use diretamente o interpretador do ambiente virtual para instalar as dependências e executar os scripts.

```powershell
.\.venv\Scripts\python.exe -m pip install -r config\requirements.txt
.\.venv\Scripts\python.exe src\scraper_passagens.py
```

No Linux/macOS, instale previamente o Chrome e o suporte a `tkinter` da sua distribuição Python. Em Ubuntu/Debian, esse suporte costuma ser fornecido pelo pacote `python3-tk`.

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r config/requirements.txt
```

As dependências incluem Selenium, pandas, openpyxl, tkcalendar, gspread e google-auth. Os módulos da interface também são importados pelo script de console, portanto a instalação completa é necessária para as duas formas de execução.

## 📦 Configuração inicial

O programa lê `config/config.ini`. Para criar esse arquivo no primeiro uso, copie o modelo público. O comando abaixo preserva uma configuração local já existente.

```powershell
if (-not (Test-Path config\config.ini)) {
    Copy-Item config\config.example.ini config\config.ini
}
```

No Linux/macOS, execute o equivalente.

```bash
test -f config/config.ini || cp config/config.example.ini config/config.ini
```

O modelo contém os valores iniciais usados para uma coleta local.

```ini
[GSHEETS]
arquivo_credenciais =
id_planilha =
nome_aba = Dados

[ANTIDETECT]
humanize = True
save_html = False
screenshot = False

[PROXY]
enabled = False
list =

[CHROME]
path =
```

| Opção | Comportamento |
| --- | --- |
| `arquivo_credenciais` | Caminho do JSON da conta de serviço. Caminhos relativos são interpretados a partir da pasta em que o programa foi iniciado. |
| `id_planilha` | ID da planilha que receberá os dados. O envio automático depende desse campo e do caminho da credencial preenchidos. |
| `nome_aba` | Nome da aba de destino. O modelo usa `Dados`. |
| `humanize` | Habilita as pausas variáveis controladas por essa opção. Algumas esperas de carregamento e entre rotas também são aplicadas pelo fluxo de execução. |
| `save_html` | Habilita HTML `.html.gz` transitório, sujeito à limpeza automática descrita abaixo. |
| `screenshot` | Habilita capturas quando uma tentativa retorna sem dados ou ocorre erro, desde que a sessão do navegador esteja acessível. |
| `PROXY.enabled` e `PROXY.list` | Habilitam a escolha aleatória de um proxy por experimento. A lista aceita uma entrada por linha e ignora linhas iniciadas por `#`. |
| `CHROME.path` | Caminho do executável do Chrome. Vazio habilita a detecção automática. Essa opção é editada diretamente no INI. |

Para usar Google Sheets, coloque o JSON da conta de serviço em `credentials/`, preencha os campos de `[GSHEETS]` e compartilhe a planilha com o e-mail dessa conta, concedendo permissão de edição. Um caminho absoluto para o JSON permite iniciar o programa de outras pastas. A interface oferece seleção do arquivo e edição dessas configurações.

No Windows, a detecção do Chrome consulta os diretórios usuais de instalação e o registro. No Linux/macOS, consulta nomes de executáveis no `PATH` e o caminho padrão do aplicativo no macOS. Se necessário, informe o caminho completo em `[CHROME]`.

## ▶️ Execução

### Interface gráfica

Com o ambiente virtual ativado, execute o ponto de entrada da interface.

```powershell
python src/scraper_passagens.py
```

Na página **Coleta Manual**, selecione as rotas, escolha a data do voo e ajuste as repetições. O valor inicial é de três repetições por rota. Cada repetição pode gerar um registro com o menor preço encontrado.

Na página **Agendador**, selecione rotas e janelas de antecedência. A configuração inicial usa todas as rotas, as janelas de 7, 15, 30 e 90 dias, uma repetição por combinação e uma espera de 60 minutos com jitter de até 8 minutos para cada lado. Essa espera começa depois que o ciclo termina, somando-se à duração das consultas. O programa precisa permanecer aberto para executar os ciclos.

A página **Histórico & Exportação** lista os arquivos `.xlsx` de `DADOS_BRUTOS/` e permite enviá-los ao Google Sheets. Cada envio acrescenta linhas à aba, incluindo reenvios do mesmo arquivo.

O botão **Parar** sinaliza a interrupção e solicita o encerramento do navegador ativo. As esperas e as consultas verificam esse sinal durante a execução.

### Coleta padronizada do TCC

Execute o script de console com o mesmo ambiente virtual.

```powershell
python src/coleta_tcc.py
```

O script percorre as quatro rotas cadastradas em `ROTAS_DISPONIVEIS` para as janelas `[7, 15, 30, 90]`, totalizando 16 combinações de rota e antecedência. Executa uma repetição por combinação, com `headless=True`, modo inicial `Sem cookies (perfil limpo)` e `registrar_sem_dados=True`. A data do voo é recalculada para cada janela a partir do horário local do computador. Essa execução realiza uma passagem pelas janelas e termina.

### Rotas cadastradas

| Origem | Destino |
| --- | --- |
| Brasília (`BSB`) | Rio de Janeiro/Galeão (`GIG`) |
| Porto Velho (`PVH`) | Brasília (`BSB`) |
| Brasília (`BSB`) | Florianópolis (`FLN`) |
| João Pessoa (`JPA`) | Brasília (`BSB`) |

As rotas disponíveis são definidas em `ROTAS_DISPONIVEIS`, em `src/scraper_passagens.py`. A interface permite selecionar as rotas cadastradas.

### Navegador e modos de privacidade

- **Sem cookies (perfil limpo)** cria um perfil temporário novo. Se a primeira tentativa da repetição retornar sem preço, a segunda tenta o modo `Incognito`.
- **Normal (com cookies)** reutiliza o perfil em `LOGS/chrome_profiles/normal`, mantendo cookies e histórico entre coletas.
- **Incognito** utiliza um perfil temporário e o argumento `--incognito`.

Os perfis temporários são criados na pasta temporária do sistema operacional e a rotina tenta removê-los ao encerrar o navegador. As duas opções atuais de user-agent, `Chrome Desktop (padrao)` e `Chrome Normal`, utilizam o user-agent nativo do Chrome instalado.

O scraper monitora sinais de bloqueio, inclusive HTTP 401, 403 e 429 na API de ofertas utilizada pela página. A coleta depende do carregamento do site, de seus seletores e da disponibilidade do portal.

## 📤 Dados e saídas

| Local | Conteúdo |
| --- | --- |
| `DADOS_BRUTOS/` | Excel `passagens_<experimento_id>_<timestamp>.xlsx`, com 17 colunas padronizadas. |
| `LOGS/tentativas_coleta.csv` | Resultado operacional de cada chamada concluída ao scraper, com status, HTTP observado, contagens, duração e detalhe. Utiliza UTF-8 com BOM e delimitador `;`. |
| `LOGS/` | Capturas de tela opcionais e HTML compactado transitório. |
| `LOGS/chrome_profiles/normal/` | Perfil persistente usado pelo modo normal. |
| Google Sheets | Linhas acrescentadas à aba configurada. Uma aba inexistente é criada e uma aba vazia recebe o cabeçalho. |

`DADOS_BRUTOS/`, `LOGS/` e o diretório de perfis são criados pelo programa. A coleta manual salva os resultados com preço. O agendador e o script do TCC também podem salvar uma linha com `preco_brl` vazio quando um experimento termina sem resultados, inclusive após bloqueios ou erros. Consulte o CSV de tentativas para interpretar essas ocorrências. Falhas anteriores à chamada do scraper, como a impossibilidade de iniciar o Chrome, aparecem no log da execução.

O campo `dias_antecedencia` usa a parte inteira da diferença entre a data do voo à meia-noite e o instante da consulta. Uma janela selecionada de 7 dias pode, por esse cálculo, gerar o valor 6 no arquivo.

Com `save_html=True`, o programa salva HTML `.html.gz` e calcula seu MD5. A rotina tenta apagar esses arquivos ao finalizar o experimento, conferindo o hash quando disponível. A limpeza também ocorre quando o envio ao Google Sheets falha, está desabilitado ou a coleta termina sem registros. Arquivos com hash divergente são preservados. Screenshots e o CSV de tentativas permanecem em `LOGS/`.

Se o envio ao Google Sheets falhar após a gravação do Excel, o arquivo local continua disponível para reenvio pelo histórico. O envio acrescenta linhas sem deduplicação e exige que a aba existente mantenha a ordem das colunas do dataset.

## 📂 Estrutura do projeto

```text
AirPrice_JD/
|-- config/
|   |-- config.example.ini
|   `-- requirements.txt
|-- src/
|   |-- scraper_passagens.py
|   |-- latam_scraper.py
|   |-- scrapers_utils.py
|   `-- coleta_tcc.py
|-- docs/
|   `-- DOCUMENTACAO_TECNICA.md
|-- data/
|   |-- agendador.png
|   |-- analise.xlsx
|   |-- config.png
|   |-- diagrama_sequência_UML.jpeg
|   |-- hist&expo.png
|   |-- inicio.png
|   `-- manual.png
|-- scripts/
|   `-- build_submission_zip.ps1
|-- .gitignore
|-- README.md
|-- SECURITY.md
`-- LICENSE
```

`config/config.ini`, `credentials/`, `DADOS_BRUTOS/`, `LOGS/` e `dist/` são usados localmente conforme a configuração e a execução.

| Módulo | Responsabilidade |
| --- | --- |
| `scraper_passagens.py` | Interface, configuração, processo do Chrome, conexão Selenium, orquestração, agendador, Excel e Google Sheets. |
| `latam_scraper.py` | Navegação na LATAM, diagnóstico de carregamento e bloqueios, extração da menor oferta e registro do resultado operacional. |
| `scrapers_utils.py` | Normalização de preços, montagem de registros, pausas interrompíveis, hashes, limpeza de HTML e CSV de tentativas. |
| `coleta_tcc.py` | Execução de console das rotas e janelas definidas no protocolo de coleta. |

A documentação técnica detalha a arquitetura, o fluxo de execução, as colunas dos arquivos e as limitações da implementação. [Consulte a documentação técnica](docs/DOCUMENTACAO_TECNICA.md).

## 🧱 Materiais complementares

- Planilha de análise disponibilizada com o projeto. [Abrir a planilha](data/analise.xlsx).
- Diagrama de sequência UML disponibilizado como referência visual. [Abrir o diagrama](data/diagrama_sequência_UML.jpeg).
- Capturas das páginas de início, coleta manual, agendador, histórico e configurações. [Início](data/inicio.png), [coleta manual](data/manual.png), [agendador](data/agendador.png), [histórico e exportação](<data/hist&expo.png>) e [configuração](data/config.png).
- Vídeo demonstrativo da interface. [Assistir ao vídeo](https://l1nk.dev/0YbEY).

## 🗂️ Pacote suplementar

O repositório contém o código-fonte, a documentação, a licença, o modelo público de configuração e os materiais complementares em `data/`. Para gerar o ZIP de submissão, execute o script PowerShell a partir da raiz do projeto.

```powershell
.\scripts\build_submission_zip.ps1
```

O script gera `dist/airprice-jd-acm-software.zip`, com a pasta `AirPrice_JD/` e o manifesto `PACKAGE_CONTENTS.txt`. Ele seleciona os arquivos de código, documentação e `.gitignore` definidos no script e os arquivos `.jpeg`, `.jpg`, `.png` e `.xlsx` diretamente em `data/`.

O pacote utiliza o modelo público `config/config.example.ini`. Os diretórios locais de credenciais, coletas e logs ficam fora da seleção do script. Revise os materiais públicos de `data/` antes de gerar um pacote, pois os arquivos com essas extensões são incluídos automaticamente.

## 🔒 Segurança

Mantenha credenciais JSON, configurações reais, tokens, proxies privados e dados de navegação nos diretórios locais ignorados pelo Git. O arquivo `.gitignore` também trata os resultados de coleta e os materiais públicos de `data/` de forma específica. A política de segurança descreve os cuidados de publicação e o procedimento para relatar problemas. [Consulte SECURITY.md](SECURITY.md).

## 📄 Licença

O projeto é disponibilizado sob a licença **GNU General Public License v3.0 or later** (`GPL-3.0-or-later`). [Consulte LICENSE](LICENSE).
