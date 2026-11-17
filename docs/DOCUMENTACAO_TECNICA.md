# Documentação técnica do AirPrice Scraper

Este documento descreve o comportamento implementado nos quatro módulos de `src/`. As instruções de instalação e execução estão no [README](../README.md). Os parâmetros locais seguem o modelo [config.example.ini](../config/config.example.ini).

## Arquitetura e módulos

O sistema consulta a página de ofertas da LATAM com Chrome e Selenium, organiza os resultados com `pandas` e exporta arquivos Excel com `openpyxl`. O envio ao Google Sheets usa `gspread` e `google-auth` quando há credenciais configuradas.

| Módulo | Responsabilidades |
| --- | --- |
| `scraper_passagens.py` | Interface Tkinter, rotas, configurações, processo do Chrome, conexão Selenium, repetições, agendador, Excel, Google Sheets e evidências. |
| `latam_scraper.py` | URL de busca, carregamento da página, diagnóstico de bloqueios e indisponibilidade, localização de ofertas, extração de preço e horários e registro operacional da tentativa. |
| `scrapers_utils.py` | Conversão de preços, escolha do menor preço, montagem de registros, IDs, classificação de horário, cancelamento, hash e remoção de HTML e CSV de tentativas. |
| `coleta_tcc.py` | Execução de uma rodada acadêmica com todas as rotas e as janelas de 7, 15, 30 e 90 dias. |

A GUI é formada por `ScraperApp`, `StartPage`, `ExperimentoPage`, `AgendadorPage`, `HistoricoPage` e `ConfigPage`. Coletas, agendamento e envio de arquivos são iniciados em threads daemon por `start_thread()`. Atualizações de log e caixas de diálogo da GUI são encaminhadas por `after()`.

`coleta_tcc.py` usa `ConsoleController` para registrar mensagens no terminal. Esse script importa o módulo central, portanto também depende dos pacotes da interface, inclusive `tkcalendar`, mesmo sem abrir a GUI.

## Rotas e parâmetros de busca

As rotas disponíveis são definidas em `ROTAS_DISPONIVEIS`.

| Origem | Destino |
| --- | --- |
| BSB | GIG |
| PVH | BSB |
| BSB | FLN |
| JPA | BSB |

`URL_TEMPLATE`, em `latam_scraper.py`, usa o portal brasileiro da LATAM, uma passagem de ida, um adulto, zero crianças, zero bebês, cabine econômica e pagamento em dinheiro. A ordenação solicitada à página é `RECOMMENDED`.

```text
https://www.latamairlines.com/br/pt/oferta-voos?origin={origem}&outbound={data}T00%3A00%3A00.000Z&destination={destino}&adt=1&chd=0&inf=0&trip=OW&cabin=Y&redemption=false&sort=RECOMMENDED
```

Cada chamada bem-sucedida de `scrape_latam()` devolve uma lista com a oferta de menor preço válido entre os elementos encontrados naquela busca. O conjunto de resultados depende dos cards carregados e dos seletores de extração. O código implementa a plataforma `LATAM`.

## Fluxo de uma coleta

1. A coleta manual, o agendador ou o script acadêmico monta as configurações de rota, data, privacidade, user-agent, repetições, janela do Chrome e experimento.
2. `executar_rotas_sequencial()` percorre as rotas na ordem recebida. A coleta manual faz seu próprio laço de rotas e chama diretamente `executar_plataformas_sequencial()`.
3. `executar_plataformas_sequencial()` acrescenta o nome da plataforma ao identificador e chama `executar_experimento_thread()`.
4. Para cada repetição, a orquestração permite até duas tentativas com um novo processo do Chrome. Um resultado encerra as tentativas daquela repetição.
5. `scrape_latam()` carrega a página, procura ofertas e retorna o registro de menor preço encontrado. Ao final da chamada, tenta acrescentar uma linha ao CSV operacional.
6. A orquestração captura evidências habilitadas e encerra o navegador no bloco `finally`.
7. Os resultados das repetições são reunidos por experimento. Quando todos estiverem vazios e `registrar_sem_dados=True`, a orquestração acrescenta uma linha sem preço.
8. `salvar_e_enviar()` normaliza as colunas, grava o Excel e tenta enviar os dados ao Google Sheets quando há configuração. A limpeza de HTML ocorre ao final desse fluxo.

Entre repetições há uma espera aleatória de 8 a 20 segundos quando `humanize=True`. Entre rotas há uma espera de 10 a 20 segundos, inclusive com `humanize=False`.

