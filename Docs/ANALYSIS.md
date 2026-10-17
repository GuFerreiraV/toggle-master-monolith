# 1. Cultura DevOps e Arquitetura de Aplicações
- O projeto toggle-master-monolith trata-se de uma API REST feita em python utilizando flask, gunicorn e postgreSQL. Python e suas respectivas dependências são instaladas e contidas no container, o mesmo ocorre para o banco de dados postgreSQL.
- O projeto é considerado uma aplicação monolítica porque toda a sua funcionalidade está em uma única base de código (app.py), executada em um único processo e distribuída como uma única unidade de deploy (imagem docker). Não há separação de serviços independentes, então qualquer alteração exige um redeploy da aplicação inteira. A característica monolítica é reforçada ainda mais quando apenas um único banco postgreSQL é usado, acessado diretamente pelas rotas.
    - Vantagens: Deploy simples e fácil de testar localmente.
    - Desvantagens: Qualquer mudança exige redeploy e uma falha pode derrubar toda a aplicação.
- 12-Factor App
    - Identificados:
        -   I. Base de código: repositório git.
        - II. Dependências: Lista de dependências com requirements.txt e isolamento por container.
        - III. Configurações:  os.getenv no app, valores vindos do docker-compose.yaml
        - IV. Serviços de apoio: O Postgres é um recurso anexado via variáveis de ambiente.
        - VI. Processos: Cada requisição abre a própria conexão e nada fica na memória.
        - VII. Vínculo de porta: aplicação se expõe sozinha via http, o gunicorn faz 0.0.0.0:5000 sem depender de servidor externo.


    - Precisa de ajuste:
        - V. Não há versionamento de releases.
        - VIII. Concorrência: Gunicorn roda com um worker e não há separação por tipo de processo.
        - IX. Descartabilidade: Inicia rápido, porém o loop de espera para conectar-se ao banco não tem timeout.
        - X. Paridade dev/prod: Mesmo banco, dev e prod.
        - XI. Logs: Há um log básico configurado (print+logs do gunicorn), mas os erros de rotas vão para a respota http, não para o log.
        - XII. Processo de admin: Existe o comando init-db, mas ele roda automáticamente a cada start, em vez de ser um processo pontual.