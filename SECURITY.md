# Segurança e publicação

O repositório distribui código-fonte, documentação, licença, configuração de exemplo e materiais complementares públicos de `data/`. As planilhas e imagens desse diretório também entram no pacote gerado por `scripts/build_submission_zip.ps1`.

## Arquivos locais

Os arquivos abaixo ficam no ambiente de execução e são excluídos do versionamento pelas regras de `.gitignore`.

- `config/config.ini`, com configurações locais e caminhos de credenciais.
- `credentials/` e arquivos JSON de Service Account.
- `DADOS_BRUTOS/`, com planilhas das coletas.
- `LOGS/`, incluindo `tentativas_coleta.csv`, screenshots, HTML compactado e o perfil persistente em `chrome_profiles/normal/`.
- Arquivos `.env`, `.env.*`, `.pem`, `.key` e `.p12`.

Use `config/config.example.ini` como modelo público. Mantenha IDs, links de planilhas privadas e listas de proxies na configuração local. Guarde senhas e tokens em arquivos privados com exclusão no `.gitignore`. Os perfis temporários dos modos de perfil limpo e incógnito são criados no diretório temporário do sistema operacional e têm remoção tentada ao encerrar o navegador.

Antes de acrescentar uma planilha ou imagem a `data/`, revise o conteúdo que será publicado. O empacotador inclui arquivos `.jpeg`, `.jpg`, `.png` e `.xlsx` diretamente desse diretório. As regras de `.gitignore` se aplicam a arquivos ainda sem rastreamento pelo Git. Arquivos já rastreados exigem revisão própria.

## Exposição acidental

Se uma credencial, chave ou token for publicado, revogue ou substitua o segredo no provedor correspondente. Remova o material exposto do repositório e dos pacotes distribuídos e revise o acesso aos arquivos envolvidos antes de gerar outro pacote.

## Relato de problemas

Relate problemas de segurança pelo rastreador de issues do projeto. Descreva o comportamento e os passos para reprodução com dados fictícios. Preserve credenciais, URLs privadas e dados de coleta ao preparar o relato.