O identificador de uma coleta acadêmica segue, por exemplo, `EXTRACAO_7D_ROTA1_LATAM_R1_T1`. Os sufixos `_R1_T1` identificam repetição e tentativa do navegador no registro de preço e no CSV. Uma linha sem dados recebe o identificador do experimento de rota/plataforma, sem esses sufixos.

### Linhas sem preço

A coleta manual usa o padrão `registrar_sem_dados=False`. O agendador e `coleta_tcc.py` habilitam essa opção para preservar combinações de rota e janela sem resultado.

A linha é criada após o esgotamento das repetições sem resultado, inclusive quando as tentativas envolveram bloqueio, timeout ou erro. Ela contém `preco_brl=None`, horários `N/D` e `captcha_detectado=False`. Esse campo da linha vazia tem valor fixo. O CSV operacional permite consultar o desfecho das chamadas ao scraper.

Ao interpretar a quantidade de registros do script acadêmico ou do agendador, conte também essas linhas sem preço. Um registro salvo pode representar uma combinação sem oferta extraída.

## Inicialização e ciclo de vida do Chrome

`_localizar_chrome()` procura o executável primeiro em `[CHROME] path`. No Windows, consulta os diretórios de instalação em `Program Files`, `Program Files (x86)`, `LocalAppData` e o registro do sistema. Nos demais sistemas, procura executáveis pelo `PATH` e o caminho padrão do aplicativo no macOS.

`criar_driver()` inicia o Chrome por `subprocess.Popen()`, com uma porta de depuração local e janela de `1366x900`. Aguarda até 30 segundos pela porta e conecta `ChromeAnexado` por `debugger_address`. A criação do WebDriver usa a resolução de driver do Selenium. Os timeouts configurados após a conexão são 60 segundos para carregamento de página e 15 segundos para scripts.

`ChromeAnexado.abrir_url_desanexado()` encerra a sessão Selenium, abre uma aba pela interface de depuração local, aguarda o carregamento por até 45 segundos e reconecta a sessão para ler os resultados. Durante essa espera, usa o título e a URL da aba como sinais de progresso. Após reconectar, fecha abas anteriores.

| Modo de privacidade | Perfil e comportamento |
| --- | --- |
| `Sem cookies (perfil limpo)` | Cria um perfil temporário no diretório temporário do sistema. A segunda tentativa da repetição usa `Incognito` como fallback. |
| `Normal (com cookies)` | Usa `LOGS/chrome_profiles/normal`, preservando cookies e histórico entre execuções. |
| `Incognito` | Cria um perfil temporário e acrescenta `--incognito` ao Chrome. |

O scraper prefere rejeitar cookies nos modos com perfil temporário e aceitar no modo normal. Os fallbacks de fechamento do banner incluem botões de aceitação, portanto o resultado depende dos controles disponíveis na página.

O parâmetro interno `headless=True` solicita uma janela normal posicionada em `-32000,-32000` e iniciada minimizada. A opção de ocultar a janela mantém essa implementação e depende do ambiente gráfico do sistema operacional.

Os rótulos `Chrome Desktop (padrao)` e `Chrome Normal`, em `USER_AGENTS`, têm valor `None` e usam o user-agent nativo do Chrome instalado. Um valor textual associado a um rótulo no código é passado como `--user-agent`.

Quando proxy está habilitado e a lista contém entradas, a orquestração escolhe uma delas aleatoriamente por experimento e a passa ao Chrome com `--proxy-server`. As repetições e as duas tentativas de navegador desse experimento reutilizam essa escolha.

`ChromeAnexado.quit()` encerra a sessão Selenium e o processo do Chrome. Perfis temporários têm remoção tentada após o encerramento, com novas tentativas para arquivos ainda bloqueados. O perfil do modo normal permanece em `LOGS/chrome_profiles/normal`.

## Extração e diagnóstico da LATAM

O carregamento permite até quatro tentativas internas por chamada ao scraper. A partir da segunda, acrescenta um parâmetro `_retry` à URL e, quando `humanize=True`, aguarda de `18 + 7 × tentativa` a cinco segundos além desse valor antes de recarregar. Há também esperas de 5 a 9 segundos após a navegação quando essa opção está habilitada.

A espera por cards é de até 55 segundos. A espera pelo desaparecimento de indicadores de carregamento é de até 20 segundos. O scraper procura cards por `data-testid`, por classes alternativas e por elementos de preço e seus ancestrais. Uma tela de lentidão pode acionar o botão de refazer busca e uma nova espera.

