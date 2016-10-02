#### Fiap - Soluções em Redes para ambientes Linux
profhelder.pereira@fiap.com.br

---
## Deploy de configuranção do bind como SOA de uma zona:

O exemplo anterior serviu para esquentar um pouco e para entendermos a estrutura basica do bind, para este laboratório executaremos o processo de configuração do bind como SOA "Start of Authority" do domínio fictício fiap.edu.br.

1 - Verifique se o pacote bind está instalado no ambiente ubuntu, caso não esteja execute a instalação e em seguida abra o arquivo de configuração de zonas ***named.conf.local***:

```sh
apt-get install bind9
vim /etc/bind/named.conf.local
```

> Lembre-se para algumas distribuições o arquivo original é o arquivo named.conf, no caso da familia Debian optou-se por utilizar uma hierarquia organizacional onde recomenda-se que as zonas sejam alocadas no arquivo named.conf.local e as opções de configuração e segurança no arquivo named.conf.options.

Adicione o trecho abaixo no arquivo de configuracao de zonas

```sh
zone "fiap.edu.br" {
        type master;
        file "/var/cache/bind/db.fiap.edu.br";
};

zone "1.168.192.in-addr.arpa" {
        type master;
        file "/var/cache/bind/rev.fiap.edu.br";
};
```

As linhas acima descrevem o seguinte:

1. A Zona a ser configurada é a zona ***fiap.edu.br***, a string ***zone*** declara que inicamos a configuração de uma nova zona;
2. O tipo de zona escolhido foi ***master*** ou seja, esse será o DNS principla resposnável pela zona
3. Outros DNS tambem poderão responder por essa zona porem como tipo ***slave***, faremos isso em breve
4. O campo file determina one está o arquivo de zona, usamos a PATH completa apenas para fins didaticos pois, o diretório "/var/cache/bbind/" é a pasta default para armazenar arquivo de zona cofigurada automaticamente na instalação do bind9.
5. A segunda zona configurada trata-se de uma zona reversa para resolução de nomes na rede local.

Configure a zona fiap.edu.br conforme o padrão salvo no repositorio git da aula (  Arquivo db.fiap.edu.br), 
Detalhes sobre cada tipo de ponteiro e sua funcao estao em comentarios no final do proprio arquivo de zona.

```sh
vim /var/cache/bind/db.fiap.edu.br
```

Verifique a zona criada utilizando o comando de checagem named-checkzone:

```sh
named-checkzone fiap.com.br /var/cache/bind/db.fiap.edu.br
```

Apos finalizar configurações e testes execute a reinicialização do serviço:

```sh
service bind9 restart
```

Verifique se o resolve.conf aponta para seu proprio servidor e inicie os testes de sua zona usando o comando host:

```sh
host gateway.fiap.edu.br
dig proxy.fiap.edu.br +short
```

Faça novos testes com o comando dig verificando respectviamente:

- Os ponteiros SOA;
- Os ponteiros NS;
- Ponteiros reversos;
- Acesso a partir do endereço de rede ao prcoesso de resolução de nomes.

```sh
dit -t SOA fiap.edu.br
dit -t NS fiap.edu.br
dig -t x 192.168.1.3
dig @192.168.1.2 web.fiap.edu.br
```

### Limitando consultas e processos recursivos no bind:

Em casos onde um servidor DNS esteja operando fora da uma rede local, alguns recrusso de seugrança são importantes, um deles é desabilitar a opção de recursão, pois isto evitará abusos, como por exemplo, algum host utilizar o servidor para fazer consultas de DNSpara outros dominios;

Para executar essa configuração iremos simplesmente especificar o seguinte: 
- Quais dominios podem executar consultas ao nosso DNS  ***allow-query***
- Quais dominios podem executar consultas recursivas através da opção ***allow-recursive***

Configure o arquivo named.conf.options conforme o modelo armazenado no repositório git da aula ( Arquivo named.conf.options ).

```sh
vim /etc/bind/named.conf.options
service bind9 restart
```
