# Hospedando-com-PHP-FPM-NGINX

Hospedando com PHP-FPM + NGINX


Neste tutorial trago minha experiência com hospedagem de um projeto PHP com Nginx usando o PHP-FPM 
para interpretar o PHP.

Este senário foi implementado no contexto de um homelab no qual o servidor é muito fraco  - notebook antigo -,
o ideal seria criar um container docker para minhas aplicações web, mas na tentativa de resguardar RAM decidi por
organizar bem meus projetos através do nginx e seus ambientes virtuais separados.


Meu homelab é um debian, atualmente na versão 13. Meu projeto PHP está no github, é um projeto bastante simples
onde não se necessita nem mesmo banco de dados.

O PHP-fpm é um serviço que interpreta o código php, ele é mais otimizado e rápido do que módulos de servidores 
pois sua única tarefa é interpretar o código e entregar ao servidor web, reduzindo a carga de processamento no servidor
fazendo com que consiga atender a mais clientes, o php-fpm também é otimizado com um cache, onde deixa em memória algumas
execuções muito requisitadas de código php, é a forma mais otimizada atualmente em termos de velocidade e capacidade de conexões.


Primeiramente vamos instalar o php-fmp e o php, também o composer, gerenciador de dependencias do php o qual será necessário 
para a hospedagem do projeto.

<code>
apt install php8.4 php-curl php8.4-fpm composer
</code>

com php-fpm instalado você pode verificar se está rodando:

<code>  
systemctl status php8.4-fpm
</code>

o fpm possui pools, que são como processos diferentes que gerenciam cada projeto, existe um pool default para todos, mas
é uma boa prática criar o seu próprio e para cada projeto.

O diretório de configuração é:

<code>
/etc/php/8.4/fpm/pool.d
</code>

você pode dar uma olhada no www.conf, pool padrão do fpm para ter uma ideia, aqui está a minha configuração:

para criar uma: 

<code>
nano projeto_pool.conf


[ projeto_site ]
user = usuario
group = usuario
listen = /var/run/fpm-project-site.sock
listen.owner = www-data
listen.group = www-data
php_admin_flag[allow_url_fopen] = off
; Choose how the process manager will control the number of child processes.
pm = dynamic
pm.max_children = 75
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 20
pm.process_idle_timeout = 10s
</code>




[ projeto_site ] é o nome do pool <br />
user e group é o usuário responsável pelo processo/pool <br />
listen é a conexão da máquina para o processo, pode ser conexão unix com sock, ou com portas locais como 127.0.0.1:9000 <br />
listen.owner e group é o user responsável por escutar e executar os scripts, por padrão em servidores web linux, o usuario <br />
www-data 
</code>



reinicie o serviço fpm:
<code>
systemctl restart php8.4-fpm
</code>

verifique se o pool com o nome do seu site/projeto está rodando com

<code>
systemctl status php8.4-fpm
</code>

o nome do seu pool deve aparecer nos processos do status de serviço


agora vamos trazer o projeto para a máquina, vá para

<code>
cd /var/www
</code>
dê um git para trazer o projeto para o server

git clone https://github.com/eu/meu-projeto.git/

dê as permissões

<code>
chown -R www:data:www-data meu-projeto/
chmod 776 meu-projeto/
</code>

no meu caso de ser um projeto feito em php que precisa do composer para seu autoload:

<code>
composer install
composer dump-autoload -o
</code>



php-fpm rodando, projeto no server, hora de instalar o nginx

<code>
apt install nginx
</code>


a escolha do ngnix foi por dois motivos, ele é leve e rápido, facilmente você consegue gerenciar para que ele redirecione 
a requisição web para outros serviços como php-fpm e ou guinicorn para django de acordo com o domínio requisitado.


o diretório de configuração do nginx é:

<code>
cd /etc/nginx
</code>

aqui vamos nos atentar em dois diretórios:
sites-enabled
sites-available



sites-available contém os arquivos de configuração para cada projeto/site, sites-enabled, contem um link simbolico para 
os arquivos de conf, os links simbolicos de sites-enabled são o on/off do servidor, o nginx só vai atender as requisições 
cujos domínios estem com o link aqui.



vamos criar o sites-available/meu-projeto.conf

<code>
server {
         listen       80;   // porta a escutar
         server_name  meu-projeto.com; // nome do domínio, url http://meu-projeto.com
         root         /var/www/meu-projeto; //path do projeto

         access_log /var/log/nginx/meu-projeto-access.log;  //arquivos de log
         error_log  /var/log/nginx/meu-projeto.log error;	// arquivos de log
         index index.html index.htm index.php;			// arquivos ao qual o server vai buscar na pasta root

         location / {
                      try_files $uri $uri/ /index.php?url=$request_uri;  //localização root da requisição,  o ultimo campo garante URL amigáveis

         }

         location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass unix:/var/run/fpm-meu-projeto-site.sock; // nome da conexão unix, a mesma do arquivo projeto_pool.conf
            fastcgi_index index.php;
            include fastcgi.conf;
    }
}
</code>


reinicie seu nginx

<code>
systemctl restart nginx
</code>

a partir de agora você poderá acessar seu site pelo navegador, mas somente usando o nome do domínio configurado
teste com curl

<code>
curl -H "Host: meu-projeto.com" http://127.0.0.1/
</code>


ou modifique o arquivo hosts para associar o dominio ao ip.