A extração tenta, em sequência, seletores de preço principais, seletores alternativos e texto do card. Os utilitários rejeitam preços inferiores a R$ 10. O fallback textual usa um limite de R$ 80. Horários são obtidos por seletores e expressões regulares, com `N/D` quando indisponíveis.

O diagnóstico considera título, URL, tamanho do HTML, mensagens de acesso negado, indisponibilidade e lentidão, além de elementos de CAPTCHA ou desafio. O status HTTP da busca de ofertas é obtido dos logs de desempenho do Chrome e, quando possível, de Resource Timing da página.

O endpoint observado é `/bff/air-offers/v2/offers/search`. Respostas HTTP 401, 403 e 429 são tratadas como bloqueio. Essa observação usa o tráfego gerado pela página. O scraper interrompe suas tentativas internas ao identificar esse bloqueio. A orquestração ainda pode iniciar a segunda tentativa com um novo navegador quando o retorno estiver vazio.

Se a sessão Selenium for perdida, o scraper sinaliza `_sessao_invalida_tcc` e retorna sem resultado. A orquestração encerra o processo e pode repetir a coleta com outro navegador, respeitando o limite de duas tentativas por repetição.

## Dataset de preços

`COLUNAS_DATASET` define 17 colunas, nessa ordem. `normalizar_colunas_dataset()` cria as colunas ausentes com valores vazios e seleciona as colunas abaixo, descartando outras colunas do arquivo de entrada.

| Coluna | Conteúdo |
| --- | --- |
| `id_coleta` | Dez caracteres hexadecimais maiúsculos, derivados de MD5 de plataforma, rota, data de voo e instante UTC da geração. |
| `data_hora_local` | Data e hora da máquina no formato `YYYY-MM-DD HH:MM:SS`. |
| `janela_horario` | `madrugada (00-06h)`, `manha (06-12h)`, `tarde (12-18h)` ou `noite (18-24h)`. |
| `plataforma` | `LATAM`. |
| `url_consultada` | URL base da busca, sem o parâmetro de recarregamento `_retry`. |
| `origem` | Código IATA em maiúsculas. |
| `destino` | Código IATA em maiúsculas. |
| `data_voo` | Data consultada no formato `YYYY-MM-DD`. |
| `dias_antecedencia` | Parte inteira em dias da diferença entre a meia-noite de `data_voo` e o instante da coleta. |
| `preco_brl` | Menor preço válido extraído em reais ou valor vazio em uma linha sem dados. |
| `hora_saida` | Horário extraído do card selecionado ou `N/D`. |
| `hora_chegada` | Horário extraído do card selecionado ou `N/D`. |
| `modo_privacidade` | Modo efetivamente usado na tentativa bem-sucedida ou modo solicitado no caso de linha sem dados. |
| `user_agent_label` | Rótulo de `USER_AGENTS` selecionado para o experimento. |
| `captcha_detectado` | `False` nas ofertas retornadas e nas linhas sem dados criadas pela orquestração. |
| `dia_semana_busca` | Resultado de `strftime("%A")`, dependente da configuração regional do processo. |
| `experimento_id` | Identificador montado pela entrada de execução e pelas camadas de orquestração. |

As datas e horas locais seguem o relógio da máquina, sem fuso gravado no registro. Para uma janela nominal de 7 dias, o cálculo atual de `dias_antecedencia` normalmente produz 6, pois a data de voo é convertida em meia-noite e a coleta acontece após a meia-noite do dia atual. As janelas nominais permanecem identificáveis no `experimento_id` acadêmico e do agendador.

## CSV operacional de tentativas

`registrar_tentativa()` acrescenta uma linha em `LOGS/tentativas_coleta.csv` ao final de cada chamada de `scrape_latam()` que alcança esse ponto. O arquivo usa UTF-8 com BOM e o delimitador `;`. O cabeçalho é escrito quando o arquivo ainda não existe.

```text
data_hora_local, plataforma, origem, destino, data_voo,
dias_antecedencia, modo_privacidade, experimento_id,
status, http_api_ofertas, tentativas_carregamento,
cards_encontrados, precos_validos, preco_brl, duracao_ms, detalhe
```

| Status | Situação registrada |
| --- | --- |
| `sucesso` | A chamada retornou uma oferta com preço. |
| `bloqueado` | Foram identificados sinais de bloqueio ou os status HTTP tratados como bloqueio. |
| `sem_oferta` | O diagnóstico encontrou mensagem de indisponibilidade de voos. |
| `sem_preco` | Foram encontrados cards, mas a extração de preço ficou vazia. |
| `timeout` | A busca terminou sem cards e sem os diagnósticos anteriores. |
| `erro` | Ocorreu falha de sessão, WebDriver ou outra exceção tratada. |
| `cancelado` | O scraper capturou `ColetaCancelada`. |

