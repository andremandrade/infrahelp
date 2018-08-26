
# MIGRAÇÃO DO GITLAB - 8.x > 11.x
Esse procedimento foi testado migrando de um `Ubuntu 14.04 LTS` - `Gitlab 8.5.9` para um `Ubuntu 18.04 LTS` - `Gitlab 11.04`, no entanto, de  acordo com a [documentação do Gitlab](https://docs.gitlab.com/ee/policy/maintenance.html), o mesmo deve funcionar para qualquer versão `8.x`, e pela lógica, deve funcionar pelo menos para qualquer versão posterior à `8.5.9`  =)

## BACKUP E RESTORE 

O backup é necessário caso você deseje atualizar também a versão do Linux. Se preferir mantê-la, clone a sua VM e pule para o primeiro upgrade (8.5.9  > 8.17.8), pois o backup é gerado automaticamente a cada upgrade.

REFERENCIA: https://docs.gitlab.com/ee/raketasks/backup_restore.html

### Na VM origem:
	
Gerar backup na maquina de produção

    sudo gitlab-rake gitlab:backup:create

Fazer backup das configuracoes

    tar -vczf gitlab-conf.tar.zg /etc/gitlab

Tranferir os dois backups para a VM da nova instância

### Na VM destino (Ubuntu 16 ou 18):

[Baixar](https://packages.gitlab.com/gitlab/gitlab-ce) e instalar o gitlab de mesma versão da VM origem:

    sudo dpkg -i gitlab-ce_8.5.9-ce.0_amd64.deb

Extrair o backup da configuração feito etapa anterior para /etc/gitlab.

Ajustar o 'external_url' no gitlab.rb, se for o caso.

Reconfigurar o gitlab:

    sudo gitlab-ctl reconfigure

Executar o gitlab:

    sudo gitlab-ctl start

Copiar (ou mover) o backup do gitlab para a pasta de backups e dê permissão ao usuário `git`:

    sudo cp 1530973830_gitlab_backup.tar /var/opt/gitlab/backups/
    sudo chown git:git /var/opt/gitlab/backups/1530973830_gitlab_backup.tar

Parar os serviços que se conectam ao banco

    sudo gitlab-ctl stop unicorn
    sudo gitlab-ctl stop sidekiq
    # Verify
    sudo gitlab-ctl status

Restaurar o backup

    sudo gitlab-rake gitlab:backup:restore BACKUP=1530973830

Reiniciar o gitlab

    sudo gitlab-ctl restart	

Checar a instalação:

    sudo gitlab-rake gitlab:check SANITIZE=true

**Se necessário**, corrigir erros (os erro apresentados, no geral, já apresentam a solução). Solução comumente apresentada para essa etapa:

    sudo chmod -R ug+rwX,o-rwx /var/opt/gitlab/git-data/repositories
    sudo chmod -R ug-s /var/opt/gitlab/git-data/repositories
    sudo find /var/opt/gitlab/git-data/repositories -type d -print0 | sudo xargs -0 chmod g+s


## UPGRADE 8.5.9  > 8.17.8
Você pode, opcionalmente, usar o `apt-get`, no entanto, para `Ubuntu 18`, a versão mais antiga disponível é a `10.7`. Portanto, vamos na mão mesmo.

[Baixar](https://packages.gitlab.com/gitlab/gitlab-ce/packages/ubuntu/xenial/gitlab-ce_8.17.8-ce.0_amd64.deb) e instalar nova versão	

`sudo dpkg -i gitlab-ce_8.17.8-ce.0_amd64.deb`
	
**Se ocorrer** o erro do secret ([documentação do Gitlab](https://docs.gitlab.com/omnibus/update/gitlab_8_changes.html)):
* Colocar a chave ['gitlab_rails']['otp_key_base'] do /etc/gitlab/gitlab-secrets.json no /var/opt/gitlab/gitlab-rails/etc/secret
* Reconfigurar:

	`sudo gitlab-ctl reconfigure`

Reiniciar:
	`sudo gitlab-ctl restart`

Checar a instalação:

    sudo gitlab-rake gitlab:check SANITIZE=true

**Se necessário** e corrigir erros (os erro apresentados, no geral, já apresentam a solução). Solução de erro comumente apresentada para essa etapa:

    sudo chown -R git /var/opt/gitlab/gitlab-rails/uploads
    sudo find /var/opt/gitlab/gitlab-rails/uploads -type f -exec chmod 0644 {} \;
    sudo find /var/opt/gitlab/gitlab-rails/uploads -type d -not -path /var/opt/gitlab/gitlab-rails/uploads -exec chmod 0700 {} \;

## UPGRADE 8.17.8 > 9.5.9
	
[Baixar](https://packages.gitlab.com/gitlab/gitlab-ce/packages/ubuntu/xenial/gitlab-ce_9.5.9-ce.0_amd64.deb) e instalar nova versão

    sudo dpkg -i gitlab-ce_9.5.9-ce.0_amd64.deb

Aumentar memória do POSTGRESQL

    # /etc/gitlab/gitlab.rb
    postgresql['shared_buffers'] = "8192MB"

Reconfigurar

    sudo gitlab-ctl reconfigure

Checar instalação

    sudo gitlab-rake gitlab:check SANITIZE=true

## UPGRADE 9.5.9 > 10.0.0

[Baixar](https://packages.gitlab.com/gitlab/gitlab-ce/packages/ubuntu/xenial/gitlab-ce_10.0.0-ce.0_amd64.deb) e instalar nova versão

    sudo dpkg -i gitlab-ce_10.0.0-ce.0_amd64.deb

O passo anterior deverá mostrar um erro no final e o `reconfigure` deixará de funcionar, porém, a instalação foi feita. Para corrigir o erro, execute:
[Solução do forum do Gitlab](https://gitlab.com/gitlab-org/gitlab-ce/issues/38246#note_41171428)

    sudo gitlab-rake gitlab:db:mark_migration_complete[20170828170502]
    sudo gitlab-rake db:migrate
    sudo gitlab-ctl reconfigure

Se checar instalação, você verá que existe um problema:

    sudo gitlab-rake gitlab:check SANITIZE=true
	    ...
	    Ruby version >= 2.3.3 ? ... Exception: undefined method 'run_command' for SystemCheck::App::RubyVersionCheck:Class
	    Did you mean?  run_commands
	    Git version >= 2.7.3 ? ... Exception: undefined method ' 'run_command' for SystemCheck::App::GitVersionCheck:Class
	    Did you mean?  run_commands

Trata-se de um bug que será corrigido na próxima atualização

## UPGRADE 10.0.0 > 11.0.4

Instalar versão 10.7.0. Agora, no `Ubuntu 18` pelo menos, já podemos usar o `apt-get`:

    sudo apt-get install gitlab-ce=10.7.0-ce.0

Checar a instalação (o bug da `10.0.0` já não deverá estar mais presente):

    sudo gitlab-rake gitlab:check SANITIZE=true

Instalar versão 10.8.6

    sudo apt-get install gitlab-ce=10.8.6-ce.0

Checar a instalação:

    sudo gitlab-rake gitlab:check SANITIZE=true

Instalar versão 11.0.0

    sudo apt-get install gitlab-ce=11.0.0-ce.0

Checar a instalação:

    sudo gitlab-rake gitlab:check SANITIZE=true

Instalar versão 11.0.4

    sudo apt-get install gitlab-ce

Checar a instalação:

    sudo gitlab-rake gitlab:check SANITIZE=true

## Verificar a instalação

Acesse http://endereco.do.gitlab/help e verifique a versão em execução.