`tentativas_carregamento` corresponde ao contador interno de carregamento daquela chamada, podendo ser zero em um bloqueio inicial. A chamada adicional com outro navegador gera outra linha no CSV. `precos_validos` corresponde à quantidade informada pela estratégia de extração usada e pode contar correspondências textuais no fallback.

Falhas anteriores à chamada do scraper, como Chrome ausente ou erro de importação do módulo, aparecem no log da execução e podem ficar sem linha nesse CSV. A função de gravação suprime exceções de escrita para permitir a continuação da coleta. A taxa calculada a partir do arquivo cobre as tentativas efetivamente registradas.

## Excel, Google Sheets e histórico

`salvar_excel()` grava um arquivo por experimento de rota/plataforma, reunindo suas repetições. O nome segue `DADOS_BRUTOS/passagens_{experimento_id}_{YYYYMMDD_HHMMSS}.xlsx`. Esses diretórios são criados ao importar o módulo central.

`salvar_e_enviar()` grava o Excel antes de consultar a configuração do Google Sheets. Se `arquivo_credenciais` e `id_planilha` estiverem preenchidos, `enviar_google_sheets()` autentica a Service Account com escopos de Sheets e Drive, abre a planilha pelo ID e localiza a aba configurada. Uma aba ausente é criada com 5.000 linhas e 17 colunas.

Em uma aba vazia, escreve cabeçalho e dados a partir de `A1`. Em uma aba com conteúdo, consulta as linhas existentes e escreve os novos valores após a última linha. Valores ausentes são enviados como strings vazias. O fluxo usa a ordem das colunas do dataset, sem conferir ou reorganizar o cabeçalho já existente na planilha.

Uma falha no envio é registrada no log após a gravação local. A tela de histórico lista os arquivos `.xlsx` em `DADOS_BRUTOS/`, em ordem de modificação mais recente, e permite selecionar um arquivo para novo envio. Esse envio lê o Excel, normaliza as colunas e acrescenta suas linhas à aba.

O envio atual acrescenta todas as linhas recebidas. Reenviar um arquivo já exportado pode duplicar dados. O cálculo da próxima linha também depende do conteúdo lido naquele instante, sem coordenação entre processos que enviem à mesma aba.

## Configuração local

O arquivo local é `config/config.ini`. A leitura usa `configparser.ConfigParser`. O caminho desse INI, os diretórios de saída e o CSV operacional são definidos a partir dos arquivos de código. Caminhos relativos informados em `arquivo_credenciais` e `[CHROME] path` são resolvidos a partir do diretório de trabalho do processo.

| Seção e chave | Padrão do modelo | Uso |
| --- | --- | --- |
| `GSHEETS.arquivo_credenciais` | Vazio | Arquivo JSON de Service Account. |
| `GSHEETS.id_planilha` | Vazio | ID da planilha de destino. |
| `GSHEETS.nome_aba` | `Dados` | Nome da aba para envio. |
| `ANTIDETECT.humanize` | `True` | Habilita as esperas opcionais de carregamento e entre repetições. |
| `ANTIDETECT.save_html` | `False` | Salva HTML compactado durante a tentativa. |
| `ANTIDETECT.screenshot` | `False` | Salva captura quando a tentativa retorna vazia ou ocorre erro com sessão válida. |
| `PROXY.enabled` | `False` | Habilita escolha de proxy por experimento. |
| `PROXY.list` | Vazio | Lista com uma entrada por linha. Linhas vazias e iniciadas por `#` são ignoradas. |
| `CHROME.path` | Vazio | Caminho explícito para o executável do Chrome. |

A GUI permite salvar as três primeiras seções. O caminho do Chrome é informado diretamente no INI, e a GUI preserva essa seção ao salvar as demais configurações.

`humanize=False` desativa somente as chamadas que recebem essa flag. Esperas de cards, spinners, conexão do Chrome, interação com botões e pausas entre rotas continuam fazendo parte do fluxo.

## Evidências e limpeza

Com `screenshot=True`, uma tentativa sem resultado pode gerar `sem_dados_LATAM_{experimento_id}_{timestamp}.png`. Exceções na orquestração podem gerar capturas com prefixo `erro`. A captura depende de uma sessão de navegador válida.

Com `save_html=True`, a página é gravada em `LOGS/html_LATAM_{experimento_id}_{timestamp}.html.gz`, e o hash MD5 do arquivo compactado é calculado. O caminho e o hash ficam em memória durante o experimento.

Os HTMLs são transitórios. A rotina tenta removê-los ao final de `salvar_e_enviar()`, inclusive com envio desabilitado ou malsucedido, e também ao final de um experimento sem resultados. Quando o hash esperado está disponível, um valor divergente preserva o arquivo. Se o cálculo inicial do hash retorna vazio, `destruir_html_log()` executa a remoção sem comparar hashes. A comparação usa o MD5 do próprio arquivo `.gz`.

Quando o envio ao Google Sheets tem sucesso, há uma limpeza dentro desse bloco e outra na saída geral da função. O log pode registrar a remoção na primeira passagem e informar que o arquivo já estava removido na segunda. Se ocorrer uma exceção antes de alcançar a limpeza, os arquivos podem permanecer em disco.

Screenshots, o CSV operacional e o perfil persistente do modo normal permanecem em seus diretórios. Os arquivos de configuração local, credenciais, dados gerados e evidências são tratados na [política de segurança](../SECURITY.md).

## Agendador, execução acadêmica e parada

O agendador percorre continuamente as janelas e rotas selecionadas. Em cada janela calcula a data por `datetime.now() + timedelta(days=janela)`, executa uma repetição por rota/plataforma e habilita o registro sem dados.

| Preset da GUI | Intervalo após o ciclo | Jitter |
| --- | --- | --- |
| `baixa (3h)` | 180 minutos | ±15 minutos |
| `media (1h)` | 60 minutos | ±8 minutos |
| `alta (30min)` | 30 minutos | ±5 minutos |
| `maxima (15min)` | 15 minutos | ±2 minutos |

O intervalo começa após terminar o ciclo de todas as janelas e rotas. A espera é `max(60, intervalo_min × 60 + random.uniform(-jitter_min × 60, jitter_min × 60))`, em segundos. Assim, a duração do próprio ciclo também compõe o tempo entre seus inícios. A execução depende da GUI e do processo Python permanecerem ativos. Os parâmetros selecionados na tela do agendador ficam em memória.

`coleta_tcc.py` executa uma rodada com quatro rotas e quatro janelas, totalizando 16 combinações nominais. Usa `Chrome Desktop (padrao)`, `Sem cookies (perfil limpo)`, uma repetição, `headless=True`, `show_dialogs=False` e `registrar_sem_dados=True`. Cada combinação pode gerar duas chamadas ao scraper, com até quatro tentativas internas de carregamento por chamada. O envio ao Google Sheets e as opções de evidência seguem o mesmo INI da GUI.

O controle global de parada é `stop_event`. Os botões de parada sinalizam o evento e iniciam o encerramento do navegador ativo. `sleep_interrompivel()` verifica cancelamento em intervalos de até 0,3 segundo nas esperas que controla. Laços de cards e spinners também verificam esse sinal. Chamadas bloqueantes de navegador e rede podem levar mais tempo para terminar. Cancelar pode deixar resultados parciais e, com `registrar_sem_dados=True`, uma linha sem preço ao final do experimento em andamento.

## Limitações para interpretação e manutenção

- O conteúdo da LATAM, seus seletores e as respostas de rede são dependências externas. Os diagnósticos de bloqueio, indisponibilidade e timeout usam os sinais que o código conseguiu observar.
- A oferta representa o menor preço extraído dos elementos encontrados na busca. Paginação, completude dos resultados, bagagem e condições tarifárias adicionais ficam fora dos campos coletados.
- As janelas nominais e `dias_antecedencia` seguem os cálculos descritos neste documento. Os registros usam horário local sem identificação de fuso.
- O CSV operacional depende de a chamada alcançar sua rotina de gravação e de a escrita funcionar. O dataset pode conter linhas sem preço com `captcha_detectado=False` após tentativas bloqueadas.
- O compartilhamento de `stop_event`, navegador ativo e perfil normal favorece a execução sequencial. O código atual mantém esses recursos globais e grava arquivos e planilhas sem coordenação entre processos simultâneos.
- As dependências Python têm versões fixadas em `config/requirements.txt`. Chrome, driver e serviço externo também participam da execução, e a validação de uma coleta real depende desse ambiente.

Alterações de rotas, parâmetros de busca, colunas, nomes de arquivos ou política de tentativas devem ser refletidas neste documento e no README. A planilha e as capturas de `data/` são materiais complementares preservados no repositório, com dados e aparência da execução em que foram produzidos.
